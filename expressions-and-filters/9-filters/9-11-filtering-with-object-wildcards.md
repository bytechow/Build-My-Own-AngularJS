### 使用对象通配符进行过滤
#### Filtering With Object Wildcards

如果你想要“让这个对象中的任意属性都匹配这个值”，我们就要在标准对象中使用一个特殊的通配符号 `$` 了：

_test/filter_filter_spec.js_

```js
it('filters with a wildcard property', function() {
  var fn = parse('arr | filter:{$: "o"}');
  expect(fn({
    arr: [
      { name: 'Joe', role: 'admin' }, { name: 'Jane', role: 'moderator' }, { name: 'Mary', role: 'admin' }
    ]
  })).toEqual([
    { name: 'Joe', role: 'admin' }, { name: 'Jane', role: 'moderator' }
  ]);
});
```

与常规的对象属性的不同之处在于，使用通配符属性之后过滤器也会对嵌套对象进行匹配——在任意层级上：

_test/filter_fitler_spec.js_

```js
it('filters nested objects with a wildcard property', function() {
  var fn = parse('arr | filter:{$: "o"}');
  expect(fn({
    arr: [
      { name: { first: 'Joe' }, role: 'admin' }, { name: { first: 'Jane' }, role: 'moderator' }, { name: { first: 'Mary' }, role: 'admin' }
    ]
  })).toEqual([
    { name: { first: 'Joe' }, role: 'admin' }, { name: { first: 'Jane' }, role: 'moderator' }
  ]);
});
```

目前这个属性跟其他原始类型过滤标准并没什么两样。那为什么使用 `{$: "o"}` 而不直接用 "o" 呢？主要原因是当我们在内嵌的对象标准中使用通配符时，这种方法能把通配符的上下文指定为它的父级：

_test/filter_filter_spec.js_

```js
it('filters wildcard properties scoped to parent', function() {
  var fn = parse('arr | filter:{name: {$: "o"}}');
  expect(fn({
    arr: [
      { name: { first: 'Joe', last: 'Fox' }, role: 'admin' },
      { name: { first: 'Jane', last: 'Quick' }, role: 'moderator' },
      { name: { first: 'Mary', last: 'Brown' }, role: 'admin' }
    ]
  })).toEqual([
    { name: { first: 'Joe', last: 'Fox' }, role: 'admin' },
    { name: { first: 'Mary', last: 'Brown' }, role: 'admin' }
  ]);
});
```