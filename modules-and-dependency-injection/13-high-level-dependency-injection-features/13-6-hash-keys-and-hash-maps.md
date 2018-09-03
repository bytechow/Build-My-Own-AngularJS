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

目前所有的单元测试，都以用 typeof 运算符来满足对值类型的判断和获取：

```js
'use strict';
function hashKey(value) {
  var type = typeof value;
  return type + ':' + value;
}
module.exports = {hashKey: hashKey};
```

对于函数和数组来说，事情就没有这么简单了。

一般来说，一个对象的 hash key 由字符串 'object' 和一个唯一识别符来组成：

test/hash_map_spec.js

```js
it('is object:[unique id] for objects', function() {
  expect(hashKey({})).toMatch(/^object:\S+$/);
});
```

我们希望 hash key 是稳定的，具体来说是两次获取某个对象的 hash key，应该得到的结果是一致的：

```js
it('is the same key when asked for the same object many times', function() {
  var obj = {};
  expect(hashKey(obj)).toEqual(hashKey(obj));
});
```

有趣的事，即使我们对某个对象进行更改，它的 hashKey 依然会是稳定的：

```js
it('does not change when object value changes', function() {
  var obj = {a: 42};
  var hash1 = hashKey(obj);
  obj.a = 43;
  var hash2 = hashKey(obj);
  expect(hash1).toEqual(hash2);
});
```

这也就是说，生成对象的 hashKey 并不会使用到对象里的某个值，只会与对象本身有关。反过来说，即使两个对象的值完全一致，它们仍然是两个独立的对象，生成出来的 hashKey 也是不同的：

```js
it('is not the same for different objects even with the same value', function() {
  var obj1 = {a: 42};
  var obj2 = {a: 42};
  expect(hashKey(obj1)).not.toEqual(hashKey(obj2));
});
```

在本章，我们会将这个规则也应用到函数上。函数的 hashKey 会由字符串 'function' 和一串数字 ID 组成：

```js
it('is function:[unique id] for functions', function() {
  var fn = function(a) { return a; };
  expect(hashKey(fn)).toMatch(/^function:\S+$/);
});
```

```js
it('is the same key when asked for the same function many times', function() {
  var fn = function() { };
  expect(hashKey(fn)).toEqual(hashKey(fn));
});
```

```js
it('is not the same for different identical functions', function() {
  var fn1 = function() { return 42; };
  var fn2 = function() { return 42; };
  expect(hashKey(fn1)).not.toEqual(hashKey(fn2));
});
```

所以 Angular 里面的 hashKey 方法与 Java 和 Ruby 的还不是完全相等。对于函数和对象来说，它只是作为基于引用的唯一性标识。

这里的技巧是，我们在调用 hashKey 时会向对象增加一个名为 $$hashKey 的属性，这个属性会保存一个代表对象的唯一 ID（也就是 hashKey 方法返回的值中冒号以后的部分）。如果你用过 Angular，你可能会发现当使用 ngRepeat 等指令时，它里面用到的对象就会有这个属性。

```js
it('stores the hash key in the $$hashKey attribute', function() {
  var obj = {a: 42};
  var hash = hashKey(obj);
  expect(obj.$$hashKey).toEqual(hash.match(/^object:(\S+)$/)[1]);
});
```

$$haskKey 属性就是我们能够保证对象的 hashKey 稳定的秘密。调用第一次后会生成这个属性，之后就会直接用这个属性，而不是重新生成：

```js
it('uses preassigned $$hashKey', function() {
  expect(hashKey({$$hashKey: 42})).toEqual('object:42');
});
```

在实现代码，我们需要区别对待对象和非对象数据类型。对于对象（或函数）我们会查找它的 $$hashKey，若不存在，则生成。为了方便，我们直接使用 Lodash 提供的 uniqueId 方法来生成唯一 ID：

src/hash_map.js

```js
'use strict';
var _ = require('lodash');

function hashKey(value) {
  var type = typeof value;
  var uid;
  if (type === 'function' ||
    (type === 'object' && value !== null)) {
    uid = value.$$hashKey;
    if (uid === undefned) {
      uid = value.$$hashKey = _.uniqueId();
    }
  } else {
    uid = value;
  }
  return type + ':' + uid;
}
```

如果你想自定义 $$hashKey，你也可以使用函数进行预配置。调用 hashKey 方法时，如果发现传入的对象包含 $$hashKey 属性，且其属性值是一个函数时，就会执行这个函数数，并把它的返回值作为对象的 haskKey：

```js
it('supports a function $$hashKey', function() {
  expect(hashKey({
    $$hashKey: _.constant(42)
  })).toEqual('object:42');
});
it('calls the function $$hashKey as a method with the correct this', function() {
  expect(hashKey({
    myKey: 42,
    $$hashKey: function() {
      return this.myKey;
    }
  })).toEqual('object:42');
});
```

测试文件也用到了 lodash，我们也要记得引入

```js
'use strict';

var _ = require('lodash');
var hashKey = require('../src/hash_map').hashKey;
```

现在我们需要对传入hashKey方法的对象进行检测，如果其 $$hashKey 值为一个函数，我们就调用它来生成 ID

```js
function hashKey(value) {
  var type = typeof value;
  var uid;
  if (type === 'function' ||
    (type === 'object' && value !== null)) {
    uid = value.$$hashKey;
    if (typeof uid === 'function') {
      uid = value.$$hashKey();
    } else if (uid === undefned) {
      uid = value.$$hashKey = _.uniqueId();
    }
  } else {
    uid = value;
  }
  return type + ':' + uid;
}
```

实现 hashKey 方法后，我们可以着手实现 hash map 了。我们将会使用 HashMap 构造函数来生成 hash map。这个构造函数将会实现 key 为任何类型的映射结构。与远房亲戚——Java中的 HashMap 类似，它会支持 put 和 get 方法：

```js
'use strict';

var _ = require('lodash');
var hashKey = require('../src/hash_map').hashKey;
var HashMap = require('../src/hash_map').HashMap;

describe('hash', function() {

  describe('hashKey', function() {
    ///
  });

  describe('HashMap', function() {
    it('supports put and get of primitives', function() {
      var map = new HashMap();
      map.put(42, 'fourty two');
      expect(map.get(42)).toEqual('fourty two');
    });
  });
});
```

我们期望 HashMap 和 hashKey 的输入输出行为是一致。（这更类似 ES6 的 Maps 特性似，所以叫 HashMap 可能还是不大恰当）：

```js
it('supports put and get of objects with hashKey semantics', function() {
  var map = new HashMap();
  var obj = {};
  map.put(obj, 'my value');
  expect(map.get(obj)).toEqual('my value');
  expect(map.get({})).toBeUndefned();
});
```

由于我们实现了 hashKey 方法，HashMap 的 put 和 get 方法实现起来就很简单了。hashKey 的返回值是字符串，我们就可以使用一个普通的 JavaScript 对象来作为存储空间。我们会直接使用 HashMap 实例作为存储空间：

```js
function HashMap() {
}

HashMap.prototype = {
  put: function(key, value) {
    this[hashKey(key)] = value;
  },
  get: function(key) {
    return this[hashKey(key)];
  }
};

module.exports = {
  hashKey: hashKey,
  HashMap: HashMap
};
```

这种实现会产生一个副产物，我们可以直接使用属性访问符直接获取某个value（如 map['number:42']，虽然我们不推荐在实际应用代码中使用。

HashMap 的第三个方法叫做 remove，顾名思义，它是用于移除 map 中的某个名值对的：

```js
it('supports remove', function() {
  var map = new HashMap();
  map.put(42, 'fourty two');
  map.remove(42);
  expect(map.get(42)).toBeUndefned();
});
```

在 remove 方法实现代码中，我们需要先通过 hashKey 获取到 key，然后就可以使用 delete 运算符删除掉对应的名值对：

```js
HashMap.prototype = {
  put: function(key, value) {
    this[hashKey(key)] = value;
  },
  get: function(key) {
    return this[hashKey(key)];
  },
  remove: function(key) {
    key = hashKey(key);
    var value = this[key];
    delete this[key];
    return value;
  }
}
```



