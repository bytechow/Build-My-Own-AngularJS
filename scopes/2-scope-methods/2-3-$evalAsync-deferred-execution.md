### $evalAsync——延迟执行
#### $evalAsync - Deferred Execution

在 JavaScript 中，要延迟执行一些代码的情况是很常见的——让这段代码延迟到当前执行上下文结束时再执行。一般的做法是使用 [setTimeout](https://developer.mozilla.org/en-US/docs/Web/API/WindowOrWorkerGlobalScope/setTimeout) 方法，传入的 delay 参数为 0。