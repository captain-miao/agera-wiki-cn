如[[上一篇|Reactive-programming]]所讲, 
被观察者(Observable)作为事件源，观察者(updatable)监听(观察)事件。   
注册观察者(Updatable)方法：`Observable.addUpdatable(Updatable)`  
注销观察者(Updatable)方法：`Observable.removeUpdatable(Updatable)`  
事件分发给观察者(Updatable)方法：`Updatable.update()`  
在activity中的用法举例:

```java
public class MyUpdatableActivity extends Activity implements Updatable {
  private Observable observable;
  
  @Override
  protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    observable = new MyObservable();
  }
   
  @Override
  public void update() {
    // 事件通知 处理
  }
      
  @Override
  protected void onResume() {
    super.onResume();
    observable.addUpdatable(this);
    update();
  }
      
  @Override
  protected void onPause() {
    super.onPause();
    observable.removeUpdatable(this);
  }
}
```

Updatable的注册和注销必须配对使用。不能重复注册Updatable，不能注销没有注册过的Updatable，也不能重复注销Updatable，等其他非法操作。

# Activation lifecycle and event chain

被观察者(Observable)的状态：  
_active_状态(活动状态)：有观察者 (至少注册一个了Updatable)  
_inactive_状态(非活动状态)：没有观察者 (没有注册的Updatable)

也就是, 注册Updatable让Observable从非活动状态激活为活动状态，当Updatable全部注销了，Observable从活动状态变为非活动状态。  

![](https://github.com/google/agera/raw/master/doc/images/observablelifecycle.png)

被观察者(Observable)还可以观察其它“上游”(事件传播路径)的被观察者们(Observables)，把它们的事件转换成自己的事件。
一个常见的例子：一个数据仓库(Repository)，它的数据依赖另一个数据仓库(Repository)的数据。 
为了事件传播路径的正确性, 处于中间位置的被观察者(Observable)一般持有上游被观察者(Observable)的强引用, 通常当自己激活的时候向上游注册观察者(Updatable)，
当自己非活动的时候向上游注销观察者(Updatable)。
这就是说，在下游方向上的强引用只用于注册/注销观察者(Updatable)，这也意味着，在最下游的观察者(Updatable)(那些不被中间位置的被观察者管理的被观察者们)
最终控制所有观察者(Updatable)的活动和非活动状态。  

![](https://github.com/google/agera/raw/master/doc/images/downstream.png)

# UI lifecycle

事件链在构建有生命周期UI的响应式架构方面非常有用。UI元素比如Activity、Fragment、其中的View，
Android生命周期中定义的成对状态变化，比如：onStart 到 onStop，onResume 到 onPause，onAttachedToWindow 到 onDetachedFromWindow 等等。
让UI元素成为或者拥有一个观察者(Updatable)，通过从数据仓库(Repository)获取数据来更新UI。反过来数据仓库(Repository)可以使用其他的事件源和数据源(不一定是Repository)来计算自己的数据。

在UI元素生命周期的开始，注册观察者(Updatable)，然后数据仓库(Repository)处于活动状态。这将连接事件链，激活所有相关的数据处理流程，保持数据和UI是最新的。

在UI元素生命周期结束时，同一个数据仓库(Repository)注销观察者(Updatable)，假设所有观察者(Updatable)都被注销，那么事件链将级联断开。
如果UI元素是不会再激活(例如activity.onDestroyed())，因为当前系统不活动了没有下游的引用，UI元素可以被垃圾回收，因此很容易预防Activity泄漏。

![](https://github.com/google/agera/raw/master/doc/images/uilifecycle.png)

# Threading

Agera 推荐指明执行线程, 使用Loopers(Looper在Android中大量存在, 比如：APP Main Looper 和 IntentService中)来定义下面的线程约定。
为了处理内部状态生命周期，每一个观察者(Updatable)在生命周期内都关联一个有Looper的线程(worker Looper thread)，有Looper的线程就是实例化被观察者(Observable)的线程。
被观察者(Observable)的注册和注销事件分发都是通过Looper。如果被观察者观察了其它被观察者，它内部的被观察者(Observable)将会从当前线程注册到上游的观察者(Updatable)。

被观察者(Observable)必须在有Looper的线程上注册观察者(Updatable)，观察者(Updatable)和被观察者(Observable)可以运行在不同的线程。
被观察者(Observable)会使用相同Looper的线程的分发`Updatable.update()`。

注销观察者(Updatable)可以在任何线程, 但为了避免观察者注销后，事件被分发在内部形成竞争条件, 推荐注册和注销观察者(Updatable)在同一个线程完成。

开发者有责任保证Looper的存活和被观察者们(Observables)一样。由于Looper导致的异常和内存泄漏是开发者的责任。
然而在实践中，绝大多数情况下都是使用主线程Looper，而主线程是随app一直存活着。

