### 即时解决－$q.when（Immediate Resolution-$q.when）

正如我们需要 $q.reject，有时我们也会想要创建一个已经被 resolve 的 Promise。例如，我们需要产生一个会返回一个 Promise 的函数，这个函数可能为同步，也可能为异步：

```js
it('can make an immediately resolved promise', function() {
  var fulflledSpy = jasmine.createSpy();
  var rejectedSpy = jasmine.createSpy();
  
  var promise = $q.when('ok');
  promise.then(fulflledSpy, rejectedSpy);
  
  $rootScope.$apply();

  expect(fulflledSpy).toHaveBeenCalledWith('ok');
  expect(rejectedSpy).not.toHaveBeenCalled();
});
```

$q.when 能做的事不仅于此，它还可以将其他类 Promise 对象变成 Angular 原生 Promise。下面我们会创建一个类 Promise 对象，然后作为 $q.when 的调用参数，这时这个类 Promise 对象就能像 Angular 原生的一般被使用：

```js
it('can wrap a foreign promise', function() {
  var fulflledSpy = jasmine.createSpy();
  var rejectedSpy = jasmine.createSpy();

  var promise = $q.when({
    then: function(handler) {
      $rootScope.$evalAsync(function() {
        handler('ok');
      });
    }
  });
  promise.then(fulflledSpy, rejectedSpy);

  $rootScope.$apply();
  
  expect(fulflledSpy).toHaveBeenCalledWith('ok');
  expect(rejectedSpy).not.toHaveBeenCalled();
});
```

下面的代码展示了 when 的实现方式：

```js
function when(value) {
  var d = defer();
  d.resolve(value);
  return d.promise;
}
```

现在，我们应该暴露这个接口：

```js
return {
  defer: defer,
  reject: reject,
  when: when
};
```

注意，我们不需要再为兼容其他 Promise 实现做额外的工作了。原因是 $q.when 的实现实际上依赖于 then 方法，我们之前实现 then 时时候已经考虑到了这种情况，对传入的 Promise 对象进行了兼容处理。这里我们注意 $q.when 能帮助我们将其他 Promise 实现融入 Angular 使用就可以了。

当然，$q.when 还有一个有用的特性，就是它能够支持传入最多三个 Promise 回调——resolved、rejected 和 notify。你可以直接在调用 $q.when 时传入，而不需要在它的返回值中再进行绑定：

```js
it('takes callbacks directly when wrapping', function() {
  var fulflledSpy = jasmine.createSpy();
  var rejectedSpy = jasmine.createSpy();
  var progressSpy = jasmine.createSpy();
  
  var wrapped = $q.defer();
  $q.when(
    wrapped.promise,
    fulflledSpy,
    rejectedSpy,
    progressSpy
  );

  wrapped.notify('working...');
  wrapped.resolve('ok');
  $rootScope.$apply();

  expect(fulflledSpy).toHaveBeenCalledWith('ok');
  expect(rejectedSpy).not.toHaveBeenCalled();
  expect(progressSpy).toHaveBeenCalledWith('working...');
});
```

这个可以借助链式处理的机制来实现。我们会为 when 方法的 Promise 返回值增加一个 Promise 节点，最终的返回值变成这个 Promise 节点：

```js
function when(value, callback, errback, progressback) {
  var d = defer();
  d.resolve(value);
  return d.promise.then(callback, errback, progressback);
}
```

另一个生成已解决的 Promise 的方式，是通过 $q.resolve 方法，实际上就是为 $q.when 换了个名字而已：

```js
return {
  defer: defer,
  reject: reject,
  when: when,
  resolve: when
};
```