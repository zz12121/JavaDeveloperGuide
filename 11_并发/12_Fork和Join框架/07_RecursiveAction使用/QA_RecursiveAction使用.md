
# RecursiveAction使用

## Q1：RecursiveAction 和 RecursiveTask 什么区别？什么时候用 RecursiveAction？

**A**：
- **RecursiveTask\<V\>**：`compute()` 有返回值，适合需要汇总结果的场景（求和、排序）
- **RecursiveAction**：`compute()` 返回 void，适合只需执行动作的场景（遍历文件、批量修改数组）

选择标准：**是否需要返回结果**。如果子任务执行完不需要合并出一个最终值，就用 RecursiveAction。

---

## Q2：RecursiveAction 没有 join() 返回值，怎么知道任务执行完了？

**A**：
1. `invokeAll(tasks...)`：等价于对所有子任务 fork + join，阻塞等待全部完成
2. `task.join()`：虽然无返回值，但仍可调用 join 等待完成
3. `pool.invoke(task)`：提交根任务并等待完成

```java
// 方式一：invokeAll
invokeAll(leftTask, rightTask);  // 等待两个都完成

// 方式二：join
leftTask.fork();
rightTask.fork();
leftTask.join();   // 等待完成
rightTask.join();  // 等待完成
```

---

## Q3：RecursiveAction 的典型应用场景有哪些？

**A**：
1. **文件系统遍历**：递归遍历目录树，对文件执行操作（索引、压缩、统计）
2. **数组批量处理**：并行修改/转换数组元素
3. **树结构遍历**：AST 分析、DOM 操作
4. **批量 I/O 写入**：并行写入多个文件

核心特征：**操作独立、无需汇总、可分治拆分**。


