### 优化对常量表达式的侦测

#### Optimizing Constant Expression Watching

如果在 watch 中使用的是表达式字符串，我们就可以做一些优化来提升 digest 循环运行速度。上一节我们已经看到如何将常量表达式的 `constant` 属性设置为 `true`。常量表达式返回的是一个固定值。这意味着常量表达式的 watch 被触发执行第一次，以后它都不会再变“脏”（dirty）了。同时也意味着我们可以把这个 watch 用安全的方式移除掉，这样就能减轻之后 digest 进行脏值检测的负担了。下面我们在 `scope_spec.js` 文件的 `describe('$digest')` 测试集中加入这个测试用例：

_test/scope\_spec.js_

```js
it('removes constant watches after first invocation', function() {
  scope.$watch('[1, 2, 3]', function() {});
  scope.$digest();

  expect(scope.$$watchers.length).toBe(0);
});
```

现在，这个测试用例执行后会抛出一个“10 iterations reached”（已重试了 10 次）的异常，因为这个表达式每次执行都会生成一个新的数组，因此使用引用比较的 watch 每次都会认为它是一个新值。但 `[1, 2, 3]` 实际上是一个常量，我们只需要执行一次运算就够了。

我们可以使用一个叫 _侦测委托_（watch delegates） 的新表达式特性来解决这个问题。侦测委托实际上是一个可以附加到表达式上的函数。

