

MESI

cache line size 64B

page size 4KB

伪共享  cacheline 补齐 技巧 

Disruptor

@Contended



可见性

memory barrier

volatile

指令重排

只要程序的最终结果等同于它在严格的顺序化环境下的结果 // 在一个线程内部来看，在外部线程不一定

if two variables are in the same cache line, and they are written to by different threads, then they present the same problems of write contention as if they were a single variable. This is a concept know as “false sharing”


DCL   double check lock