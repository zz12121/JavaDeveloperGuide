---
title: 循环语句
tags:
  - Java/基础语法
  - 对比型
  - 问答
module: 01_基础语法
created: 2026-04-18
---
# 循环语句（for、while、do-while、foreach）

## Q1：四种循环的语法与区别

**A**： 
- `for`：`for(初始化; 条件; 迭代){}`，适合已知循环次数
- `while`：`while(条件){}`，先判断后执行，适合未知循环次数
- `do-while`：`do{}while(条件);`，先执行后判断，至少执行一次
- `foreach`：`for(类型 变量 : 集合){}`，语法简洁，适合只读遍历

---

## Q2：foreach 的底层原理？

**A**： 
- foreach 是语法糖，编译后调用 Iterator
- 遍历中通过集合自身 API 修改会触发 fail-fast 机制
- 应使用 Iterator.remove() 或 removeIf()（Java 8+）

---

## Q3：for vs while

**A**：
- `for` 的循环变量作用域仅限循环体内
- `while` 的循环变量需要在外部声明，作用域更大

---

## Q4：为什么 foreach 中不能修改集合？

**A**：foreach 底层使用 Iterator，每个 Iterator 维护一个 expectedModCount。
遍历中通过集合 API 修改会使 modCount 变化，Iterator 检测到不一致就抛 ConcurrentModificationException。
用 Iterator 自带的 remove() 则会同步更新 expectedModCount。

---

## Q5：do-while 和 while 的区别？什么场景用 do-while？

**A**：do-while 至少执行一次，while 可能一次都不执行。
典型场景：输入验证（先获取用户输入，再判断是否合法）、菜单显示（先显示菜单，再判断用户选择）。

---

## Q6：for 循环中声明变量的作用域？

**A**：`for(int i=0; ...)` 中 i 的作用域仅限于 for 循环体内（含条件判断和迭代部分），循环结束后 i 不可访问。
如果需要在循环外使用循环变量，要在 for 外部声明。

---

## Q7：三者核心区别

**A**： 
- `break`：跳出当前整个循环（或 switch），循环后代码继续
- `continue`：跳过本次迭代，进入下一次循环
- `return`：结束当前方法，返回值给调用者

---

## Q8：带标签的 break/continue？

**A**：
- `break label;` 可以跳出指定外层循环
- `continue label;` 可以跳过指定外层循环的本次迭代
- 不推荐使用，影响可读性

---

## Q9：finally 中 return 的问题？

**A**：
- try 中 return 的值先保存副本，finally 修改基本类型不影响返回值
- finally 中如果有 return，会覆盖 try/catch 的 return
- finally 中 return 会吞掉未捕获的异常

---

## Q10：finally 中 return 会覆盖 try 中的 return 吗？

**A**：会。try 中 return 时，返回值会被暂存。
如果 finally 中也有 return，则会用 finally 的返回值替换。
这是 Java 规范的明确行为，但不推荐在 finally 中写 return。

---

## Q11：continue 在 while 循环中要注意什么？

**A**：如果 while 循环的迭代变量 `i++` 写在 continue 之后，会导致死循环。
因为 continue 跳回条件判断，但 `i++` 没有执行，条件永远满足。建议迭代变量写在 for 循环的迭代部分。

---

## Q12：break 能跳出 if 语句吗？

**A**：不能。break 只能用于 switch 和循环（for/while/do-while）。如果要跳出 if，只能用 return 结束方法，或用标志位配合 break 跳出外层循环。

---

## 关联知识点
