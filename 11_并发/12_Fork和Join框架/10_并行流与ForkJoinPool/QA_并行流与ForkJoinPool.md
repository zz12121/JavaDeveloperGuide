
# 并行流与ForkJoinPool

## Q1：parallelStream 底层是怎么实现的？

**A**：parallelStream 底层使用 **ForkJoinPool.commonPool** 执行：
1. 通过 `Spliterator` 将数据源拆分为多个子范围
2. 将每个子范围封装为 `ForkJoinTask` 提交到 commonPool
3. 各工作线程并行执行中间操作（map/filter 等）
4. 通过 join 合并各子结果，执行终端操作（collect/reduce）

整个过程就是 ForkJoinPool 的分治（Fork → 并行执行 → Join 合并）。

---

## Q2：什么场景下 parallelStream 比 stream 更快？什么场景更慢？

**A**：
**更快**：数据量大（万级以上）+ CPU 密集操作 + 数据源易拆分（ArrayList/数组）
**更慢**：数据量小 + 操作简单 + 数据源难拆分（LinkedList）+ I/O 阻塞

拆分/合并本身有开销（创建 Spliterator、ForkJoinTask 入队出队），只有当并行收益超过这些开销时才更快。一般建议**数据量 > 1万 且 单元素处理耗时 > 1微秒**时考虑并行。

---

## Q3：parallelStream 中使用 forEach 往 ArrayList 添加元素有什么问题？

**A**：`ArrayList` 不是线程安全的，多个线程同时 `add` 可能导致：
1. 数组越界（size 检查与赋值之间被其他线程修改）
2. 数据丢失（覆盖写入）
3. 抛出 `ArrayIndexOutOfBoundsException`

正确做法：
- 用 `collect(Collectors.toList())` — collect 内部使用线程安全合并
- 或用 `Collections.synchronizedList` / `CopyOnWriteArrayList`（但性能较差）
- 或用 `forEachOrdered` 保证顺序但牺牲并行性

---
```java
// parallelStream 底层使用 ForkJoinPool.commonPool
List<Integer> numbers = IntStream.range(0, 1_000_000).boxed().toList();

// 并行流：自动拆分任务 → ForkJoinPool 并行执行
int sum = numbers.parallelStream()
    .filter(n -> n % 2 == 0)
    .mapToInt(Integer::intValue)
    .sum();

// ❌ 错误：parallelStream + ArrayList.add（线程不安全）
List<Integer> result = new ArrayList<>();
numbers.parallelStream().forEach(result::add);  // 可能数据丢失

// ✅ 正确：用 collect
List<Integer> safe = numbers.parallelStream()
    .filter(n -> n % 2 == 0)
    .collect(Collectors.toList());  // 线程安全合并
```

