---
date: 2024-08-03
title: Node.js 中 Array、Map 和 Object 查询性能对比
tags: nodejs, javascript
---

几个月前给 Nest 提了一个优化 `WsAdapter` 寻找消息处理器性能的 PR [#13378](https://github.com/nestjs/nest/pull/13378)，只是简单的将遍历数组改为使用 `Map`。Nest 中其实存在大量通过遍历数组进行查询的代码，大多数都是启动时只执行一次，所以性能影响不大。但是 `WsAdapter` 中的代码是每次接收到消息都会执行，对性能有一定影响。提这个 PR 的时候，我并没有做性能测试，只是觉得 `Map` 的查询性能应该比遍历数组快，快多少我也不知道。直到最近有人在 PR 中问我有没有做性能测试，有没有对比过不同数据量下 `Array`, `Map` 和 `Object` 的查询性能。感觉事情可能没有我想的那么简单，于是我写了一个简单的 benchmark 程序进行测试。

## 测试代码

```javascript
const { performance } = require("perf_hooks");

function generateKey() {
  const length = Math.floor(Math.random() * 10) + 5;
  return Math.random()
    .toString(36)
    .substring(2, length + 2);
}

function generateTestData(size) {
  const keys = new Set();
  const data = [];
  for (let i = 0; i < size; i++) {
    let key = generateKey();
    while (keys.has(key)) {
      key = generateKey();
    }
    keys.add(key);
    data.push({ key, value: `value${i}` });
  }
  return data;
}

function prepareDataStructures(data) {
  const arr = data;
  const map = new Map(data.map((item) => [item.key, item]));
  const obj = Object.fromEntries(data.map((item) => [item.key, item]));
  return { arr, map, obj };
}

function runQuery(structure, key, { arr, map, obj }) {
  switch (structure) {
    case "array":
      return arr.find((item) => item.key === key);
    case "map":
      return map.get(key);
    case "object":
      return obj[key];
  }
}

function runBenchmark(dataSize, iterations) {
  const data = generateTestData(dataSize);
  const structures = prepareDataStructures(data);

  const structureTypes = ["array", "map", "object"];
  const results = {};
  const queryKeys = Array.from(
    { length: iterations },
    () => structures.arr[Math.floor(Math.random() * dataSize)].key
  );

  structureTypes.forEach((structure) => {
    const start = performance.now();
    queryKeys.forEach((key) => runQuery(structure, key, structures));
    const end = performance.now();
    results[structure] = end - start;
  });

  console.log(`Data size: ${dataSize}, Iterations: ${iterations}`);
  structureTypes.forEach((structure) => {
    console.log(`${structure}: ${results[structure]}ms`);
  });
  console.log();
}

const dataSizes = [10, 100, 1000, 1100, 10000, 100000, 1000000];
const iterations = 10000;

dataSizes.forEach((size) => runBenchmark(size, iterations));
```

## 测试结果

测试环境：Node.js v20.10.0

```
Data size: 10, Iterations: 10000
array: 4.13458251953125ms
map: 0.550999641418457ms
object: 0.47237491607666016ms

Data size: 100, Iterations: 10000
array: 2.8057079315185547ms
map: 0.23449993133544922ms
object: 0.6267919540405273ms

Data size: 1000, Iterations: 10000
array: 10.07683277130127ms
map: 0.3053750991821289ms
object: 1.0291252136230469ms

Data size: 1100, Iterations: 10000
array: 10.884374618530273ms
map: 0.2077503204345703ms
object: 0.16404247283935547ms

Data size: 10000, Iterations: 10000
array: 142.96483325958252ms
map: 1.2331666946411133ms
object: 0.2637920379638672ms

Data size: 100000, Iterations: 10000
array: 1674.5742092132568ms
map: 1.2257919311523438ms
object: 0.8130826950073242ms

Data size: 1000000, Iterations: 10000
array: 17534.821665763855ms
map: 2.955416679382324ms
object: 0.935333251953125ms
```

## 结论

可以很明显的看出在任何数据量下，`Map` 和 `Object` 的查询性能都远远优于 `Array`，这并没有什么意外。但测试结果中有另一个意外的发现，`Map` 和 `Object` 的性能差别还挺大的。在数据量小于 1000 的情况下，`Map` 的性能优于 `Object`。但在数据量大于 1000 之后，`Map` 的性能反而逐渐不如 `Object`。从测试结果中可以看出在数据量为 1100 时，`Map` 和 `Object` 的查询性能比数据量为 1000 时反而要更好。这应该是当数据量超过 1000 后，`Map` 和 `Object` 内部的实现都进行了优化，但 `Map` 的优化效果不如 `Object`。

我们可以得出一个简单的结论，当数据量较小时使用 `Map`，当数据量较大时使用 `Object`。但这个结论并不是绝对的，可能还会受到 `key` 值的影响，测试中的 `key` 都是随机生成的长短不一的。有兴趣的话可以测一下当 `key` 为数字时的性能、当 `key` 都有重复前缀时的性能等等。

Nest `WsAdapter` 中的消息处理器数量一般不会太多，所以简单改为 `Map` 就可以提升十倍的查询性能。
