---
title: "@unchecked Sendable 骗过了编译器：一次 Swift 并发字典崩溃复盘"
date: 2026-07-10 21:30:00
description: 这篇文章复盘我在 Owlet 里一天内连修的两次 Swift 崩溃。第一次是并发数据竞争：一个被 @unchecked Sendable 骗过编译器的普通 Dictionary，在两个并发 import 里被同时写坏内存，崩溃点还飘到毫不相干的第三个线程上。第二次是确定性的 Data 下标越界：removeFirst 之后 Data 的 startIndex 不归零，用字面量 0 当索引直接越界 SIGTRAP。文章从崩溃日志反推根因，讲清楚为什么细粒度锁不够、read-modify-write 为什么必须整段串行、Swift 6 里 async 上下文为什么不让你裸用 NSLock.lock()，以及切片类型的索引为什么不能信 0。
categories:
  - [AI]
tags:
  - Swift
  - 并发
  - macOS
  - Owlet
  - Agent
cover: /images/swift-unchecked-sendable-dictionary-crash.webp
---

Owlet 是我用 Claude Code + Codex vibe 出来的一个常驻 menu-bar app，用来统计每天 Claude Code 和 Codex 的 Token 用量、排名、实时会话活动。常驻类 app 最怕两件事：一是空闲耗电，二是莫名其妙崩。这次一天之内，我连着修了两个崩溃——而且它们像俄罗斯套娃：修好外面那个，里面那个才露出来。

第一个崩得相当有迷惑性——崩溃栈指向的那一行代码本身没有任何问题。真正的凶手，是一个我亲手写下 `@unchecked Sendable`、然后就再也没管过它线程安全的普通 `Dictionary`。第二个则相反，是个藏在一行 `0` 里、只要用起来就必崩的确定性 bug。两个坑的共同点是：它们都利用了 Swift 里某个"看起来无害"的默认行为。

![Swift 并发字典崩溃复盘](/images/swift-unchecked-sendable-dictionary-crash.webp)

## 一句话总结

`@unchecked Sendable` 不是"让这个类型线程安全"，而是"我向编译器保证它已经安全了，你别查了"。如果你写下它却没做任何同步，普通 `Dictionary` 在并发写下会直接损坏底层存储、破坏整个进程的堆内存，崩溃点还会飘到无辜的其他线程上。

## 崩溃现场：两个线程站在同一个栈上

先看崩溃日志最关键的部分。信号是 `EXC_BAD_ACCESS (SIGSEGV)`，访问了一个明显非法的地址 `0x8000000000000028`：

```
Thread 4 Crashed:
0   libswiftCore.dylib   __RawDictionaryStorage.find<A>(_:) + 36
1   libswiftCore.dylib   _NativeDictionary.setValue(_:forKey:isUnique:) + 192
2   libswiftCore.dylib   Dictionary._Variant.setValue(_:forKey:) + 196
3   libswiftCore.dylib   Dictionary.subscript.setter + 604
4   Owlet                ClaudeCodeAdapter.discoverSessions() (ClaudeCodeAdapter.swift:200)
```

崩在 `__RawDictionaryStorage.find` / `setValue`——也就是往字典里写一个 key 的时候，字典的内部哈希存储已经是坏的了。这不是"key 不存在"这种业务问题，是底层内存结构被破坏。

再往下看，会发现一个决定性的线索：**另一个线程也停在同一个函数里。**

```
Thread 2:
5   Owlet   ClaudeCodeAdapter.discoverSessions() (ClaudeCodeAdapter.swift:199)
```

两个后台线程，同一时刻，都在执行 `discoverSessions()`，都在读写同一个 `Dictionary`。一个在 199 行读，一个在 200 行写。到这里，数据竞争（data race）已经基本实锤了。

还有一个更迷惑人的细节：崩溃报告里第三个线程停在完全不相干的地方——

```
Thread 7 (GRDB.DatabasePool.reader):
   key path getter for Character.isNewline
   Collection.split(whereSeparator:)
   CodexDataReader.rolloutTokenUsageEvents(path:)
```

`CodexDataReader` 是读 Codex 数据的另一套代码，跟 Claude 的 adapter 八竿子打不着，凭什么它也崩？

答案是：**堆内存被写坏之后，整个进程都在雷区里跑。** Thread 4 把字典的底层存储写坏，破坏的是共享的堆。此后任何线程在任何位置做一次内存访问，都可能踩到那片被污染的区域，然后随机崩溃。所以排查并发内存崩溃时，有一个重要原则:

> 崩溃日志里"触发信号的那个线程"（`triggered: true`）才是凶手，其他线程往往只是受害者。

如果你盯着 Thread 7 的 `CodexDataReader` 去查，会彻底跑偏。

## @unchecked Sendable：一句你亲手写下的空头承诺

顺着凶手 `ClaudeCodeAdapter` 看类型声明：

```swift
final class ClaudeCodeAdapter: AgentAdapter, @unchecked Sendable {
    private var sessionEntryCache: [String: CachedSessionEntry] = [:]
    // ...
}
```

问题全在这两行里。

`Sendable` 是 Swift 并发模型里的一个协议，意思是"这个类型的值可以安全地跨并发域传递"。对于 `class` 这种引用类型，编译器默认不认为它是 `Sendable`——因为多个线程可能同时持有同一个引用、改同一块状态。

而 `@unchecked Sendable` 是一个逃生舱：它告诉编译器"这个类型我人工审计过了，线程安全我自己保证，你不用检查"。它常用于两种情况：一种是内部用了锁/串行队列做同步，只是编译器看不出来；另一种是……写的人当时觉得"应该没事吧"。

我这次就是第二种。`sessionEntryCache` 是一个普通的 `Dictionary`，没有锁、没有串行队列、没有 actor，什么保护都没有。`@unchecked Sendable` 关掉了编译器的并发检查，于是这个隐患一路绿灯上了线。

这里要记住一件事：**`@unchecked Sendable` 不提供任何运行时保护，它只是关掉编译期的报错。** 你写下它的那一刻，线程安全就从"编译器帮你保证"变成了"你自己口头保证"。

那为什么会有两个线程并发进来？看调用方：

```swift
// AppState.refresh() 里
_ = try await Task.detached(priority: .utility) {
    for runtime in runtimesToImport {
        imported += try await capturedIngestion.importSessions(from: runtime.adapter)
        // -> 最终调用 adapter.discoverSessions()
    }
}.value
```

`refresh()` 会在多个时机被触发——app 启动 bootstrap、文件监听 debounce 后的自动刷新、手动刷新。每次触发都开一个 `Task.detached` 跑 import。当两次刷新时间上挨得很近（比如启动时 bootstrap 和文件监听几乎同时发生），两个 detached task 就并发进入了同一个 adapter 实例的 `discoverSessions()`，一起写那个没有任何保护的字典。

## 为什么不能用细粒度锁：read-modify-write 的原子性

找到根因，很多人第一反应是"给字典加个锁不就行了"。但加在哪、加多粗，是这次真正值得记下来的点。

先看 `discoverSessions()` 里对 cache 的访问模式：

```swift
if let entry = sessionEntryCache[cacheKey] {        // ① 读旧 entry
    if fileSize > entry.fileSize {
        // 只读新增的字节，从 entry.offset 处 seek
        parseIncremental(file, entry: entry, ...)    // ② 用 entry.offset seek 文件
    }
}
// ...
sessionEntryCache[cacheKey] = newEntry               // ③ 写回带新 offset 的 entry
```

这是一个典型的 **read-modify-write**：读出旧的 `entry`（里面记着"上次读到文件的哪个字节偏移"），基于这个 offset 只读文件新追加的部分（增量解析，避免每次全量重读几十 MB），再把新的 offset 写回去。

如果只给字典的每次下标访问加一把细粒度锁（读一次锁一次、写一次锁一次），会发生什么？

> 线程 A 读到 offset=1000，线程 B 也读到 offset=1000。两个线程都从 1000 开始读文件、各自解析、各自把"新 offset"写回。其中一个的结果被覆盖，offset 错乱，同一段 Token 用量被重复计入。

字典本身不崩了，但数据错了。细粒度锁保护了"每一次字典操作"的原子性，却没保护"读—改—写"这个序列整体的原子性。这是并发编程里最容易栽的一类坑：**锁的粒度对齐的应该是不变量（invariant），而不是单个操作。**

这里的不变量是"cache 里的 offset 必须和已经解析进去的字节严格对应"。要维持它，整个 `discoverSessions()` pass 必须串行执行。所以正确的修法是一把粗粒度锁，锁住整趟：

```swift
private let discoverLock = NSLock()

func discoverSessions() async throws -> [AgentSessionDraft] {
    discoverLock.withLock { discoverSessionsLocked() }
}

private func discoverSessionsLocked() -> [AgentSessionDraft] {
    // 原来那一整段读目录、读文件、增量解析、写 cache 的逻辑
}
```

有人会担心：持锁期间还在做磁盘 I/O，第二个并发调用不就干等着吗？会。但这恰恰是最优解——第二个调用等第一个做完之后拿到锁，此时 cache 已经是热的，绝大多数文件的 `mtime` / `fileSize` 都没变，直接命中缓存、零 I/O 秒回。串行化在这里几乎没有额外代价，反而消除了两个 task 各跑一遍的重复劳动。

## Swift 6 的暗礁：async 上下文里不能 lock()

按上面思路，我一开始是直接在 `discoverSessions()` 里写的：

```swift
func discoverSessions() async throws -> [AgentSessionDraft] {
    discoverLock.lock()
    defer { discoverLock.unlock() }
    // ...
}
```

编译能过，但�报了个 warning，而且明说了在 Swift 6 语言模式下会直接变成 error：

```
warning: instance method 'lock' is unavailable from asynchronous contexts;
Use async-safe scoped locking instead;
this is an error in the Swift 6 language mode
```

原因很本质：`async` 函数可能在任意 `await` 处挂起，挂起之后恢复执行的线程**不一定是原来那个线程**。而 `NSLock` 是要求"谁上锁谁解锁、同一线程"的。如果你在 `lock()` 和 `unlock()` 之间跨了一个 `await`，挂起前后换了线程，就会变成"A 线程上锁、B 线程解锁"，直接是未定义行为。

编译器无法证明你两者之间没有 `await`，于是干脆禁止在 async 上下文里裸用 `lock()`/`unlock()`。

正解是用 scoped 的 `withLock { }`：把需要保护的逻辑抽成一个**同步**函数（我这里是 `discoverSessionsLocked()`），闭包体内不可能出现 `await`，锁的获取和释放必然在同一个同步执行段里完成。这也顺便逼你把"要保护的临界区"想清楚、圈干净，而不是散着写一堆 lock/unlock。

## 加餐：同一天的第二个崩溃，藏在一行 `0` 里

修完并发那个崩溃、重新打包上线，没过多久又收到一份新的崩溃日志。这次信号不一样，是 `EXC_BREAKPOINT (SIGTRAP)`——Swift 运行时主动触发的陷阱，通常意味着数组或 `Data` 越界。

崩溃点在一个监听 JSONL 会话文件、检测"用户中断"的后台组件里：

```swift
private func scanBufferForInterrupt() {
    var scanStart = 0
    while scanStart < leftoverData.count {
        guard let index = leftoverData[scanStart...].firstIndex(of: newline) else { break }
        // ... 处理一行 ...
    }
    if consumed > 0 {
        leftoverData.removeFirst(consumed)   // ← 雷埋在这
    }
}
```

`leftoverData` 是一个不断追加、扫描完就把已处理部分 `removeFirst` 掉的 `Data` buffer。看起来天经地义。但它踩中了 `Data` 一个反直觉的设定：

> **Swift `Data` 的 index 是绝对的。`removeFirst(n)` 之后，它的 `startIndex` 会前移到 n，而不是回到 0。**

第一次扫描时 `startIndex == 0`，`leftoverData[0...]` 一切正常。扫完执行 `removeFirst(consumed)`，`startIndex` 变成了 consumed（比如 87）。等文件第二次增长、再进来扫描时，`scanStart` 还是硬编码的 `0`，而 `leftoverData[0...]` 里的这个 `0` 已经落在 `startIndex` 之前——越界，SIGTRAP。

这不是偶发的并发时序问题，是个**确定性 bug**：只要有一个 Claude 会话在跑、它的 JSONL 被追加写入两次，就必崩。之所以之前一直没暴露，是因为上一个并发崩溃发生得更早，先把进程带走了；修掉前一个，这个就顶到了最前面。

修复本身很简单，但记性要长——别用字面量 `0` 当 `Data` 的索引，一切从 `startIndex` 起：

```swift
var lineStart = leftoverData.startIndex
while lineStart < leftoverData.endIndex {
    guard let newlineIndex = leftoverData[lineStart...].firstIndex(of: newline) else { break }
    let lineData = leftoverData.subdata(in: lineStart..<newlineIndex)
    // ...
    lineStart = leftoverData.index(after: newlineIndex)
}
```

`Collection` 的下标是绝对索引，这件事在 `Array` 上很少咬到你——因为 `Array` 的 index 永远从 0 开始。但 `Data`、`ArraySlice`、`Substring` 这些"可能是别人一部分"的类型，`startIndex` 随时可能不是 0。**凡是切片类型，就别信 0。**

## 我的判断框架：actor / 锁 / @unchecked Sendable 怎么选

这次踩坑之后，我给自己定了一套在 Swift 并发里选同步手段的判断顺序：

| 场景 | 首选 | 理由 |
|---|---|---|
| 新写的、可以 async 化的共享可变状态 | `actor` | 编译器强制隔离，从根上杜绝数据竞争，不用自己管锁 |
| 已有同步 API、不方便全改 async，临界区清晰 | `NSLock` + `withLock` | 改动最小，粗粒度锁住不变量即可 |
| 高频只读、极少写 | 读写锁 / 原子类型 | 读不互斥，避免锁竞争成为热点 |
| 确实由底层机制保证了安全 | `@unchecked Sendable` + **注释写清凭什么安全** | 逃生舱，但必须留下审计依据 |

几条给自己的硬规矩：

- **看到 `@unchecked Sendable`，就当成一个待验证的 TODO。** 它旁边必须有一句注释说明"靠什么保证线程安全"（哪把锁、哪个串行队列、哪个 actor）。说不出来，就是隐患。
- **锁的粒度对齐不变量，不对齐单个操作。** 遇到 read-modify-write，先问自己"这三步之间的中间状态能不能被别人看到"，不能就整段串行。
- **排查并发内存崩溃，先认准 `triggered` 线程。** 堆一旦被写坏，崩溃点会飘，别被无辜的受害线程带偏。
- **async 里要锁，用 `withLock`，别用裸 `lock()`。** 顺手把临界区抽成同步函数，思路和编译器都会更清楚。
- **切片类型的索引从 `startIndex` 起，别信字面量 `0`。** `Data`、`ArraySlice`、`Substring` 的 `startIndex` 随时可能不是 0，尤其在 `removeFirst` / 切片之后。遍历用原生 index，不用 `Int` 硬编码。

`@unchecked Sendable` 这类逃生舱不是不能用——`actor` 化整个 adapter 也要改协议、改一堆调用点，工程上未必划算。真正的问题是：我当初写下它的时候,把"我保证安全"当成了"它已经安全"。这两者之间隔着的，正是一次会飘到无辜线程上的堆内存崩溃。
