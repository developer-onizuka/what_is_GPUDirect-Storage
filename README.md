# This is my own understandings about GPUDirect Storage based on my general knowledges of computer sciences. So please use this just for your information.



# 1. What is GPUDirect Storage ?
GDS enables a DMA engine near storage (NVMe or NIC) to push (or pull) data directly into (and out of) GPU memory. But, before talking about GDS, we should study how DMA works between devices. DMA is a copy between host memory and device's memory. 
(https://github.com/developer-onizuka/what_is_DMA/blob/main/README.md)

But, DMA works not only between host memory and device's memory but also between two devices without host memory. 
DMA Engine needs to know the physical address of target prior to the copy even between host memory and device's memory or between devices. 
(https://linuxreviews.org/Peer_To_Peer_DMA)

# 2. BAR Space
One of BAR Spaces is used by DMA between host memory and device.<br><br>
As you can see below, /proc/iomem says not only the physical address originated from DIMM but the physical address from PCI device or BIOS. When kernel is booting, the device provides its "BAR" which the kernel then maps into main memory. The memory mappings is exposed to userspace via /proc/iomem.
```
$ sudo cat /proc/iomem 
00000000-00000fff : Reserved
00001000-0009fbff : System RAM
0009fc00-0009ffff : Reserved
000a0000-000bffff : PCI Bus 0000:00
000c0000-000c99ff : Video ROM
000ca000-000cadff : Adapter ROM
000cb000-000cd3ff : Adapter ROM
000f0000-000fffff : Reserved
  000f0000-000fffff : System ROM
00100000-7ffcffff : System RAM
7ffd0000-7fffffff : Reserved
80000000-afffffff : PCI Bus 0000:00
b0000000-bfffffff : PCI MMCONFIG 0000 [bus 00-ff]
  b0000000-bfffffff : Reserved
c0000000-febfffff : PCI Bus 0000:00
  d0000000-efffffff : PCI Bus 0000:04
    d0000000-dfffffff : 0000:04:00.0
    e0000000-e1ffffff : 0000:04:00.0
  f0000000-f07fffff : 0000:00:01.0

... snip ...

  fb000000-fcffffff : PCI Bus 0000:04
    fb000000-fbffffff : 0000:04:00.0
      fb000000-fbffffff : nvidia
    fc000000-fc07ffff : 0000:04:00.0
  fd000000-fd1fffff : PCI Bus 0000:09
  fd200000-fd3fffff : PCI Bus 0000:08
  fd400000-fd5fffff : PCI Bus 0000:07
  fd600000-fd7fffff : PCI Bus 0000:06
    fd600000-fd603fff : 0000:06:00.0
      fd600000-fd603fff : nvme
    
... snip ...

feffc000-feffffff : Reserved
fffc0000-ffffffff : Reserved
100000000-27fffffff : System RAM
  135000000-135e00eb0 : Kernel code
  135e00eb1-13685763f : Kernel data
  136b20000-136ffffff : Kernel bss
280000000-a7fffffff : PCI Bus 0000:00
```

# 3. The case of DMA between host memory and PCI device (traditional DMA):
First of all, a Device Driver will inform the DMA engine about the target Physical Address to copy from BAR to host memory through the Virutal Address and will queue it on the FIFO of the descriptor. DMA engine should actually fetch these instructions created by the Device Driver from host memory in advance. So, it is very important to let the DMA Engine know the Virtual Address before DMA operation.<br>
After DMA between host and device, Device's DMA Engine interrupts to CPU so that CPU can start copying this DMAed data (it's still in kernel space) to the user space by CPU load, which is not DMA. Next, the space in kernel is released for the next DMA operation, which we know it as flow control of DMA.
```
          Physical Memory
          +----------+
          |          |
          |          |
          +----------+ 0xdfffffff
          |XXXXXXXXXX|
   +----- |XXXXXXXXXX|
   |      +----------+ 0xd0000000 (GPU BAR#1)
   |      |          |                                          Kernel Space (Virtual Address)
 Copy     |          |                                          +----------+
 (DMA)    |          |                                          |          |
   |      +----------+                                          |          | 
   +----> |XXXXXXXXXX|                                          |XXXXXXXXXX|
   +----- |XXXXXXXXXX| <================Mapping===============> |XXXXXXXXXX| -----+
   |      +----------+ Host Memory for DMA operation            +----------+      |
 Copy     |          | (Physical Address)                                        Copy
 (CPU)    |          |                                          +----------+     (CPU)
   |      +----------+                                          |          |      |
   +----> |XXXXXXXXXX|                                          |XXXXXXXXXX| <----+
          |XXXXXXXXXX| <================Mapping===============> |XXXXXXXXXX|
          +----------+ User Space (Physical Address)            +----------+
          |          |                                          User Space (Virtual Address)
          |          |
          +----------+ 0x00000000
```
# 4. The case of DMA between GPU and NVMe device (P2P DMA):
We can use BAR space on a PCI device as a target instead of host memory for DMA operation. According the P2P DMA, One device needs to present a memory BAR, and the other one accesses it. 
In GDS, GPU provides its BAR and the NVMe's DMA Engine accesses the physical address which the kernel mapped into thru the GPU's BAR. The P2P DMA is done thru the copy between GPU BAR space and NVMe Bar space (only 16KB, any legitimate NVMe device must have) by NVMe's DMA Engine. (Not by GPU's DMA Engine) 


After P2P DMA, NVMe DMA Engine interrupts to GPU so that GPU can start copying this DMAed data in BAR space to GPU Memory, so that GPU processor can do his jobs. I believe GPU's logic is also interruted by NVMe's DMA Engine after DMA. It is same as CPU was interruted while traditional DMA was finished by DMA Engine. 
It is same as vice versa. But, please note that any host memories are not at all involved with GDS operations.


I guess Controller Memory Buffer (CMB) which NVMe controller exposes could contribute performance improvement on GDS which utilizes P2P DMA. But GDS is actually using the BAR space on GPU card for P2P DMA.
(https://www.eideticom.com/media-news/blog/33-p2pdma-in-linux-kernel-4-20-rc1-is-here.html)
```
         Physical Memory
          +----------+                                          The file in NVMe Storage
          |          |                                          (mount -t ext4 -o data=ordered /dev/nvme0n1 /mnt)
          +----------+ 0xfd603fff                Fetching       +----------+
   +----- |XXXXXXXXXX| 16KB  <--------------------------------- |XXXXXXXXXX| /mnt/test.txt
   |      +----------+ 0xfd600000 (NVMe's BAR)                  |          |
  Copy*   |          |                                          +----------+
(P2P DMA) |          |                                           
   |      +----------+ 0xdfffffff                               GPU Memory associated with CudaMalloc()
   |      |          |     Copy from BAR space to GPU Memory    +----------+
   +----> |XXXXXXXXXX| ---------------------------------------> |XXXXXXXXXX| ---> Cunsumed by GPU core
          +----------+ 0xd0000000 (GPU's BAR)                   |          |
          |          |                                          +----------+
          |          |
          |          |                                          Kernel Space (Virtual Address)
          |          |                                          +----------+
          |          |                                          |          |
          |          |                                          |          |
          |          |                                          +----------+
          |          |                                                      
          |          |                                          +----------+
          |          |                                          |          |
          |          |                                          |          |
          |          |                                          |          |
          |          |                                          +----------+
          |          |                                          User Space (Virtual Address)
          |          |
          +----------+ 0x00000000

   * : I believe the NVMe's DMA Engine do this copy. Not by GPU's DMA Engine.

```
# 5. How to know the BAR Space ?
But how NVMe DMA engine knows the BAR of GPU ??? 


Linux kernel has some storage stacks like below:
```
 User Application
                              User Space
 ----------------------------
                              Kernel Space
 Virtual File System (VFS)
 
 Filesystem Driver

 Block IO Driver
 
 NVMe Driver
 
 ----------------------------
                              Hardware
 NVMe Hardware (DMA Engine)
 
```
Linux is not enabled to handle GPU Virtual Addresses needed for DMA operation, which is in user space (not in kernel space) and it finally introduces page fault because of no mapping between GPU's BAR space and Virutal Address. This means NVMe Driver can not let the NVMe's DMA Engine know the GPU's BAR which is target address of DMA. Again, Existing operating systems attempting to program DMA engines cannot process GPU virtual addresses without help. 

But I believe nvidia-fs Driver API already made solutions to do that on EXT4 filesystem and block IO drivers instead of the Linux kernel's modification. So you can do GDS today.

https://docs.nvidia.com/gpudirect-storage/overview-guide/index.html#software-arch
