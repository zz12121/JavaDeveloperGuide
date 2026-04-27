
# 并行流与ForkJoinPool

## 核心结论

Java 8 的 `stream().parallel()` 底层使用 **ForkJoinPool.commonPool** 执行。Stream 框架将数据源（数组/集合）拆分为多个子任务，通过 ForkJoinPool 的分治机制并行处理，最后合并结果。需要注意：并行流有拆分/合并开销，不是所有场景都适合并行化。

## 深度解析

### 执行原理

```
list.parallelStream().map().filter().collect()

数据源 [1, 2, 3, 4, 5, 6, 7, 8]
    ↓ Spliterator 拆分
    ├─ [1, 2, 3, 4]    ─┐
    │   ├─ [1, 2]       │  ForkJoinPool 分治
    │   └─ [3, 4]       │  各线程并行执行 map/filter
    └─ [5, 6, 7, 8]    ─┘
    ↓ 结果合并
    → collect()
```

### 适合并行的场景

| 适合 ✅ | 不适合 ❌ |
|---------|----------|
| 数据量大（万级以上） | 数据量小（百级以下） |
| CPU 密集型计算 | I/O 阻塞操作 |
| 无状态操作（map/filter） | 有状态操作（sorted/limit） |
| 数据源易拆分（ArrayList） | 数据源难拆分（LinkedList） |
| 操作本身耗时 | 操作很快（如简单 getter） |

### 并行性能影响因素

```java
// ✅ 适合并行：数据量大 + CPU 计算
List<Integer> bigList = IntStream.range(0, 1_000_000).boxed().toList();
bigList.parallelStream()
    .map(x -> heavyCompute(x))  // CPU 密集
    .collect(Collectors.toList());

// ❌ 不适合并行：数据量小 + 简单操作
List<Integer> smallList = List.of(1, 2, 3, 4, 5);
smallList.parallelStream()
    .map(x -> x * 2)  // 操作太简单，拆分/合并开销 > 计算开销
    .collect(Collectors.toList());

// ❌ 不适合并行：I/O 阻塞
list.parallelStream()
    .forEach(url -> httpClient.get(url));  // 阻塞 commonPool！
```

### 数据源与拆分效率

| 数据源 | 拆分效率 | 原因 |
|--------|---------|------|
| ArrayList | 高 | 数组，O(1) 按索引拆分 |
| IntStream.range | 高 | 范围，天然可拆分 |
| HashSet | 中 | 底层数组，但可能有空桶 |
| LinkedList | 差 | 链表，拆分需遍历 |
| Stream.iterate | 极差 | 无界流，无法预知大小 |

### 线程安全问题

```java
// ❌ 并行流中使用非线程安全集合
List<Integer> result = new ArrayList<>();  // 非线程安全！
list.parallelStream().forEach(result::add);  // 可能丢数据或抛异常

// ✅ 使用 collect 或线程安全集合
List<Integer> result = list.parallelStream()
    .collect(Collectors.toList());  // collect 内部保证线程安全
```

### 完整业务场景：并行流 + 自定义池隔离

**场景**：从数据库批量拉取用户列表，对每个用户调用外部 API 获取画像数据，最后聚合结果。**必须使用自定义 ForkJoinPool 隔离 commonPool**，否则阻塞 API 调用会影响其他并行操作。

```java
public class UserProfileAggregator {

    private final UserRepository userRepository;
    private final ProfileApiClient profileApi;
    private final ExecutorService ioExecutor; // I/O 线程池

    public UserProfileAggregator(UserRepository userRepository, ProfileApiClient profileApi) {
        this.userRepository = userRepository;
        this.profileApi = profileApi;
        // I/O 密集操作用独立线程池（推荐核心数 × 2）
        this.ioExecutor = Executors.newFixedThreadPool(
            Runtime.getRuntime().availableProcessors() * 2
        );
    }

    /**
     * 并行聚合用户画像
     * ✅ 使用自定义 ForkJoinPool + parallelStream
     * ✅ I/O 操作委托给独立 ioExecutor
     */
    public List<UserProfile> aggregateProfiles(List<Long> userIds) {
        // 创建自定义 ForkJoinPool，避免污染 commonPool
        ForkJoinPool customPool = new ForkJoinPool(
            Runtime.getRuntime().availableProcessors()  // CPU 密集，并行度 = 核心数
        );

        try {
            return customPool.submit(() ->
                userIds.parallelStream()
                    .map(userId -> {
                        // 查询用户基本信息（CPU 操作，在 FJP 中执行）
                        User user = userRepository.findById(userId);
                        if (user == null) return null;

                        // 调用外部 API（I/O 阻塞，委托给 ioExecutor）
                        Profile profile = CompletableFuture
                            .supplyAsync(() -> profileApi.getProfile(userId), ioExecutor)
                            .join();  // 在 FJP 中 join（I/O 已在另一线程池执行，不阻塞 FJP）

                        return new UserProfile(user, profile);
                    })
                    .filter(Objects::nonNull)
                    .collect(Collectors.toList())
            ).get();
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return Collections.emptyList();
        } catch (ExecutionException e) {
            throw new RuntimeException("聚合失败", e.getCause());
        } finally {
            customPool.shutdown();
        }
    }

    /**
     * ❌ 错误示范：直接在 parallelStream 中执行阻塞 I/O
     * 会占满 commonPool，导致其他组件（如 CompletableFuture）卡死
     */
    public List<UserProfile> wrongAggregate(List<Long> userIds) {
        return userIds.parallelStream()
            .map(userId -> {
                User user = userRepository.findById(userId);
                Profile profile = profileApi.getProfile(userId); // ❌ 阻塞 commonPool！
                return new UserProfile(user, profile);
            })
            .collect(Collectors.toList());
    }
}
```

**架构图**：

```
userIds.parallelStream()（在 customPool 中执行）
    │
    ├─ map: userRepository.findById()    → CPU 操作，在 ForkJoinPool 中
    │
    └─ map: profileApi.getProfile()       → I/O 阻塞，委托给 ioExecutor
                                                ↓
                                          独立线程池，不占用 FJP

commonPool（完全不受影响）
    ├─ 其他并行流
    └─ CompletableFuture.supplyAsync()
```

## 关联知识点
