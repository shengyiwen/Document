### atomic操作

在编程过程中我们经常会使用到原子操作，这种操作既不像互斥那样耗时，又可以保证对变量操作的原子性，常见的原子操作有fetch_add、load、increment等

而对于atomic的实现最基础的解释就是:**原子操作是由底层硬件支持的一种特征**


    #include <atomic>
    
    int main() {
        std::atomic<int> a;
        a ++;
        return 0;
    }

然后进行编译，查看编译文件

    _ZNSt13__atomic_baseIiEppEi:
    .LFB362:
    	pushq	%rbp
    	.seh_pushreg	%rbp
    	movq	%rsp, %rbp
    	.seh_setframe	%rbp, 0
    	subq	$16, %rsp
    	.seh_stackalloc	16
    	.seh_endprologue
    	movq	%rcx, 16(%rbp)
    	movl	%edx, 24(%rbp)
    	movq	16(%rbp), %rax
    	movq	%rax, -8(%rbp)
    	movl	$1, -12(%rbp)
    	movl	$5, -16(%rbp)
    	movl	-12(%rbp), %edx
    	movq	-8(%rbp), %rax
    	lock xaddl	%edx, (%rax)
    	movl	%edx, %eax
    	nop
    	addq	$16, %rsp
    	popq	%rbp
    	ret
    	.seh_endproc
    	.ident	"GCC: (x86_64-posix-seh-rev0, Built by MinGW-W64 project) 8.1.0"

我们可以看到自增操作的时候，在xaddl的指令前多了一个lock前缀，而cpu对这个lock指令的支持就会所谓的底层硬件支持。增加了这个前缀后，就保证了load-add-store步骤的不可分割

### lock指令的实现

cpu在执行任务的时候，不是直接从内存里加载数据的，而是先将数据加载到L1和L2的Cache中，然后再从Cache中读取数据进行运算的。而现代计算机通常都是多核的，每一个内核都对应了一个独立的L1层缓存，多核之间的缓存数据同步是CPU框架设计的重要部分。MESI是比较常用的多核缓存同步方案。当我们在单线程里进行atomic操作的时候，不会涉及到多核之间数据同步的问题，但是我们在多线程多核的情况下，CPU如何保证lock特征呢。

在X86的cpu架构下，官方的lock实现有两种

- 锁bus，性能消耗大，在486处理器上用此方式实现
- 锁cache，在现代处理器上使用此方式，但是在无法锁定cache的时候，仍然回去锁bus


**Bus Lock:**

英特尔提供Lock信号，该信号在某些关键内存操作期间会自动断言，以锁定系统总线或等效链接。在断言该输出信号时，来自其他处理器或总线代理的控制总线的请求将被阻止。此度量标准度量总线周期的比率，在此期间，在总线上声明Lock信号，当由于不可缓存的内存，跨越两条缓存行的锁定操作以及来自不可缓存的页表的页面遍历而导致在锁定的内存访问时，将发出Lock信号

**Cache Lock:**

cache lock的实现比bus lock要复杂的多，这里涉及到内存cache同步，还有memory barriers、cache line和shared memory等概念


### Mutex的实现

Mutex的实现也是类似于此操作，也是通过这些原语进行的实现，通过对一块内存的开辟，通过记录其状态、当前线程等，然后通过原语如CAS，TEST AND SET原语等进行修改