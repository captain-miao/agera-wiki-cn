Agera 可以按需自定义observable。

# Proxy observables(代理被观察者)

代理被观察者是简单的代其他被观察者分发通知事件，几乎不做处理。工具类(`Observables`)提供了如下创建代理被观察者的方法：

- `compositeObservable`: 组合多观察(事件)源;
- `conditionalObservable`: 通过被观察者设置的条件来控制事件通知;
- `perMillisecondObservable` and `perLoopObservable`:  限制事件通知频率。

# BaseObservable

基类`BaseObservable`实现了注册、注销观察者和通知给[[threading contract|Observables-and-updatables#threading]]。
继承`BaseObservable`是创建一个自定义被观察者的最简单的方法。
在任何线程，当要发送事件通知的时候，子类只需要调用`dispatchUpdate()`方法。
举个例子，下面的类将View的点击事件实现为被观察者：

```java
public class ViewClickedObservable extends BaseObservable
    implements View.OnClickListener {
    
  @Override
  public void onClick(View v) {
    dispatchUpdate();
  }
}
```

`BaseObservable`的子类可以重写`observableActivated()` 和 `observableDeactivated()`方法，可以监测被观察者的激活周期，分别在激活和去激活的时候调用。
这两个方法是在`BaseObservable`的线程上调用的-- 也就是实例化`BaseObservable`的线程。
这减轻任何同步锁在最典型的情况下的需要，使用main Looper作为所有被观察者的worker Looper。

# UpdateDispatcher

当不能或者不方便继承`BaseObservable`的时候(比如：已经继承一个基类)，也可以很容易的实现自定义被观察者，直接实现`Observable`接口。 
`UpdateDispatcher`实例(更新事件分发器)帮助自定义的被观察者管理客户端更新事件(和继承`BaseObservable`一样), 具有相同的线程规则。

自定义的被观察者应该创建一个私有更新分发器调用`Observables.updateDispatcher()`，或者接受`ActivationHandler`实例。
`ActivationHandler`定义了`observableActivated` 和 `observableDeactivated`方法，可以监测被观察者的激活周期。
像`BaseObservable`一样，更新分发器需要有Looper的线程上工作，所以需要在有Looper的线程上创建。

自定义的被观察者应该简单地转发所有的注册和注销观察者调用来更新。要向所有观察者发送事件调用`UpdateDispatcher.update()`。
名符其实，更新分发器可以方便更新，所以自定义的被观察者自己就是一个代理，需要注册一个内部观察者到其他事件源，可以简单地使用它的更新分发。
这熟悉的名字意味着更新调度是方便可更新的，因此如果自定义观察到的是自己观察到的一个代理，需要注册一个内部更新到其他事件源，它可以简单地使用它的更新调度。

一个额外的提示点, 
`UpdateDispatcher`也是`Observable`的一个子类型，也可以作为被观察者使用。就像可变repository作为数据生产者和消费者的桥梁，`UpdateDispatcher`作为事件生产者和消费者的桥梁。
数据生产者使用`Receiver`作为`MutableRepository`提供数据方, 数据消费者使用`Repository`作为数据消费方。 
同样, 事件生产者使用`Updatable`作为`UpdateDispatcher`发送事件方, 事件消费者使用`Observable`作为事件消费方。

参考更新分发(update dispatcher)示例:[[下一篇|Incrementally-Agerifying-legacy-code]]。














