Direct Memory Access (DMA) is a feature in computer systems that allows certain hardware components (such as disk drives, network cards, or graphics cards) to directly read from or write to the system memory without involving the CPU for each transfer. This capability is crucial for high-performance I/O operations because it frees up the CPU to perform other tasks while the data transfer occurs in the background.

### How DMA Works:
1. **Traditional Data Transfer**: In a typical I/O operation without DMA, the CPU is involved in every step of data transfer. For example, if data needs to be transferred from a disk to memory, the CPU would first read the data from the disk into its registers and then write it into the memory. This process is CPU-intensive and inefficient, especially for large data transfers.

2. **DMA-Enabled Transfer**:
   - **DMA Controller**: The system includes a DMA controller, a dedicated hardware component that manages DMA operations. When a peripheral device (e.g., a disk drive) needs to transfer data to or from memory, it requests the DMA controller to handle the transfer.
   - **Initiating Transfer**: The CPU initializes the DMA operation by programming the DMA controller with the source address, destination address, and the size of the data to be transferred.
   - **Data Transfer**: The DMA controller takes over and performs the data transfer directly between the peripheral and the system memory, bypassing the CPU. During this process, the CPU can continue executing other tasks.
   - **Completion**: Once the data transfer is complete, the DMA controller sends an interrupt to the CPU, signaling the completion of the operation. The CPU can then process the transferred data if needed.

### Types of DMA:
- **Burst Mode**: The DMA controller transfers the entire block of data in one continuous operation, temporarily taking control of the system bus from the CPU.
- **Cycle Stealing Mode**: The DMA controller takes control of the system bus for short periods, transferring small amounts of data (usually one word) per cycle, allowing the CPU to continue operating but at a reduced speed.
- **Transparent Mode**: The DMA controller only transfers data when the CPU is not using the system bus, ensuring no performance impact on the CPU.

### Benefits of DMA:
- **Reduced CPU Overhead**: The CPU is freed from managing every byte of data during large data transfers, significantly reducing CPU overhead.
- **Increased Data Transfer Speed**: DMA can transfer data much faster than CPU-driven transfers, especially in systems with high-bandwidth peripherals.
- **Improved System Performance**: By offloading data transfer tasks to the DMA controller, the CPU can focus on other computational tasks, leading to overall better system performance.

### Use Cases of DMA:
- **Disk I/O**: DMA is commonly used for transferring data between hard drives and memory, allowing for fast data read/write operations.
- **Network I/O**: Network cards use DMA to efficiently transfer data packets to and from memory without involving the CPU in every packet transfer.
- **Graphics Processing**: Graphics cards utilize DMA to transfer large amounts of graphical data to and from memory, enabling high-performance rendering in video games and other graphics-intensive applications.

In summary, DMA operations are vital for efficient data transfer in modern computer systems, enabling faster I/O operations and reducing CPU load.

***Q1: Why DMA Controller require system bus for its operations?***

***Ans:*** the DMA controller requires the system bus to directly access system memory and peripheral devices, enabling efficient data transfers without involving the CPU. The system bus is the backbone of communication within the computer, making it essential for the DMA controller's operations.

During a DMA operation, the CPU might be limited in its ability to access the system bus, but it can still perform various tasks that do not require immediate memory access. The overall efficiency of this process is part of why DMA is beneficial—it allows data to be moved independently of the CPU, freeing the CPU to perform other operations concurrently, albeit with some limitations.


### NUMA
In a NUMA (Non-Uniform Memory Access) system, memory is logically divided into banks, also referred to as "nodes." Each node typically corresponds to a memory region that is physically close to a specific processor or group of processors. The key concept in NUMA is that memory access times are not uniform across the system—accessing memory within the same node (local memory) is faster than accessing memory located in another node (remote memory).

With large scale machines, memory may be arranged into banks that incur a different cost to access depending on the “distance” from the processor. For example, there might be a bank of memory assigned to each CPU or a bank of memory very suitable for DMA near device cards.

---

# Describing Physical Memory
Each bank is called a _node_ and the concept is represented under Linux by a struct pglist_data even if the architecture is UMA. This struct is always referenced to by it's typedef pg_data_t. Every node in the system is kept on a NULL terminated list called pgdat_list and each node is linked to the next with the field pg_data_t→node_next. For UMA architectures like PC desktops, only one static pg_data_t structure called contig_page_data is used.


https://www.kernel.org/doc/gorman/html/understand/understand005.html

---

# SLUB Allocator

https://kernel.org/doc/gorman/html/understand/understand011.html
https://blogs.oracle.com/linux/post/linux-slub-allocator-internals-and-debugging-1

![Cache Object Structure](/images/Pasted_image_20240815141754.png)

![Cache Tree](/images/Pasted_image_20240815141928.png)

- In the SLUB allocator, a `kmem_cache` object represents a slab cache with per-CPU data managed by `kmem_cache_cpu`. Each slab is a `slab` object, and in kernels before 5.17, slab info is stored in a page object's union. `kmem_cache_node` represents a memory node.

- Each slab cache in SLUB has a per-CPU active slab, a per-CPU partial slab list, and a per-node partial slab list, with slabs connected via `slab.next` or `slab.slab_list`. Objects are always allocated from the per-CPU active slab, and when it's full, the next slab becomes active.

![Per Slab Freelist](/images/Pasted_image_20240815144722.png)
- The per-CPU active slab has a lockless freelist(`kmem_cache_cpu.slab.freelist`) and a regular freelist(`kmem_cache_cpu.freelist`). Object allocation is attempted from the lockless freelist first, which allows lock-free operations on architectures supporting `cmpxchg`, but other operations like manipulating the regular freelist still require locking.

![All Object Lists](/images/Pasted_image_20240815145051.png)

- Both `kmem_cache_cpu.freelist` and `kmem_cache_cpu.slab.freelist` point to objects on the active slab, but they are distinct lists, each containing objects from the same slab.

- Slabs can be full, partially full, or empty. Full slabs are not listed in partial slab lists and can be destroyed or reclaimed if empty. When an object from a full slab is freed, the slab is moved to the appropriate partial list, eliminating the need to maintain a separate list of full slabs (except for debugging purposes).

- A slab can consist of one or more pages, regardless of object size, with the number of pages determined by `kmem_cache.oo` (the number of objects per slab).

- `kmem_cache_cpu.slab.freelist` points to the first free object on the slab in case of single page slab. For slabs consisting of a compound page, the freelist of the head page points to the first free object, which can be located on the head page or any of the tail pages.

- Allocation occurs from the active slab, with the per-CPU freelist and the active slab's freelist pointing to objects on this slab. A slab cache is initially created with a fixed number of objects and can be extended if needed.

- There are two allocation paths: the **fastpath** for quick access via the per-CPU freelist, and the **slowpath** for more complex operations when the fastpath cannot fulfill the request.
- **Fastpath Allocation**: Occurs when the per-CPU lockless freelist (`kmem_cache.slab.freelist`) contains free objects. This path is simple, with no locking or irq/preemption disabling needed. The first object in the freelist is allocated, and the freelist head updates to the next object. If the freelist is empty, it becomes `NULL` until more objects are added. Fastpath is bypassed if the CPU doesn’t support `cmpxchg` for two words or if SLUB debugging is enabled.

-  When a slab becomes a per-CPU active slab, its freelist is transferred to the per-CPU lockless freelist, leaving the slab's freelist `NULL`. Objects allocated from this lockless freelist, if freed by the same CPU, return to the lockless freelist, but if freed by another CPU, they return to the slab's freelist. This process can lead to a scenario where a CPU's lockless freelist is empty, but the active slab's freelist still has free objects.

- When a CPU's active slab has no free objects, but the per-CPU partial slab list contains slabs with free objects, the first slab in this list becomes the new active slab. Its freelist is transferred to the CPU's lockless freelist, from which objects are allocated. This path, marked as "SLOWPATH 2," involves additional overhead compared to previous paths, as it requires updating the active slab and the per-CPU partial slab list, along with disabling preemption and acquiring `kmem_cache_cpu.lock`.

- If the per-CPU slabs (active and partial) have no free objects, allocation attempts are made from the per-node partial slab list. Slabs enter the per-node partial slab list when a full slab becomes empty or partial and cannot be placed into the per-CPU partial slab list, either because the per-CPU list isn't supported or is already full. This mechanism allows the per-node partial slab list to accumulate slabs for future allocations.