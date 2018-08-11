### Hash 键和 Hash 映射（Hash Keys And Hash Maps）

我们要实现的第一个函数叫 hashKey。这个函数会接收一个 JavaScript 的值，并返回一个字符串作为它的 hash key。这与 Java 的 Object.hashCode\(\) 和 Ruby 的 Object\#hash 机制类似。一个 hash key 应该是唯一的。我们将会马上用到 haskKey 这个函数。

我们将会新建一个 src/hash\_map.js 文件来实现 Hash Map，当然，也有对应的测试文件 test/hash\_map\_spec.js。

一般来说，一个值的 hash key 由两部分来组成。一部分是定义值类型，另一部分定义值的字符串表达。The two parts are separated with a colon。特别的，对于 undefined 这个值，它两部分都会是 'undefined'

test/hash\_map\_spec.js

```js
var hashKey = require('../src/hash_map').hashKey;
describe('hash', function() {
  'use strict';
  describe('hashKey', function() {
    it('is undefned:undefned for undefned', function() {
      expect(hashKey(undefned)).toEqual('undefned:undefned');
    });
  });
});
```

对于 null 来说，它的类型会是 'object'，而字符表达会是 'null'：

test/hash\_map\_spec.js

```js
it('is object:null for null', function() {
  expect(hashKey(null)).toEqual('object:null');
});
```

对于布尔值来说，类型会是 'boolean'，而字符表达是 'true' 或者 'false'：

test/hash\_map\_spec.js

```js
it('is boolean:true for true', function() {
  expect(hashKey(true)).toEqual('boolean:true');
});
it('is boolean:false for false', function() {
  expect(hashKey(false)).toEqual('boolean:false');
});
```

对于数字类型来说，类型自然是 'number'，而字符表达就是该数字的字符串表示：

test/hash\_map\_spec.js

```js
it('is number:42 for 42', function() {
  expect(hashKey(42)).toEqual('number:42');
});
```

对于字符串来说，类型是 'string'，字符表达就是它本身。 看到这里，我们可以清楚为什么我们需要标明值的类型，否则数字 42 和字符串 '42' 的 hash key 就会是一样的了，无法区分开来：

test/hash\_map\_spec.js

```js
it('is string:42 for "42"', function() {
  expect(hashKey('42')).toEqual('string:42');
});
```

目前所有



