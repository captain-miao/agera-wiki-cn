# Incrementally Agerifying legacy code

Agera引入的代码风格也许适合从零开始的新建app项目。这篇包括一些提示，来帮助想在遗留代码中使用Agera的开发者，如此往下做。

## Upgrading legacy observer pattern

### --升级原有观察者模式

观察者模式有很多种实现方式，但不是所有的都可以通过简单的放入迁入到Agera中。下面是一个例子：演示将一个监听(listenable)类添加到`Observable`接口的一种方法。

`MyListenable`类可以增加(`addListener`)和删除(`removeListener`)`Listener`，作为额外完整的演示，它继承了`SomeBaseClass`。

该实例使用`UpdateDispatcher`来解决Java的单继承约束，使用一个内部类(`Bridge`)来做桥接， 保持其完整的原始API，同时也使Agera可见。

```java
public final class MyListenable extends SomeBaseClass implements Observable {

  private final UpdateDispatcher updateDispatcher;

  public MyListenable() {
    // Original constructor code here...
    updateDispatcher = Observables.updateDispatcher(new Bridge());
  }

  // Original class body here... including:
  public void addListener(Listener listener) { … }
  public void removeListener(Listener listener) { … }

  @Override
  public void addUpdatable(Updatable updatable) {
    updateDispatcher.addUpdatable(updatable);
  }

  @Override
  public void removeUpdatable(Updatable updatable) {
    updateDispatcher.removeUpdatable(updatable);
  }

  private final class Bridge implements ActivationHandler, Listener {
    @Override
    public void observableActivated(UpdateDispatcher caller) {
      addListener(this);
    }

    @Override
    public void observableDeactivated(UpdateDispatcher caller) {
      removeListener(this);
    }

    @Override
    public void onEvent() { // Listener implementation
      updateDispatcher.update();
    }
  }
}
```

## Exposing synchronous operations as repositories  

### --揭秘repository的同步操作    

Java本质是一种同步语言，如：在Java中最低级别的操作都是同步方法。 当操作可能会花一些时间才能返回(耗时操作)，这种方法通常称为阻塞方法，而且禁止开发者在主线程(UI Thread)调用。

假设app的UI需要从阻塞的方法获得数据。Agera可以很容易的通过后台线程完成调用，然后UI可以在主线程中接收数据。首先，这个阻塞操作需要封装成一个Agera操作，像这样：

```java
public class NetworkCallingSupplier implements Supplier<Result<ResponseBlob>> {
  private final RequestBlob request = …;
    
  @Override
  public Result<ResponseBlob> get() {
    try {
       ResponseBlob blob = networkStack.execute(request); // blocking call
       return Result.success(blob);
    } catch (Throwable e) {
       return Result.failure(e);
    }
  }
}
    
Supplier<Result<ResponseBlob>> networkCall = new NetworkCallingSupplier();

Repository<Result<ResponseBlob>> responseRepository =
    Repositories.repositoryWithInitialValue(Result.<ResponseBlob>absent())
        .observe() // no event source; works on activation
        .onUpdatesPerLoop() // but this line is still needed to compile
        .goTo(networkingExecutor)
        .thenGetFrom(networkCall)
        .compile();
```

上面的代码段假定了，在Repository.compile()之前这个请求是已知且永远不变的。

这个很容易升级成为一个动态请求，甚至在repository同样的激活周期期间。

要可以修改请求，简单的方式是使用`MutableRepository`。
此外，为了在第一次请求为完成之前就可以提供数据，可以在`Result`中一个提供初始值：`absent()`。

MutableRepository这种用法类似于是一个可变的变量(可为null)，故命名为`requestVariable`。

```java
// MutableRepository<RequestBlob> requestVariable =
//     mutableRepository(firstRequest);
// OR:
MutableRepository<Result<RequestBlob>> requestVariable =
    mutableRepository(Result.<RequestBlob>absent());
```

然后, 不是在supplier中封装阻塞方法，使用function实现动态请求:

```java
public class NetworkCallingFunction
    implements Function<RequestBlob, Result<ResponseBlob>> {
  @Override
  public Result<ResponseBlob> apply(RequestBlob request) {
    try {
       ResponseBlob blob = networkStack.execute(request);
       return Result.success(blob);
    } catch (Throwable e) {
       return Result.failure(e);
    }
  }
}

Function<RequestBlob, Result<ResponseBlob>> networkCallingFunction =
    new NetworkCallingFunction();
```

升级后的repository可以像这样compiled：

```java
Result<ResponseBlob> noResponse = Result.absent();
Function<Throwable, Result<ResponseBlob>> withNoResponse =
    Functions.staticFunction(noResponse);
Repository<Result<ResponseBlob>> responseRepository =
    Repositories.repositoryWithInitialValue(noResponse)
        .observe(requestVariable)
        .onUpdatesPerLoop()
        // .getFrom(requestVariable) if it does not supply Result, OR:
        .attemptGetFrom(requestVariable).orEnd(withNoResponse)
        .goTo(networkingExecutor)
        .thenTransform(networkCallingFunction)
        .compile();
```

这部分代码段还演示了一点：通过给操作一个特殊的名字，让repository的编译表达式更易读。  


## Wrapping asynchronous calls in repositories  

### --repository的异步调用封装 
 
现在的很多Library都有异步API和内置的线程，但是客户端不能控制或禁用。

app中有这样的Library的话，引入Agera将是一个具有挑战性的工作。
一个直接的办法就是找到Library中同步选择的点，采用[[如上段所述|Incrementally-Agerifying-legacy-code#exposing-synchronous-operations-as-repositories]]方法。

另一个方式(反模式)：切换后台线程执行异步调用并等待结果，然后同步拿结果。上面方式不可行时，这一节讨论一个合适的解决方法。

异步调用的一个循环模式是请求-响应 结构。下面的示例假定这样结构：未完成的工作可以被取消，但是不指定回调的线程。

```java
interface AsyncOperator<P, R> {
  Cancellable request(P param, Callback<R> callback);
}
    
interface Callback<R> {
  void onResponse(R response); // Can be called from any thread
}

interface Cancellable {
  void cancel();
}
```

下面repository例子，使用`AsyncOperator`提供数据, 完成响应式请求(一个抽象的supplier类)。

这段代码假定`AsyncOperator`已经有足够的缓存，因此重复的请求不会影响性能。 

```java
public class AsyncOperatorRepository<P, R> extends BaseObservable
    implements Repository<Result<R>>, Callback<R> {

  private final AsyncOperator<P, R> asyncOperator;
  private final Supplier<P> paramSupplier;

  private Result<R> result;
  private Cancellable cancellable;

  public AsyncOperatorRepository(AsyncOperator<P, R> asyncOperator,
      Supplier<P> paramSupplier) {
    this.asyncOperator = asyncOperator;
    this.paramSupplier = paramSupplier;
    this.result = Result.absent();
  }

  @Override
  protected synchronized void observableActivated() {
    cancellable = asyncOperator.request(paramSupplier.get(), this);
  }

  @Override
  protected synchronized void observableDeactivated() {
    if (cancellable != null) {
      cancellable.cancel();
      cancellable = null;
    }
  }

  @Override
  public synchronized void onResponse(R response) {
    cancellable = null;
    result = Result.absentIfNull(response);
    dispatchUpdate();
  }

  @Override
  public synchronized Result<R> get() {
    return result;
  }
}
```

这个类可以很容易地升级到可以修改请求参数，而这个过程就类似于前面的讨论：让repository提供请求参数，并让`AsyncOperatorRepository`观察请求参数变化。

在激活期间，观察请求参数的变化，取消任何正在进行的请求，并发出新的请求，如下所示：

```java
public class AsyncOperatorRepository<P, R> extends BaseObservable
    implements Repository<Result<R>>, Callback<R>, Updatable {

  private final AsyncOperator<P, R> asyncOperator;
  private final Repository<P> paramRepository;

  private Result<R> result;
  private Cancellable cancellable;

  public AsyncOperatorRepository(AsyncOperator<P, R> asyncOperator,
      Repository<P> paramRepository) {
    this.asyncOperator = asyncOperator;
    this.paramRepository = paramRepository;
    this.result = Result.absent();
  }

  @Override
  protected void observableActivated() {
    paramRepository.addUpdatable(this);
    update();
  }

  @Override
  protected synchronized void observableDeactivated() {
    paramRepository.removeUpdatable(this);
    cancelOngoingRequestLocked();
  }

  @Override
  public synchronized void update() {
    cancelOngoingRequestLocked();
    // Adapt accordingly if paramRepository supplies a Result.
    cancellable = asyncOperator.request(paramRepository.get(), this);
  }

  private void cancelOngoingRequestLocked() {
    if (cancellable != null) {
      cancellable.cancel();
      cancellable = null;
    }
  }

  @Override
  public synchronized void onResponse(R response) {
    cancellable = null;
    result = Result.absentIfNull(response);
    dispatchUpdate();
  }

  // Similar process for fallible requests (typically with an
  // onError(Throwable) callback): wrap the failure in a Result and
  // dispatchUpdate().

  @Override
  public synchronized Result<R> get() {
    return result;
  }
}
```