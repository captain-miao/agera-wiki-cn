由于_push event, pull data_模型和多线程情况下，观察者可能看不到数据全部的更新记录(ps:因为总是获取最新的数据)。这是特意设计的: 因为大多数情况下(尤其更新app UI), 本来就只需要关心最新的数据。
然而, 如果客户端(一般很少)想知道全部的变化历史记录, Agera 提供`Repository`的子类型: `Reservoir`，可以满足这种场景。

`Reservoir`是一个响应版本队列。数据可以通过`Receiver`接口加入队列，然后发起通知事件，然后观察者可以从队列中读取数据(`Repository.get()`)。
`Reservoir`队列的访问是完全同步的，所以不会出现两个客户端从队列中读取到同一个数据(ps:就是线程安全)。
如果相同的值放入队列多次，这也会认为是不同的实例(ps:就是会生成多次通知事件)。
if the same value (the same Java object reference) is enqueued multiple times, they are different instances in the context of a reservoir).
返回数据类型是`Result`, 所以如果客户端尝试从空队列读取数据的时候，可以接收`Result.absent()`作为一个失败的通知。

基于这种行为，`Reservoir`非常适合必须处理事件源的每一个数据的响应。
这种响应可以使用[[compiled repository|Compiled-repositories]]实现。
如果合适的话，可以用`Reservoir`作为事件源，使用`.attemptGetFrom(reservoir).orSkip()`作为数据处理流程的开始指令。
一旦repository激活，`Reservoir`中的观察者模式就会建立好(ps:消费者和生产者的模式)。

使用`Reservoir`可以实现简单的并行任务，使用多个repository提供事件源和数据源，任务放入队列，然后每个repository去执行数据处理流程。
注意：为了做到并行执行，repository都需要提交到线程池执行或者运行在不同的工作Looper(worker Loopers)。









