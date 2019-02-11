## Promise 扩展（Promise 

如果你使用过 Angular 的 `$http`，你可能会发现它返回的 Promise 还会包含一些方法，而这些方法目前我们还没有实现。实际上也就是`success`和`error`两个方法，而且这两个方法只会在`$http`返回的 Promise 才会拥有的。