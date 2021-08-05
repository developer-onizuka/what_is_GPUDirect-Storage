# 1. What is GPUDirect Storage ?
Before talking about GDS, we should study how DMA works between devices. DMA is a copy between host memory and device's memory. 
But DMA works not only between host memory's but between devices. DMA Engine needs to know the physical address of target prior to the copy between devices. 
```
```
The case of DMA between host memory and PCI device:
```

```
The case of DMA between GPU and NVMe device:
We can use BARs on a PCI device for the target of DMA.
```

```


