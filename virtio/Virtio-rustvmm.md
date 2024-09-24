# vm-virtio调研

项目初衷：通过复用该仓库的代码，方便hypervisor的实现。

包含：virtio-blk、virtio-console，但还没有支持virtio-net。里面包含了定义virtio-queue、virtio-device必要的数据结构或特征，但还需要自己实现一部分代码。virtio-blk和virtio-console只是实现了从virtio-queue中读取请求，并将请求交给后端执行，后端执行后将结果写入virtio-queue的过程。后端（与真实设备交互的那部分代码）的实现还需要自己完成。

另外要直接复用该项目，还需要适配vm-virtio依赖的一些库，比如mem、sys_util。

结论：感觉暂时还用不上。

virtio-bindings：通过bindgen自动生成的，将linux中有关virtio各设备的头文件中的数据结构，都转换成了rust形式。

关键问题是咱们怎么用。。。

最好能直接复用，动咱们自己的接口也行，否则之后更新咱们跟不上。