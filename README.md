# This is my own understandings about GPUDirect Storage by my general knowledge of computer science. So please use this just for your informations.



# 1. What is GPUDirect Storage ?
Before talking about GDS, we should study how DMA works between devices. DMA is a copy between host memory and device's memory. 
(https://github.com/developer-onizuka/what_is_DMA/blob/main/README.md)

But DMA works not only between host memory and device's memory but also between two devices without host memory. 
DMA Engine needs to know the physical address of target prior to the copy even between host memory and device's memory or between devices. 
(https://linuxreviews.org/Peer_To_Peer_DMA)

# 2. The case of DMA between host memory and PCI device (traditional DMA):
After DMA between host and device, Device's DMA Engine interrupts to CPU so that CPU can start copying this DMAed data (it's still in kernel space) to the user space by CPU load, which is not DMA. Next, the space in kernel is released for the next DMA operation, which we know it as flow control of DMA.
```
               Pysical Memory
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
# 3. The case of DMA between GPU and NVMe device (P2P DMA):
We can use BAR space on a PCI device as a target of DMA. According the P2P DMA, One device needs to present a memory BAR, and the other one accesses it. 
In GDS, GPU provides its BAR and the NVMe Engine accesses the physical address which the kernel mapped into thru the GPU's BAR. The P2P DMA is done thru the copy between GPU BAR space and NVMe Bar space (only 16KB, any legitimate NVMe device must have) by NVMe's DMA Engine. (Not by GPU's DMA Engine) 
After P2P DMA, NVMe DMA Engine interrupts to GPU so that GPU can start copying this DMAed data in BAR space to GPU Memory, so that GPU processor can do his jobs. 
It is same as vice versa. But, please note that any host memories are not at all involved with GDS operations.


I guess Controller Memory Buffer (CMB) which NVMe controller exposes could contribute performance improvement on GDS which utilizes P2P DMA. But GDS is actually using the BAR space on GPU card for P2P DMA.
(https://www.eideticom.com/media-news/blog/33-p2pdma-in-linux-kernel-4-20-rc1-is-here.html)
```
              Pysical Memory
               +----------+                                          The file in NVMe Storage
               |          |                                          (mount -t ext4 -o data=ordered /dev/nvme0n1 /mnt)
               +----------+ 0xfd603fff                Fetching       +----------+
	+----- |XXXXXXXXXX| 16KB  <--------------------------------- |XXXXXXXXXX| /mnt/test.txt
        |      +----------+ 0xfd600000 (NVMe's BAR)                  |          |
       Copy*   |          |                                          +----------+
     (P2P DMA) |          |                                           
        |      +----------+ 0xdfffffff                               GPU Memory (My Quadro P1000 has 4GB memory)
        |      |          |                           Copying        +----------+
        +----> |XXXXXXXXXX| ---------------------------------------> |XXXXXXXXXX| -----> go to GPU processor
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
# 4. How to know the BAR Space ?
But how NVMe engine knows the BAR of GPU ??? 


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
Linux is not enabled to handle GPU Virtual Addresses needed for DMA and it finally introduce page fault because of no mapping between GPU's BAR space and Virutal Address. This means NVMe Driver can not let the NVMe's DMA Engine know the GPU's BAR which is target address of DMA. To improve this, NVIDIA is actively working with the community on upstream first to enable Linux to handle GPU VAs for DMA. See also followings:

https://on-demand.gputechconf.com/supercomputing/2019/pdf/sc1922-gpudirect-storage-transfer-data-directly-to-gpu-memory-alleviating-io-bottlenecks.pdf

But I believe nvidia-fs Driver API made solutions to do that on EXT4 filesystem and block IO drivers. So you can do GDS today.
