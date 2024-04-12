# type1 sysHyper学习

## 启动命令

```
bootm 0x40400000 - 0x40000000
```

## 启动流程

1. uboot先启动第一个核，执行hvisor.bin，之后通过wakeup_secondary_cpus唤醒其他核。

## 疑问