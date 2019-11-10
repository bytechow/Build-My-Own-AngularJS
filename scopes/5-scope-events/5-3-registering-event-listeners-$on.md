### 注册事件监听器：$on
#### Registering Event Listeners: $on

要在事件触发时得到通知，你需要先注册这个事件。在 AngularJS 中是通过调用 `$on` 来注册事件的。这个函数会接收两个参数：感兴趣的事件名称，还有需要在事件触发时调用的回调函数。

> AngularJS 中的订阅者被称为 _listener_，下面我们会继续使用这个名词。

