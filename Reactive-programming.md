Agera 使用著名的 _观察者模式_ 作为响应式编程的驱动机制。
被观察者(observable)实现`Observable`接口， 并向所有注册的观察者们(observers)广播事件。
观察者(observer)实现`Updatable`接口, 可以注册和注销到`Observable`中，接受通知事件触发更新操作，故此命名为`Updatable`。

### `Observable`
```
public interface Observable {

  void addUpdatable(@NonNull Updatable updatable);

  void removeUpdatable(@NonNull Updatable updatable);
}
```
### `observer`  
```
public interface Updatable {

  void update();
}
```

接下来的文档中，将用_observable_和_updatable_来表示被观察者和观察者对象。

# Push event, pull data

Agera 使用 _push event, pull data_ 模型(推送事件,拉取数据)。
push event：被观察者只做事件通知，不携带任何数据;
pull data：观察者根据自己需要从数据仓库(Repository.get())拉取数据。

这种 _push event, pull data_ 模型, 观察者就不需要提供数据了(ps:通常意义上的观察者模式是支持携带数据和不携带数据的), 可以封装简单的事件，
比如 按钮点击事件，下拉刷新触发事件，一个同步信号(比如:谷歌推送(GCM)消息到app)等。  
```
    // push event
    mObservable = new OnClickObservable() {
        @Override
        public void onClick(View view) {
            dispatchUpdate();
        }
    };

    @Override
    public void update() {
    	// pull data
        String result = mRepository.get();
        mBinding.setImageUrl(result);
    }
```

然而, 被观察者一般也提供数据。一个可以提供数据，还可以定义在提供数据发生变化时的事件的被观察者，称为数据仓库(`Repository`)。
这并没有改变 _push event, pull data_ 模型: 当数据变化时，数据仓库(`Repository`)通知所有观察者更新；观察者各自从数据仓库(`Repository`)拉取数据响应事件。
这种模型的好处是：将数据和事件通知分离，数据仓库(`Repository`)可以实现懒加载。  

由于 _push event, pull data_ 模型和多线程情况下，观察者可能看不到数据全部的更新记录。
这是特意设计的: 因为大多数情况下(尤其更新app UI), 本来就只需要关心最新的数据。 

一个典型Agera风格的响应式Client由以下几部分组成：
* 向Observables注册一个Updatable，用于事件通知;
* 可直接调用update，来初始化或更改Client状态;
* 等待Observables的通知，拉取最新的数据来更新Client;
* 当不需要事件了，需要向Observables注销updatable，释放资源。

ps: 一个Repository的定义
```
    // 数据提供者 text color Supplier
    Supplier<Integer> supplier = new Supplier<Integer>() {
        @NonNull
        @Override
        public Integer get() {
            return MockRandomData.getRandomColor();
        }
    };
    
    mRepository = Repositories.repositoryWithInitialValue(0)
            .observe(mObservable)// 事件源
            .onUpdatesPerLoop()
            .thenGetFrom(supplier)// 数据源
            .compile();
```
