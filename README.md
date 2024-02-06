# DPDK

本项目是对 DPDK 的源码的分析笔记。基于 dpdk 22.11.4 版本。

## 内存布局

DPDK 初始化之后内存布局如下：

![](assets/dpdk-memory-layout.svg)

* DPDK 的共享内存默认起始地址为 `0x100000000`，通过 EAL 参数 `--base-virtaddr` 可以设置起始地址。`primary` 和 `secondary` 进程使用相同的起始地址，因此它们可以跨进程使用共享内存中的虚拟地址。
* `mem_cfg` 是文件 `/run/dpdk/rte/config` 的内存映射，它保存了内存的配置信息
* `rte_fbarray` 是一个对象池管理器，它由对象数组和 `mask` 组成
  * 对象可以是任意类型，在初始化时设置好对象的大小和数量
  * `mask` 是一个 BitMap 类型的数据，纪录着哪些位置上的对象已经分配
  * 在进程的私有内存区域中，`mem_area_tailq` 链表保存着当前在当前进程创建的 `rte_fbarray`
* `mem_cfg.memzones` 保存着 `rte_memzone_reserve` 分配的内存区域的描述信息，最大支持 `RTE_MAX_MEMZONE` 个内存区域
* `memsegs[]` 数组保存着各 NUMA 结点的大页信息
  * `memsegs[]` 实际是一个三维数组：大页类型数量、NUMA 数量、SegList 数量
    * 大页类型数量：该维度的长度是系统支持的大页类型的数量。例如，系统支持 256K，2M，1G，则维度是 3
    * NUMA 数量：系统上 NUMA 结点的数量。NUMA 是一种内存访问的硬件架构，常见于服务器上。在 NUMA 架构下，CPU 的内存分为本地和远程两种。本地内存是与 NUMA 直接相连接的内存，而远程内存是与其它 NUMA 相连接的内存。访问本地内存的速通常较快。
    * SegList 数量：每个分段设计了最大内存上限，因此分成多个 SegList
  * `memsegs[i]` 中保存着大页内存段列表的开始位置和一个纪录着大页信息的 `rte_fbarray` 对象
    * `memseg_arr` 是一个由若干个 `rte_memseg` 对象组成的 `rte_fbarray` 对象池，每个 `rte_memseg` 对象描述一个大页的信息
    * 在 `memseg_arr` 所指向的 `rte_fbarray` 内存之后，向上对齐到大页大小的地址是若干个大页内存段的开始地址
