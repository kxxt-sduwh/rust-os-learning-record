# 总结

- 自己实现了一下 `os3` 中缺少的 `os3-ref` 的内容。
- 实现了两个系统调用
  - 查询时间
  - 查询当前 Task 的信息

# 其他

做这个 lab 的时候我自己试着把 `os3-ref` 的内容实现了一下，结果有个 bug 我找了很久才发现，我在写 `trap.S` 的时候把 `__restore` 最后释放 `TrapContext` 的 `addi sp, sp, 34*8` 写成了 `addi sp, sp, -34*8`。这导致了一些有趣的行为：系统会在执行一会之后不停输出七八个 `Panicked at` 之后停止，也有可能遇上 `StoreFault`. 我调了很久也没找到原因。后来一看是 `trap.S` 的锅🫠🫠🫠，修了之后就没问题了。

另一个问题就是我给 `TaskControlBlock` 里塞个大数组存系统调用次数，结果操作系统就卡在执行第一个程序之前了。我感觉这好像是内存问题，这堆数组占的空间有点大，把内存中的其他重要部分给覆写掉了。最后在微信群里问了一下，我把 lto 打开，opt-level 改到 2， 系统就能正常运行了（然而还是没搞懂具体原因），感觉这个问题值得在文档里提一下，不然第一个 lab 就十分劝退。

# 简答

> 正确进入 U 态后，程序的特征还应有：使用 S 态特权指令，访问 S 态寄存器后会报错。 请同学们可以自行测试这些内容 (运行 Rust 三个 bad 测例 (ch2b_bad_*.rs) ， 注意在编译时至少需要指定 LOG=ERROR 才能观察到内核的报错信息) ， 描述程序出错行为，同时注意注明你使用的 sbi 及其版本。

sbi: RustSBI version 0.3.0-alpha.4

`bad_address` 程序触发了 `StoreFault` 同步异常。

`bad_instruction` 和 `bad_register` 程序触发了 `IllegalInstruction` 同步异常。

> 深入理解 trap.S 中两个函数 __alltraps 和 __restore 的作用，并回答如下问题:
> 1. L40：刚进入 __restore 时，a0 代表了什么值。请指出 __restore 的两种使用情景。

在第二章中，`__restore` 有两种使用场景，一种是开始执行 APP，
另一种场景是从 trap 中返回到 U 模式继续执行 APP，在这两种场景中， `a0` 都是指向要恢复的 `TrapContext` 的指针。

而在第三章中， `__restore` 只剩下了开始执行 APP 这一种使用场景，也不再使用 `a0` 的值。根据 `TaskContext.goto_restore` 的代码，我们可以判断出 `a0` 的值为 0.

> 2. L46-L51：这几行汇编代码特殊处理了哪些寄存器？这些寄存器的的值对于进入用户态有何意义？请分别解释。

这部分代码恢复了 `sstatus`, `sepc`, `ssratch` 这三个 CSR。

- `sepc` 存储了返回 U 模式后要执行的指令的地址。
- `sstatus` 的 `SPP` 字段会被正确的恢复，确保最后 `sret` 进入 U 模式。
- `sscratch` 在这部分代码执行完成后指向用户栈栈顶，在 __restore 全部完成后指向分配这个 `TrapContext` 之前的内核栈栈顶，即恢复到进入 Trap 之前的状态，同时 `sp` 指向用户栈栈顶。

> 3. L53-L59：为何跳过了 x2 和 x4？

- `x2` 是 `sp`, 已经在前面的代码中特殊处理了（先把它加载到 `sscratch` 里，最后和当前 `sp` 交换）。
- `x4` 是 `tp` 寄存器，除非我们手动出于一些特殊用途使用它，否则一般也不会被用到。

> 4. L63：该指令之后，sp 和 sscratch 中的值分别有什么意义？
> `csrrw sp, sscratch, sp`

`sp` 指向用户栈栈顶， `sscratch` 指向内核栈栈顶。

> 5. __restore：中发生状态切换在哪一条指令？为何该指令执行之后会进入用户态？

发生在 `sret` 指令。因为我们在 `__restore` 中恢复了 `sstatus` 寄存器的值之后， `SPP` 就是 `U`, 所以会进入用户态。

> 6. L13：该指令之后，sp 和 sscratch 中的值分别有什么意义？
> `csrrw sp, sscratch, sp`

`sp` 指向内核栈栈顶， `sscratch` 指向用户栈栈顶。

> 7. 从 U 态进入 S 态是哪一条指令发生的？

非法指令、访存错误、还有系统调用会 Trap 到 S 模式。