# 1. What is GPUDirect Storage ?
Before talking about GDS, we should study how DMA works between devices. DMA is a copy between host memory and device's memory. 
But DMA works not only between host memory and device's memory but also between two devices without host memory. 
DMA Engine needs to know the physical address of target prior to the copy even between host memory and device's memory or between devices. 
See also followings:

https://www.eideticom.com/media-news/blog/33-p2pdma-in-linux-kernel-4-20-rc1-is-here.html

The case of DMA between host memory and PCI device:
```

```
The case of DMA between GPU and NVMe device (p2pdma):


We can use BARs on a PCI device for the target of DMA.
```

```


