### mmap详解

### 什么是mmap

mmap是一种内存映射文件的方法，即将一个文件或者其他对象映射到进程的地址空间，实现文件磁盘地址和进程虚拟地址空间中一段虚拟地址的映射关系。文件被映射到多页上，如果文件的大小不是所有页的大小之和，最后一个页不会被使用清零。实现这样的映射关系后，进程就可以采用指针的方式读写操作这一段内存，而系统会自动会写脏页面到对应的文件磁盘上，即完成了对文件的操作而不必调用read、write等系统函数。相反，内核空间对这段区域的修改也直接反应用户空间，从而可以实现不同进程间的文件共享。

![mmap地址映射](../Images/mmap地址映射.png)

### 内存映射原理

- 进程启动映射过程，并在虚拟地址空间中为映射创建虚拟映射区域

    1. 进程在用户空间调用mmap函数，原型：void *mmap(void *start, size_t length, int prot, int flags, int fd, off_t offset)
    
    2. 在当前进程的虚拟地址空间中，寻找一段空闲的满足要求的连续的虚拟地址
    
    3. 为此虚拟区分配一个vm_area_struct结构，接着对这个结构的各个区域进行了初始化
        
    4. 为新建的虚拟区结构vm_area_struct插入进程虚拟地址区域链表或树中
    

- 调用内核空间的系统调用函数mmap(不同于用户函数)，实现文件物理地址和进程虚拟地址的一一映射

    1. 为映射分配了新的虚拟地址区域后，通过待映射的文件指针，心思文件描述符表中找到对应的文件描述符，通过文件描述符，连接到内核"已打开文件集"中该文件结构体(struct file)，每个文件结构体维护着和这个已打开文件相关各项信息
    
    2. 通过该文件的文件结构体，链接到file_operations模块，调用内核mmap，其原型为: int map(struct file *filp, struct vm_area_struct *vma)，不同于用户函数
    
    3. 内核mmap函数通过虚拟文件系统inode模块定位到文件磁盘物理地址
    
    4. 通过remap_pfn_range函数建立页表，即实现了文件地址和函数地址区域的映射关系。此时，这篇虚拟地址并没有任何数据关联到主存中
    
- 进程发起对这片映射空间的访问，引发缺页异常，实现文件内容到物理内存的拷贝

    1. 前两个阶段仅在创建虚拟区间并完成地址映射，但是并没有将任何文件数据的拷贝至主存，真正的文件读取是当进程发起读或写操作时
    
    2. 进程的读或写操作访问虚拟地址空间的这一映射地址，通过查询页表，发现这一段地址并不在物理页面上，因为目前只是建立了地址映射，因此发生缺页异常
    
    3. 缺页异常进行一系列判断，确定无法操作后，内核发起请求调页过程
    
    4. 调页过程先在交换缓存空间(swap cache)中寻找需要访问的内存页，如果没有则调用nopage函数把所缺的页从磁盘装入到主存中
    
    5. 之后进程即可对这片主存进行读或写的操作，如果写操作改变了其内容，一定时间后系统会主动回写脏页到对应磁盘地址，也即完成了写入到文件的过程
    
### mmap的优点

- 对文件的读取操作跨过了页缓存，减少了数据的拷贝次数，用内存读写取代了IO读写，提高了文件读取效率

- 实现了用户空间和内核空间的高效交互方式，两空间的各自修改操作可以直接反应在映射区域内，从而被对方空间及时捕捉

- 提供进程间共享内存及相互通信的方式，不管是父子进程还是没有关系的进程，都可以将自己用户空间映射到同一文件或者匿名映射到同一片区域。从而通过各自对映射区域的修改，达到进程间通信和进程间共享的目的

- 可用于实现高效的大规模数据传输，内存空间不足，是制约大数据操作的一个方面，解决方案往往是借助硬盘空间协助操作，补充内存的不足。但是进一步会造成大量的文件IO操作，极大影响效率，这个问题可以通过mmap映射很好的解决，换句话说，但凡是需要用磁盘替代内存的时候，mmap都可以发挥其效

### mmap缺点

- 文件如果很小，是小于4096个字节的，比如10字节，由于内存的最小粒度是页，而进程虚拟地址空间和内存的映射也是以页为单位。虽然被映射的文件只有10字节，但是对应到进程虚拟地址区域的大小需要满足整页大小，因此mmap函数执行后，实际映射到虚拟内存区域的是4096个字节，11~4096的字节部分用零填充。因此如果连续mmap小文件，会浪费内存空间

- 对变长文件不适合，文件无法完成拓展，因为mmap到内存的时候，你所能够操作的范围就确定了

- 如果更新文件的操作很多，会触发大量的脏页回写及由此引发的随机IO上。所以在随机写很多的情况下，mmap方式在效率上不一定会比带缓冲区的一般写快

### 函数原型

- 函数
  
```c
    void *mmap(void *start, size_t length, int prot, int flags, int fd, off_t offset)
```

- 参数

    - start: 映射区的开始地址
      
    - length: 映射区的长度
    
    - prot: 期望的内存保护标志，不能与文件的打开模式冲突，可以通过or运算合理组合
        - PROT_EXEC: 页内容可以被执行
        - PROT_READ: 页内容可以被读取
        - PROT_WRITE: 页内容可以被写入
        - PROT_NONE: 页不可被访问
    
    - flags: 指定映射对象的类型，映射选项和映射页是否可以共享，它的值可以是一个或者多个
        - MAP_FIXED 使用指定的映射起始地址，如果由start和len参数指定的内存区重叠于现存的映射空间，重叠部分将会被丢弃。如果指定的起始地址不可用，操作将会失败。并且起始地址必须落在页的边界上
        - MAP_SHARED 与其它所有映射这个对象的进程共享映射空间。对共享区的写入，相当于输出到文件。直到msync()或者munmap()被调用，文件实际上不会被更新
        - MAP_PRIVATE 建立一个写入时拷贝的私有映射。内存区域的写入不会影响到原文件。这个标志和以上标志是互斥的，只能使用其中一个
        - MAP_DENYWRITE 这个标志被忽略
        - MAP_EXECUTABLE 同上
        - MAP_NORESERVE 不要为这个映射保留交换空间。当交换空间被保留，对映射区修改的可能会得到保证。当交换空间不被保留，同时内存不足，对映射区的修改会引起段违例信号
        - MAP_LOCKED 锁定映射区的页面，从而防止页面被交换出内存
        - MAP_GROWSDOWN 用于堆栈，告诉内核VM系统，映射区可以向下扩展
        - MAP_ANONYMOUS 匿名映射，映射区不与任何文件关联
        - MAP_ANON MAP_ANONYMOUS的别称，不再被使用
        - MAP_FILE 兼容标志，被忽略
        - MAP_32BIT 将映射区放在进程地址空间的低2GB，MAP_FIXED指定时会被忽略。当前这个标志只在x86-64平台上得到支持
        - MAP_POPULATE 为文件映射通过预读的方式准备好页表。随后对映射区的访问不会被页违例阻塞
        - MAP_NONBLOCK 仅和MAP_POPULATE一起使用时才有意义。不执行预读，只为已存在于内存中的页面建立页表入口
    
    - fd，有效的文件描述词，如果MAP_ANONYMOUS被设定，为了兼容问题，其值为-1
    
    - offset: 被映射对象内容的起点

- 返回说明

成功执行时，mmap()返回被映射区的指针。失败时，mmap()返回MAP_FAILED

    1.EACCES：访问出错
    2.EAGAIN：文件已被锁定，或者太多的内存已被锁定
    3.EBADF：fd不是有效的文件描述词
    4.EINVAL：一个或者多个参数无效
    5.ENFILE：已达到系统对打开文件的限制
    6.ENODEV：指定文件所在的文件系统不支持内存映射
    7.ENOMEM：内存不足，或者进程已超出最大内存映射数量
    8.EPERM：权能不足，操作不允许
    9.ETXTBSY：已写的方式打开文件，同时指定MAP_DENYWRITE标志
    10.SIGSEGV：试着向只读区写入
    11.SIGBUS：试着访问不属于进程的内存区


- 相关函数

```c
int munmap( void * addr, size_t len) 
```

该函数在进程地址空间中解除一个映射关系，addr是调用mmap()时返回的地址，len是映射区的大小。成功执行时mumap返回0，失败时返回-1，error返回标示和mmap一致。关系解除后，对原先地址的访问会导致错误


```c
int msync ( void * addr , size_t len, int flags)
```

一般来说，进程的映射空间的对共享内容的修改并不直接写回到磁盘文件中，往往在调用munmap()后才执行该操作，可以通过调用msync函数，实现磁盘上文件内容和共享文件内容区的一致

### 使用细节

- mmap的使用关键点，mmap映射的区域大小必须物理页(page_size)大小的整数倍(32位系统中通常是4KB)，原因是内存的最小颗粒是页，而进程虚拟地址空间和内存的映射以页为单位，为了匹配内存操作，mmap从磁盘到虚拟空间的映射也必须是页

- 内核可以跟踪被内存映射的底层对象(文件)的大小，进程可以合法的访问在当前文件大小以内又在映射区以内的那些字节，也就是说，如果文件的大小一直在扩张，只要在映射区域范围内的数据，进程都可以合法得到，这和映射建立时文件的大小无关

- 映射建立后，即文件关闭，映射依然存在，因为映射的是磁盘地址，不是文件本身，和文件句柄无关。同时可用于进程内通信的有效地址空间不完全受限于被映射文件的大小，因为是按页映射

### 代码演示

- 使用C语言演示

```c
#include "sys/stat.h"
#include "fcntl.h"
#include "sys/mman.h"
#include "unistd.h"
#include "stdio.h"
#include "errno.h"
#include "string.h"

#define TRUE 1
#define FALSE -1
#define FILE_SIZE 100

#define MMAP_FILE_PATH "./mmap.txt"

size_t get_file_size(const char* file_path) {
    struct stat buf;
    if (stat(file_path, &buf) < 0) {
        printf("%s[%d]%s",__FUNCTION__,__LINE__,strerror(errno));
        return -1;
    }
    return buf.st_size;
}

int main() {
    int fd = -1;

    void *result;

    int lseek_result = -1;
    int file_length = -1;

    // open file
    fd = open(MMAP_FILE_PATH, O_RDWR|O_CREAT, S_IRUSR|S_IWUSR|S_IRGRP|S_IWGRP|S_IROTH|S_IWOTH);
    if (fd == -1) {
        printf("open failed. %s\n", strerror(errno));
        return FALSE;
    }

    // lseek移动文件的读写位置 off_t lseek(int fd, off_t offset, int whence)
    // 每一个已打开的文件都有一个读写位置，除了使用追加的方式外，默认都是指向文件开头
    // 调用read()和write()时，读写位置会随之增加，fd是已打开的文件描述词，参数offset以根据
    // 参数whence来移动读写位置的位移数
    // SEEK_SET 参数offset即新的读写位置
    // SEEK_CUR 以目前的读写位置往后增加offset个位移量
    // SEEK_END 将读写位置指向文件尾后再增加offset个位移量，允许offset出现负值
    // 这里将文件指向末尾的位置
    lseek_result = lseek(fd, FILE_SIZE - 1, SEEK_SET);
    if (lseek_result == -1) {
        printf("seek failed. %s\n", strerror(errno));
        return FALSE;
    }

    // ssize_t	 write(int __fd, const void * __buf, size_t __nbyte) __DARWIN_ALIAS_C(write);
    // write()会把参数buf所指定的内存写入__nbyte个字节到参数fd所指的文件内
    write(fd, "\0", 1);

    // 将文件指向开头
    lseek(fd, 0, SEEK_SET);

    // 获取文件大小，由于已经在文件尾写入了内容，所以可以获取到文件大小
    file_length = get_file_size(MMAP_FILE_PATH);
    if (file_length == -1) {
        printf("get file size failed \n");
        return FALSE;
    }
    printf("file size: %d\n", file_length);

    // 调用mmap，将文件和内存进行映射
    result = mmap(0, file_length, PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0);
    if (result == (void*)-1) {
        printf("mmap failed. %s\n", strerror(errno));
        return FALSE;
    }

    // 关闭文件描述符
    close(fd);

    // 写入内存到mmap的地址
    strncpy(result, "hello world.\n", file_length);

    // 解除映射关系
    munmap(0, file_length);

    return 0;
}
```
- java代码

```java
import org.junit.Test;
import sun.misc.Cleaner;
import sun.nio.ch.DirectBuffer;

import java.io.IOException;
import java.io.RandomAccessFile;
import java.nio.MappedByteBuffer;
import java.nio.channels.FileChannel;

/**
 * @author asheng
 * @since 2021/7/20
 */
public class MmapTest {

    @Test
    public void test() throws IOException {
        RandomAccessFile file = new RandomAccessFile("./mmap.log", "rw");

        int size = 100;

        MappedByteBuffer mmap = file.getChannel().map(FileChannel.MapMode.READ_WRITE, 0, size);

        System.out.println(mmap.capacity());
        System.out.println(mmap.position());
        System.out.println(mmap.limit());

        mmap.put("hello world\n".getBytes());

        System.out.println(mmap.capacity());
        System.out.println(mmap.position());
        System.out.println(mmap.limit());

        if (mmap instanceof DirectBuffer) {
            Cleaner cleaner = ((DirectBuffer) mmap).cleaner();
            if (cleaner != null) {
                cleaner.clean();
            }
        }

        file.close();
    }
}
```