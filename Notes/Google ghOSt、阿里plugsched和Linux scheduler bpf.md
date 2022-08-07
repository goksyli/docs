# Google ghOSt、阿里plugsched和Linux scheduler bpf

## 背景和問題

- 应用场景不同，最佳调度策略不同。 应用种类极大丰富，应用特征也是千变万化 （throughput-oriented workloads, 𝜇s-scale latency critical workloads, soft real-time, and energy efficiency requirements），使得调度策略的优化比较复杂，不存在“一劳永逸”的策略。因此，允许用户定制调度器满足不同的场景是必要的。

- 调度器迭代慢。 Linux 内核经过很多年的更新迭代，代码变得越来越繁重。调度器是内核最核心的子系统之一，它的结构复杂，与其它子系统紧密耦合，这使得开发和调试变得越发困难。此外，Linux 很少增加新的调度类，尤其是不太可能接受非通用或场景针对型的调度器，上游社区在调度领域发展缓慢。

- 内核升级困难。调度器内嵌 （built-in）在内核中，上线调度器的优化或新特性需要升级内核版本。内核发布周期通常是数月之久，这将导致新的调度器无法及时应用在生产系统中。再者，要在集群范围升级新内核，涉及业务迁移和停机升级，对业务方来说代价昂贵。

- 无法升级子系统。kpatch 和 livepatch 是函数粒度的热升级方案，可修改能力较弱，不能实现复杂的逻辑改动；eBPF 技术在内核网络中广泛应用，但现在调度器还不支持 ebpf hook，将来即使支持，也只是实现局部策略的灵活修改，可修改能力同样较弱。

## plugsched

- Plugsched 是 Linux 内核调度器子系统热升级的 SDK，它可以实现在不重启系统、应用的情况下动态替换调度器子系统，毫秒级 downtime。Plugsched 可以对生产环境中的内核调度特性动态地进行增、删、改，以满足不同场景或应用的需求，且支持回滚。

- Plugsched 能将调度器子系统从内核中提取出来，以模块的形式对内核调度器进行热升级。通过对调度器模块的修改，能够针对不同业务定制化调度器，而且使用模块能够更敏捷的开发新特性和优化点，并且可以在不中断业务的情况下上线。

- 优势
  
  - 与内核发布解耦：调度器版本与内核版本解耦，不同业务可以使用不同调度策略；建立持续运维能力，加速调度问题修复、策略优化落地；提升调度领域创新空间，加快调度器技术演进和迭代
  
  - 可修改能力强 ：可以实现复杂的调度特性和优化策略，能人之所不能
  
  - 维护简单：不修改内核代码，或少量修改内核代码，保持内核主线干净整洁；在内核代码 Tree 外独立维护非通用调度策略，采用 RPM 的形式发布和上线
  
  - 简单易用：容器化的 SDK 开发环境，一键生成 RPM，开发测试简洁高效
  
  - 向下兼容：支持老内核版本，使得存量业务也能及时享受新技术红利
  
  - 高效的性能：毫秒级 downtime，可忽略的 overhead。

![](Google%20ghOSt、阿里plugsched和Linux%20scheduler%20bpf.assets/2022-08-07-21-59-13-image.png)

### Plugsched实现

- 依赖gcc-python-plugin的一个fork版本，追踪内核调度子系统的call graph，暂时还没有合入upstream。

- [gcc-python-plugin fork版本](https://github.com/ampresent/gcc-python-plugin.git)

- 机制
  
  - 内核开发人员根据自己的经验定义调度子系统涉及的文件、函数和变量，生成配置文件。
  
  - 根据这些和其他子系统的交互区分public和private接口，只有一些关键变量需要rebuild。
  
  - 通过gcc的plugin的机制，在编译内核的时候根据配置把函数和变量导出。
  
  - 修改导出的变量和函数，比如添加调度子系统的feature。
  
  - 使用跳转机制，实际是修改原函数的一个指令为jmp（opcode 0xe9)+4byte的相对地址，一共5byte的数据。相对地址就是新函数和旧函数的相对偏移。
  
  - 使用内核stop machine机制（kpatch等技术也在使用）[kernel_debug_notes/stop_machine at master · w-simon/kernel_debug_notes · GitHub](https://github.com/w-simon/kernel_debug_notes/blob/master/stop_machine)

## 与相关工作比较

- Q：Plugsched与sched eBPF有什么区别？

- A：目前，上游社区还不支持调度器eBPF的hook点，即便是支持了（其实有patch[Scheduler BPF [LWN.net]](https://lwn.net/Articles/869433/)，ghOSt也有对于eBPF的增强[eBPF in CPU Scheduler (lpc.events)](https://lpc.events/event/11/contributions/954/attachments/776/1463/eBPF%20in%20CPU%20Scheduler.pdf)），也只能支持局部策略的修改，可修改能力有限，而且eBPF不能实现很复杂的修改，它的检查机制很严格。

- Q：Google的ghOSt，似乎是和plugsched做类似工作？区别？

- A：ghOSt与plugsched面向的场景不同，性能也不同。ghOSt有两种工作模式，local模式开销较大，每次调度要多经历一次上下文切换，即切入切出用户态调度器软件，所以只能在一些延迟要求不高的场景用。而global模式过度依赖IPI，IPI的开销会导致调度不及时，增加延迟。再者，ghOSt针对无内核开发经验的用户态软件开发者，容错性比较高，但是性能相对较差，只能用于部分场景。plugsched依旧针对内核开发者，要求开发者有与传统内核同样的开发经验，但是为开发者降低了开发、测试、上线和回滚的难度，性能好，适用于绝大部分场景。

## ghOSt 用户态内核调度

- 本文来自谷歌，是继Snap<u>用户态</u>协议栈后的<u>用户态</u>调度——ghOSt。ghOSt用于满足谷歌数据中心复杂和平台快速演化的需求。

- ghOSt提供了通用的调度策略并托管给userspace进程。ghOSt提供状态封装、通信和action机制，允许在用户态代理中表达复杂策略。程序员可以使用任何语言来开发和优化策略。

- ghost系统的目标：
  
  1. 调度策略应当实现和测试容易
  2. 调度需要有效、广泛
  3. 突破per-CPU调度模型的限制
  4. 支持同时多个调度策略
  5. 升级不需要重启host和错误隔离

![](Google%20ghOSt、阿里plugsched和Linux%20scheduler%20bpf.assets/2022-08-07-23-06-54-image.png)

整个系统的调度架构如下图所示，ghost可以设置起多个`enclave`，每个enclave中可以运行不同的调度策略。ghost内核模块会将调度的决定逻辑托管给运行在每个物理CPU上的ghost agent。

*![](Google%20ghOSt、阿里plugsched和Linux%20scheduler%20bpf.assets/2022-08-07-23-10-24-image.png)*

- **与谷歌内部的MicroQuanto对比**：google把ghost和内部的kernel scheduler MicroQuanta进行对比，这个MicroQuanta是一个软实时的kernel scheduler用来管理snap框架，这个snap框架类似dpdk，需要维护一些polling线程来和网卡交互或是实现一些定制化的网络和安全协议。snap会根据网络负载来新增或减少polling的线程。

![](Google%20ghOSt、阿里plugsched和Linux%20scheduler%20bpf.assets/2022-08-07-23-13-29-image.png)测试结果如上图所示,在处理64B的网络包时ghost的时延会比MicroQuanta差10%是因为小数据包的处理时间很短所以ghost进出userspace的开销就会变得明显。处理64KB大小的网络包时，ghost的时延会更好则是因为ghost能跨cpu进行负载均衡。


