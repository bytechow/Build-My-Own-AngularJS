### 解析 true, false 和 null
#### Parsing true, false, and null

我们需要支持的第三种字面量是布尔值字面量 `true` 和 `false`，还有 `null`。它们都是所谓的 _标识符_ token，这意味着它们在输入时是一个纯字母数字（alphanumeric）字符。随着不断开发，我们会遇到很多标识符。大多数情况它们用于通过名称查找作用域上的属性，但它们也可以是保留词，如果 `true`、`false` 或 `null`。在解析的时候，让它们变成对应的 JavaScript 值就好了。