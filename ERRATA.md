## p31
Angular scopes 实际上列出了五个内容，但却说是“four main areas”

## p74
单元测试名 'never executes $applyAsynced function in the same cycle' 中的 '$applyAsynced' 应改为 '$applyAsync'

## p96
单元测试名 'inherits the parent's properties' 中的 'parent's' 的单引号跟标记字符串的前后两个单引号冲突了，需要使用转义符

## p106
该页的第一个代码块下第三段文字中 "Armed with the knowledge about this difference between digest and $apply,..." 中的 digest 应改为 `$digest`

## p115
"First of all, we should share the queue between scopes, just like we did with the $evalAsync and $postDigest queues:" 

这里的 $postDigest 应该改为 $$postDigest

## p225
页中的 `test/parse.js` 需要改为 `test/parse_spec.js`

## p328
9-7 节的最后一段中，`compare` 应该改成 `comparator`

## p341
最后一段，`fourth` 应该改为 `fifth`，`inWildcard` 应该改为 `isWildcard`， `isWildcard` 是第 5 个参数，而不是第 4 个。
