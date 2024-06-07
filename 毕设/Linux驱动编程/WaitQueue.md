# WaitQueue

WaitQueue是内核的一个队列，用于阻塞Kernel thread或者进程。对于阻塞进程，进程可以通过ioctl陷入内核模块，然后在内核模块中执行wait_event，这样进程就被阻塞了，直到条件满足。