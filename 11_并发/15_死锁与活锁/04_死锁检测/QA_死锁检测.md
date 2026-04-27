
# 死锁检测

## Q1：如何检测 Java 程序中的死锁？

**A**：常用工具：

1. **jstack**：`jstack <pid>`，输出末尾会显示 "Found one Java-level deadlock" 及等待链
2. **jConsole**：连接到 JVM → 线程选项卡 → "检测死锁"
3. **JMX 编程**：`ManagementFactory.getThreadMXBean().findDeadlockedThreads()`
4. **Arthas**：`thread -b` 命令直接找到死锁线程

生产环境最常用 jstack 和 Arthas。

---

## Q2：如何在代码中自动检测死锁？

**A**：通过 ThreadMXBean：

```java
ThreadMXBean mxBean = ManagementFactory.getThreadMXBean();
long[] deadlockedThreads = mxBean.findDeadlockedThreads();
if (deadlockedThreads != null) {
    ThreadInfo[] infos = mxBean.getThreadInfo(deadlockedThreads, true);
    // 打印或告警
}
```

可以配合定时任务定期检测。

---

## Q3：jstack 输出中死锁信息长什么样？

**A**：

```
Found one Java-level deadlock:
=============================
"Thread-1":
  waiting to lock monitor 0x00007f8a4c012b58 (object 0x000000076ab4f7a0, a java.lang.Object),
  which is held by "Thread-2"
"Thread-2":
  waiting to lock monitor 0x00007f8a4c013418 (object 0x000000076ab4f7b0, a java.lang.Object),
  which is held by "Thread-1"
```

明确显示：Thread-1 持有的锁被 Thread-2 等待，Thread-2 持有的锁被 Thread-1 等待。


# 死锁定位步骤

## Q1：线上出现死锁，如何快速定位？

**A**：标准流程：

1. `jps -l` 找到 Java 进程 PID
2. `jstack <pid>` 打印线程栈
3. 搜索输出中的 `deadlock` 或 `BLOCKED`
4. 从线程栈中找到具体的类名、方法名和行号
5. 分析锁的获取顺序，找出循环等待

更快的方案：用 Arthas 的 `thread -b` 命令直接定位死锁线程。

---

## Q2：为什么要多次 jstack 打印线程栈？

**A**：区分**瞬时阻塞**和**真正死锁**：

- 瞬时阻塞：线程短暂等待锁，几秒后恢复，多次打印结果不同
- 真正死锁：线程永久阻塞，多次打印结果一致

间隔 3-5 秒打印 2-3 次，如果 BLOCKED 状态的线程和等待链不变，则确认是死锁。

---

## Q3：CPU 利用率高低对死锁排查有什么影响？

**A**：

- **CPU 低，线程阻塞**：典型死锁特征——线程都在等锁，不做任何计算
- **CPU 高**：可能是死循环（活锁），不是死锁
- **CPU 正常**：可能是某些线程死锁，其他线程正常工作

通过 `top -H -p <pid>` 查看各线程 CPU 占用，辅助判断。

---

```bash
# 死锁定位步骤
# 1. 找到 Java 进程
jps -l

# 2. 打印线程栈
jstack <pid> > thread_dump.txt

# 3. 搜索死锁信息
grep -A 20 "deadlock" thread_dump.txt

# 4. 多次打印确认（间隔 3-5 秒）
jstack <pid> > dump1.txt
sleep 5
jstack <pid> > dump2.txt
diff dump1.txt dump2.txt  # BLOCKED 线程相同 → 确认死锁

# 5. Arthas 快速定位
# thread -b  （直接输出死锁线程栈）
```