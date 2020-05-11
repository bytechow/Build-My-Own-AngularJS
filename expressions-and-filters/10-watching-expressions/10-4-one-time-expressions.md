### 单次绑定表达式
#### One-Time Expressions

作为 Angular 开发者，我们经常会遇到有些 watch 只需要赋值一次，后续不会再发生改变的情况。比较常见的就是像下面这样的对象列表：

```xml
<li ng-repeat="user in users">
  {{user.firstName}} {{user.lastName}}
</li>
```

这段代码使用 `ng-repeat` 生成了一个侦听器集合，但对于每个 `user` 来说，它同时也会增加两个 watcher——第一个是姓（first name），第二个是名（last name）。