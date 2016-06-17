如[[上一篇|Reactive-programming#push-event-pull-data]]所讲, 
`Repository`是一个被观察者(Observable)，可以提供数据，还可以定义在提供数据发生变化时的事件。 
获取数据方法：`Repository.get()`.

# 简单的 repositories

简单的repository可以使用`Repositories`类中的工具方法来创建。  
有如下选择:
* _static repository_：提供相同的数据源而且不生成通知事件,只有get()方法;
* _mutable repository_：可提供变化的数据源(accept输入->get输出)，数据变化时生成通知事件(依赖方法`Object.equals(Object)`).

本质上说，简单的repository总是提供最新数据，不论它们是否被激活。

```
private void setUpRepository() {

    mRepository = Repositories.repository(object);
    // or
    mRepository = Repositories.mutableRepository(object);
}
```

# 复杂的 repositories

复杂的数据仓库(Repository)可以响应其他数据仓库(Repositories)、任意被观察者(Observables)(也可以是该Repository的事件), 
并把从其他数据源获取的数据经过同步或者异步内部转换处理后作为数据仓库(Repository)的产出值。 
从响应事件中数据仓库(Repository)的数据提供者总保持数据最新的，但由于处理的复杂性，在数据仓库(Repository)不活动时，可以选择不保持数据为最新的。 
任何数据消费者都需要通过注册观察者(Updatable)来表示自己需要读取数据的意图。
数据仓库(Repository)进入活动状态，但数据不用立即更新，消费者看到的数据仍然是旧的，直到数据仓库(Repository)分发第一个事件。

Agera 提供了[[repository compiler|Compiled-repositories]]类，帮助以接近自然语言的表达式来声明、实现复杂的数据仓库(Repository)。
