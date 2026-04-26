---
title: AtomicReference
tags:
  - Java/并发
  - 场景型
module: 06_原子类
created: 2026-04-18
---

# AtomicReference（对象引用的原子更新，CAS + volatile）

## 先说结论

`AtomicReference<V>` 提供对**对象引用**的原子 CAS 更新，常用于实现无锁数据结构、无锁单例、并发状态机等场景。与 `AtomicInteger` 不同的是，它操作的是整个对象引用，通过 `compareAndSet` 保证引用更新的原子性。

## 深度解析

### 核心源码

```java
public class AtomicReference<V> implements java.io.Serializable {
    private static final Unsafe U = Unsafe.getUnsafe();
    private static final long VALUE = U.objectFieldOffset(AtomicReference.class, "value");
    private volatile V value;

    public final V get() { return value; }
    public final void set(V newValue) { value = newValue; }
    public final boolean compareAndSet(V expected, V newValue) {
        return U.compareAndSetObject(this, VALUE, expected, newValue);
    }
    public final V getAndSet(V newValue) {
        return (V)U.getAndSetObject(this, VALUE, newValue);
    }

    // JDK8+ 函数式更新
    public final V updateAndGet(UnaryOperator<V> updateFunction) {
        V prev, next;
        do {
            prev = get();
            next = updateFunction.apply(prev);
        } while (!compareAndSet(prev, next));
        return next;
    }
}
```

### 典型应用

**场景一：CAS 无锁栈**

```java
public class ConcurrentStack<E> {
    private static class Node<E> {
        final E item;
        Node<E> next;
        Node(E item) { this.item = item; }
    }

    private final AtomicReference<Node<E>> top = new AtomicReference<>();

    public void push(E item) {
        Node<E> newNode = new Node<>(item);
        Node<E> oldTop;
        do {
            oldTop = top.get();
            newNode.next = oldTop;
        } while (!top.compareAndSet(oldTop, newNode));
    }

    public E pop() {
        Node<E> oldTop;
        do {
            oldTop = top.get();
            if (oldTop == null) return null;
        } while (!top.compareAndSet(oldTop, oldTop.next));
        return oldTop.item;
    }
}
```

**场景二：并发状态机**

```java
class TaskState {
    final String status;
    final String data;
    TaskState(String status, String data) {
        this.status = status;
        this.data = data;
    }
}

AtomicReference<TaskState> state = new AtomicReference<>(
    new TaskState("INIT", "")
);

// 状态转换：INIT → RUNNING → DONE
TaskState current = state.get();
TaskState next = new TaskState("RUNNING", "starting...");
state.compareAndSet(current, next);
```

### 注意：比较的是引用

```java
// ⚠️ 比较的是对象引用，不是 equals()
User u1 = new User("Alice");
User u2 = new User("Alice"); // equals 但不同对象
AtomicReference<User> ref = new AtomicReference<>(u1);
ref.compareAndSet(u1, new User("Bob"));  // ✅ true
ref.compareAndSet(u2, new User("Bob"));  // ❌ false（u1 ≠ u2）
```

## 易错点/踩坑

- ❌ 认为 AtomicReference 比较 equals——它比较的是引用相等（`==`），不是 `equals`
- ❌ 用 AtomicReference 更新对象内部字段——它只能原子替换整个引用，不能原子修改对象内部属性
- ✅ 需要原子修改对象内部属性时，使用不可变对象 + 整体替换（Immutable + CAS）

## 代码示例

```java
// 不可变对象 + CAS 模式
public final class Config {
    final int timeout;
    final int maxSize;
    final String endpoint;

    Config(int timeout, int maxSize, String endpoint) {
        this.timeout = timeout;
        this.maxSize = maxSize;
        this.endpoint = endpoint;
    }
}

public class ConfigManager {
    private final AtomicReference<Config> configRef =
        new AtomicReference<>(new Config(5000, 100, "localhost"));

    // 原子更新整个配置（不可变对象替换）
    public void updateTimeout(int newTimeout) {
        Config old, next;
        do {
            old = configRef.get();
            next = new Config(newTimeout, old.maxSize, old.endpoint);
        } while (!configRef.compareAndSet(old, next));
    }
}
```

## 关联知识点
