### 避免不必要的对象迭代
#### Preventing Unnecessary Object Iteration

现在我们会对对象进行了两次遍历，如果对象体积很大，遍历的成本也会很高。而且我们是在每轮 digest 都会执行的 watch 函数中进行遍历，更要避免这个函数的工作量过大。

因此，我们会在侦测对象变化的过程中应用一项优化措施。

首先，我们会记录新旧两个对象的长度：

- 对于旧对象，我们会用一个变量进行记录，这个变量会在对象新增属性时就递增，在对象移除属性时递减。
- 对于新对象，我们会在 `innerWatchFn` 第一次对对象的遍历时统计它的长度。

这样当我们完成第一次循环时，我们就知道了新旧两个对象的长度了。然后，我们只需要在第二次循环中检查旧对象的长度是否比新对象的更长。如果两者的长度相等，那就说明并没有移除过属性，可以直接跳过第二次循环了。下面是具体代码：

```js
Scope.prototype.$watchCollection = function(watchFn, listenerFn) {
  var self = this;
  var newValue;
  var oldValue;
  var oldLength;
  var changeCount = 0;

  var internalWatchFn = function(scope) {
    var newLength;
    newValue = watchFn(scope);
  
    if (_.isObject(newValue)) {
      if (isArrayLike(newValue)) {
        // if (!_.isArray(oldValue)) {
        //   changeCount++;
        //   oldValue = [];
        // }
        // if (newValue.length !== oldValue.length) {
        //   changeCount++;
        //   oldValue.length = newValue.length;
        // }
        // _.forEach(newValue, function(newItem, i) {
        //   var bothNaN = _.isNaN(newItem) && _.isNaN(oldValue[i]);
        //   if (!bothNaN && newItem !== oldValue[i]) {
        //     changeCount++;
        //     oldValue[i] = newItem;
        //   }
        // });
      } else {
        if (!_.isObject(oldValue) || isArrayLike(oldValue)) {
          // changeCount++;
          // oldValue = {};
          oldLength = 0;
        }
        newLength = 0;
        _.forOwn(newValue, function(newVal, key) {
          newLength++;
          if (oldValue.hasOwnProperty(key)) {
            // var bothNaN = _.isNaN(newVal) && _.isNaN(oldValue[key]);
            // if (!bothNaN && oldValue[key] !== newVal) {
            //   changeCount++;
            //   oldValue[key] = newVal;
            // }
          } else {
            changeCount++;
            oldLength++;
            oldValue[key] = newVal;
          }
        });
        if (oldLength > newLength) {
          changeCount++;
          _.forOwn(oldValue, function(oldVal, key) {
            if (!newValue.hasOwnProperty(key)) {
              oldLength--;
              // delete oldValue[key];
            }
          });
        }
      }
    } else {
      // if (!self.$$areEqual(newValue, oldValue, false)) {
      //   changeCount++;
      // }
      // oldValue = newValue;
    }
    // return changeCount;
  };

  // var internalListenerFn = function() {
  //   listenerFn(newValue, oldValue, self);
  // };
  
  // return this.$watch(internalWatchFn, internalListenerFn);
};
```

注意，我们现在需要区分处理属性新增和属性改变这两种情况了，因为如果是新增属性的话，我们还要对 `oldLength` 进行递增了。