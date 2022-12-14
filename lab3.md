# 总结

- 把之前实现的 mmap、munmap、get_time、task_info 等系统调用 port 过来
- 实现 sys_spawn 和 sys_set_priority 及 stride 进程调度算法
- 修复了 os5-ref 代码里一个 bug: `INITPROC` 退出会导致 `UPSafeCell` 的双重排他借用而引起 panic.

# 问答作业

> stride 算法原理非常简单，但是有一个比较大的问题。例如两个 stride = 10 的进程，使用 8bit 无符号整形储存 pass， p1.pass = 255, p2.pass = 250，在 p2 执行一个时间片后，理论上下一次应该 p1 执行。
>
> - 实际情况是轮到 p1 执行吗？为什么？

实际上轮不到 p1 执行。因为 p2 执行一个时间片后，他的 pass 会发生溢出从 250 变成 4，导致它还是 pass 最小的进程，因此还是 p2 会被执行。

> 我们之前要求进程优先级 >= 2 其实就是为了解决这个问题。可以证明， **在不考虑溢出的情况下** , 在进程优先级全部 >= 2 的情况下，如果严格按照算法执行，那么 PASS_MAX – PASS_MIN <= BigStride / 2。
>
> - 为什么？尝试简单说明（不要求严格证明）。

在最极端的情况下，考虑两个进程，一个进程 A 优先级为 2，另一个进程 B 优先级很大。若 A 进程先执行，那么现在的 PASS_MAX – PASS_MIN 取得最大值 `BigStride / 2`, 之后 B 会不停的执行来减小 PASS_MAX - PASS_MIN，在 `B.pass >= A.pass` 时，轮到 A 执行，差距再次到达 `BigStride / 2` 附近。在其他情况下，此差值至多取得在极端情况下的最大取值，因此得证。

> - 已知以上结论，**考虑溢出的情况下**，可以为 pass 设计特别的比较器，让 `BinaryHeap<Pass> `的 pop 方法能返回真正最小的 Pass。补全下列代码中的 `partial_cmp` 函数，假设两个 Pass 永远不会相等。
>
> TIPS: 使用 8 bits 存储 pass, BigStride = 255, 则: `(125 < 255) == false`, `(129 < 255) == true`.

```rust
use core::cmp::Ordering;

struct Pass(u64);

impl PartialOrd for Pass {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        let delta = self.0 as f32 - other.0 as f32;
        if delta < 0f32 {
            other.partial_cmp(&self).map(|ord| ord.reverse())
        } else if delta > 127.5 {
            Some(Ordering::Less)
        } else {
            Some(self.0.cmp(&other.0))
        }
    }
}

impl PartialEq for Pass {
    fn eq(&self, other: &Self) -> bool {
        false
    }
}
```

