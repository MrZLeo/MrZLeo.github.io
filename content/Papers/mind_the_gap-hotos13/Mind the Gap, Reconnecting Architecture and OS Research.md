---
conference:
  - HOTOS'13
---
![[Pasted image 20240130192930.png]]

## OS 和 Arch 研究的 Gap 是什么？

- Arch 的研究学者往往忽视 OS 带来的关键开销
- OS 的研究学者没有充分地考虑底层硬件
- 将二者紧密结合起来对提高系统的性能和能效至关重要

## 现状

对于 Arch：把 OS 当作是黑盒
- 几乎只使用纯用户态的 benchmark
- 不计算用户态与内核态的开销
- 不讨论硬件设计对 OS 的影响


对于 OS：“on commodity hardware”

## 什么导致了 Gap？

### 缺少 OS 执行重要性的证据（关注度不够）
![[Pasted image 20240130194521.png|700]]

### 缺少合适的模拟环境
- 模拟器不能跑真实 OS
- X86 模拟器不成熟、难开发

### 缺少合适的 benchmark
- 没有合适的 benchmark 能够为 arch 研究者用来分析 OS 的行为和影响

### 历史上对硬件缺陷的容忍
- 长期以来，人们对硬件特性的缺失是可以接受的，因此 OS 研究者很少向硬件设计提出要求
- 因此 arch 研究者也很少关注 OS 的需求和 OS 的行为

## Arch 应该采用的研究原则

### P1 : Facilitate resource multiplexing.

- 促进资源的复用能够让 OS 的设计和实现上拥有更大的自由度
- 一个反例是 Intel 的 Single-chip Cloud Computer[^1] (SCC)，他们提供了一个 8 KB 的 Scratchpad 来快速交换 inter-core 的消息，但是内核切换需要(≈10µs)，8KB 的消息复制大概只需要 10ns

### P2: Keep mechanisms orthogonal.

- 不同的机制尽可能正交，将功能的组合和配置交给内核
- 一个反例是 AMD-V 虚拟化扩展的 tagged TLB 功能，这个功能自带自动 flush TLB 的功能，导致非 VM 上下文切换也会 flush TLB

### P3: Avoid over-abstraction.

- 硬件层面不要做过多的抽象
- 一个反例是 Intel 的超线程功能，超线程隐藏了一个核内两个线程的特性（为了向前兼容），导致 OS 无法为核内的两个线程采用快速的核间通信技术

### P4: Stay independent of kernel architecture.

- 不要过多假设内核的架构
- 尽管 Linux 内核大行其道（13 年并没有今天这么夸张），但保持 kernel-independent 的设计是好的

## 总结

作为 OS 研究者，不应该认为某些问题“可以用合适的硬件解决”。这篇文章虽然大篇幅在讲 arch 研究应该注重软件设计，但 OS 研究也应该注意不能只专注于当前的硬件限制下如何提升，毕竟一个复杂的软件虚拟化设计可以被硬件实现轻松击败。OS 研究还更应该关注如何通过硬件设计来提升软件能力，甚至设计自己的硬件。毕竟，“一个认真对待自己软件的人应该设计自己的硬件”

[^1]: J. Howard et al. A 48-core IA-32 message-passing processor with DVFS in 45nm CMOS. In Proc. ISSCC, Feb. 2010.