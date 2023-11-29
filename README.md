# HQOS
A microkernel-based operating system implemented in C language for x86 architecture.

# 序
"THE MAN WHO CHANGED CHROME" 曾经指出如何学习计算机基础知识:
> 自己写一个CPU,在自己写的CPU上运行自己写的操作系统,然后用自己写的编译器编译运行一个程序.

于是便有了写一个操作系统的想法.在粗略看完一遍《操作系统真相还原》后,感觉从引导操作系统到实现各种操作系统概念的完整过程工作量有点太大了,暂时还没有那么多时间和精力来完成,便暂且搁置了.

感谢MIT的6.828课程,让我能在有限的时间里一步一步地实现一个操作系统雏形JOS.在接下来的一段时间,我会继续开发扩展该操作系统,并重构目前实现模型中我不太喜欢的实现,就叫它HQOS吧.

本博客记录当前版本HQOS的设计与实现,你可以在github上找到其历史文档及代码实现.
博客地址: hanqi-blogs.cn/2023/HQOS-Design-and-Implementation

# 设计与实现 Version 0.0
## 总览
HQOS使用C语言进行开发,是一个基于x86架构的Unix-like微内核操作系统,你可以从中找到很多Unix-like的设计和功能.一些模块如File system,Network等在用户态实现.
1. Memory management
2. Processes and Threads
3. File systems
4. Network
5. System calls, Interrupts, Exceptions
6. User library
## Memory management 
### Physical Page Management 
#### Keep track of (physical) page frames
HQOS以页面粒度进行物理内存管理.对于每一个物理页,使用一个PageInfo结构来跟踪其的使用情况(空闲或使用中,被多少"用户"使用).所有的PageInfo结构位于一个连续的数组中,PageInfo结构在数组中的索引与物理页的地址相关联(IDX <---> PGNUM = PADDR / PGSIZE).
```c
struct PageInfo {
	// Next page on the free list.
	struct PageInfo *pp_link;

	// pp_ref is the count of pointers (usually in page table entries)
	// to this page, for pages allocated using page_alloc.
	// Pages allocated at boot time using pmap.c's
	// boot_alloc do not have valid reference count fields.

	uint16_t pp_ref;
};
```

#### Dynamically allocate / free page frames
##### Interface
```c
//For kernel:
//请求分配一个物理页
struct PageInfo *page_alloc(int alloc_flags);
//减少对某物理页的一次引用
void
page_decref(struct PageInfo* pp);
//释放某物理页
void	page_free(struct PageInfo *pp);

//For User:
//分配并映射一个页面
int	sys_page_alloc(envid_t env, void *pg, int perm);
//映射一个页面
int	sys_page_map(envid_t src_env, void *src_pg,
		     envid_t dst_env, void *dst_pg, int perm);
//取消一个页面的映射
int	sys_page_unmap(envid_t env, void *pg);
```
##### Implementation
HQOS以单向链表的数据结构组织空闲物理页.page_alloc与page_free分别从空闲链表中取出一个物理页或加入一个到空闲链表中.在目前的实现中,page_free仅在page_decref中调用,当且仅当某物理页不再被任何一处引用,释放该物理页.

### Virtual memory management
#### Page translation
##### Interface
```c
//For kernel:
//插入一个页面到页表中(映射一个虚拟地址对应的页面)
int	page_insert(pde_t *pgdir, struct PageInfo *pp, void *va, int perm);
//从页表中移除一个页面(取消一个虚拟地址对应页面的映射)
void	page_remove(pde_t *pgdir, void *va);
//寻找虚拟地址对应页面的PageInfo结构及页表条目
struct PageInfo *page_lookup(pde_t *pgdir, void *va, pte_t **pte_store);
```
##### Implementation
HQOS运行在x86's protected-mode memory management architecture之上.
逻辑地址(虚拟地址)经Segmentation Mechanism转换为线性地址,再经由Paging Mechanism转换到物理地址,最终送上硬件总线.

HQOS仅通过页面翻译过程完成虚拟地址到物理地址的转换,(实现的方式是: 将全局描述符表中所有条目的段基址设置为0,段界限设置为0xffffffff,此时线性地址就等于虚拟地址.),segmentation在HQOS的实现中更多地作用于权限控制.

HQOS使用二级页表来完成页面翻译,一个页目录表中有1024个页表条目,一个页表中有1024个页条目,每个页大小为4096字节.(当然,这是x86硬件决定的)
#### Pagefault handling
```c
//For User
sys_env_set_pgfault_upcall(envid_t envid, void *func)
```
HQOS对于内核态的页面错误,产生Kernel panic.而对于用户态的页面错误,销毁产生页面错误的用户进程或派发到一个完成注册的用户态页面错误处理函数.

#### Page replacement (待实现)

## Processes and Threads 
### Processes 
#### Process model
HQOS使用术语"环境(Environment)“来表示进程.
    HQOS使用struct Env的数组来管理系统中所有进程(无论进程是否存在,进程状态由env_status标识),数组的大小即为HQOS允许的最大进程数NENV.
```c
struct Env {
	struct Trapframe env_tf;	// Saved registers
	struct Env *env_link;		// Next free Env
	envid_t env_id;			// Unique environment identifier
	envid_t env_parent_id;		// env_id of this env's parent
	enum EnvType env_type;		// Indicates special system environments
	unsigned env_status;		// Status of the environment
	uint32_t env_runs;		// Number of times environment has run
	int env_cpunum;			// The CPU that the env is running on

	// Address space
	pde_t *env_pgdir;		// Kernel virtual address of page dir

	// Exception handling
	void *env_pgfault_upcall;	// Page fault upcall entry point

	bool env_ipc_recving;		// Env is blocked receiving
	void *env_ipc_dstva;		// VA at which to map received page
	uint32_t env_ipc_value;		// Data value sent to us
	envid_t env_ipc_from;		// envid of the sender
	int env_ipc_perm;		// Perm of page mapping received
};
```

每个进程有自己独立的页目录表,即拥有一个独立的32位虚拟地址空间.这意味着同一个物理页可以映射到多个进程中,这是HQOS目前进程间通信实现的关键.HQOS中属于内核的物理空间完整的映射到每个进程的虚拟地址空间中.

进程目前不具有也不需要独立的内核堆栈.

#### Process creation
##### Interface
```c
//For Kernel:
// Allocates a new env with env_alloc, loads the named elf
// binary into it with load_icode, and sets its env_type.
// This function is ONLY called during kernel initialization,
// before running the first user-mode environment.
// The new env's parent ID is set to 0.
//
void
env_create(uint8_t *binary, enum EnvType type)

//For User:
//Create the new environment
//the register set is copied from the current environment
//新进程的虚拟地址空间仅初始化了内核部分
envid_t
sys_exofork(void)

```
#### Process scheduling
##### Interface
```c
//For Kernel:
// Choose a user environment to run and run it.
void
sched_yield(void);

//For User:
void
sys_yield(void);
```
##### Implementation
HQOS的进程调度采用轮转调度(round robin).但目前还没有加入时间片的概念.每当时钟中断产生,时钟中断处理例程从当前进程在进程数组中的位置开始遍历其他进程,切换到下一个可运行状态的进程.若无其他可运行的进程,进程重启自身.内核进程不会因时钟中断发生调度(因为目前处理器在内核态时屏蔽中断).
进程当然也可主动让出处理器.

这样的实现当然是让人非常不满的,很快其将会重新实现.
#### Interprocess Communication (IPC)
##### Locking
HQOS支持多处理器,可能会有多个进程同时进入内核态.HQOS使用一个内核整体的Kernel Lock,进入内核时获取,离开内核时释放,也就是说,HQOS目前仅支持单个内核进程运行.这样的实现依赖于HQOS内核空间对所有进程而言是共享的.
##### Sending and Receiving Messages
###### By Value
通过用户进程可读的进程表(envs数组)来传递一字长的value.然而该value对所有进程都可见,且一字长的数据太短,所以一般用来辅助By page的传递方式.
###### By page
将sender的某页面映射到recver的虚拟地址空间中.
sender指示接收进程及页面权限.
recver指示将要映射到的虚拟地址.

recver阻塞直到sender将其唤醒.检测消息来源是否为期望的sender的工作由用户态调用者完成.
```c
 int
sys_ipc_try_send(envid_t envid, uint32_t value, void *srcva, unsigned perm)
int
sys_ipc_recv(void *dstva)
```

### Threads 线程(待实现)


## File systems 
### On-Disk File System Structure 
#### Sectors and Blocks
磁盘每个扇区大小为512字节,JOS使用的块大小为512字节.
#### Superblocks
HQOS仅有一个超级块,位于磁盘1(第二个磁盘)的块1(第二个扇区).

```c
#define FS_MAGIC	0x53465148	// related vaguely to 'HQFS'

struct Super {
	uint32_t s_magic;		// Magic number: FS_MAGIC
	uint32_t s_nblocks;		// Total number of blocks on disk
	struct File s_root;		// Root directory node
};

```

#### File Meta-data
文件的元数据保存在其所在的目录文件中.
f_direct数组中记录的为保存该文件数据的磁盘的块号(直接块).
f_indirect为间接块的块号,该块的数据为该文件对应的其他直接块的块号,相当于direct数组的扩展.

```c
// Number of block pointers in a File descriptor
#define NDIRECT		10
// Number of direct block pointers in an indirect block
#define NINDIRECT	(BLKSIZE / 4)

struct File {
	char f_name[MAXNAMELEN];	// filename
	off_t f_size;			// file size in bytes
	uint32_t f_type;		// file type

	// Block pointers.
	// A block is allocated iff its value is != 0.
	uint32_t f_direct[NDIRECT];	// direct blocks
	uint32_t f_indirect;		// indirect block

	// Pad out to 256 bytes; must do arithmetic in case we're compiling
	// fsformat on a 64-bit machine.
	uint8_t f_pad[256 - MAXNAMELEN - 8 - 4*NDIRECT - 4];
} __attribute__((packed));	// required only on some 64-bit machines
```
#### Directories
保存文件的元数据,目录本身也是一个文件,其元数据保存在其上层目录中.根目录的内容保存在超级块中.

### File System Structure
#### Disk Access
HQOS没有在内核中添加IDE磁盘驱动程序及相关系统调用,而是将其实现于用户态的文件系统中.文件系统进程使用了EFLAGS中的IOPL位,有权限执行IO指令.
#### Block Cache
HQOS借助虚拟地址空间来实现块缓存(不得不夸一下JOS的设计者们,好多巧妙的实现).文件系统的DISKMAP(0x10000000 )至DISKMAP+DISKMAX(0xD0000000)用来缓存磁盘,共3GB空间,故HQOS仅支持3GB以下的磁盘.
```c
// Return the virtual address of this disk block.
void*
diskaddr(uint32_t blockno)
{
	if (blockno == 0 || (super && blockno >= super->s_nblocks))
		panic("bad block number %08x in diskaddr", blockno);
	return (char*) (DISKMAP + blockno * BLKSIZE);
}
```


HQOS并不直接读取完整的3GB磁盘空间到地址空间中,而是当访问磁盘块所在页时,触发页面错误,由错误处理例程从磁盘中读入并完成映射.
```c
// Fault any disk block that is read in to memory by
// loading it from disk.
static void
bc_pgfault(struct UTrapframe *utf)
```

文件系统依赖页面的PTE_D(dirty)位来决定是否需要刷新块(写回到磁盘).

由于目前HQOS还未实现页面置换算法,我很担心Block Cache对内存的消耗.
#### Block Bitmap
HQOS使用Bitmap来记录磁盘中Block的使用状态,进而完成分配释放等操作.
### The file system interface
文件系统接口是以进程间通信的方式提供的.文件系统持续无休止的接收其他进程的IPC请求,将请求调度到特定的处理例程,完成请求的文件操作.

## Network
我暂时还不熟悉计算机网络相关知识,所以对Network部分的说明不会太多.
### Network System
这是HQOS网络体系的总览.HQOS使用E1000网卡,在Kernel中实现由E1000的驱动程序,负责将packet传给E1000或从E1000接收packet.与E1000驱动程序进行交互的是用户态的Network server,由核心网络进程、输出进程、输入进程构成.
核心网络进程的实现使用了开源的IwIP的TCP/IP协议套件.

其他用户进程通过IPC机制与Network server交互.
![](https://obsidian-1314737433.cos.ap-beijing.myqcloud.com/202311152114274.png)


## System calls, Interrupts, Exceptions
实话说我不认为System calls, Interrupts, Exceptions的概念是与其他操作系统概念并列的,不过作为OS实现的重要部分,我需要单独的一章来说明,之前已经在其他操作系统概念的说明中介绍过的部分不再说明.

### Protected Control Transfer
#### The Interrupt Descriptor Table
异常和中断导致处理器从用户模式切换到内核模式,内核模式的入口点和中断处理例程由内核明确规定.在HQOS中,所有的中断或异常都会导致切换到内核态,并由内核态进行处理(即使HQOS允许用户态页面错误处理例程,那也是先进入内核再由内核处理例程dispatch),这是在中断描述符表IDT中实现的.

HQOS目前仅支持中断门,意味着HQOS不支持嵌套中断.
#### The Task State Segment
发生中断时,处理器将状态保存到TSS段中ESP3和SS3中,并加载ESP0和SS0,从内核返回是逆过程.HQOS目前仅使用TSS作为用户/内核堆栈的切换.HQOS目前为每个CPU分配了一个内核堆栈.

### syscall
```c
// syscall.c
void	sys_cputs(const char *string, size_t len);
int	sys_cgetc(void);
envid_t	sys_getenvid(void);
int	sys_env_destroy(envid_t);
void	sys_yield(void);
static envid_t sys_exofork(void);
int	sys_env_set_status(envid_t env, int status);
int	sys_env_set_trapframe(envid_t env, struct Trapframe *tf);
int	sys_env_set_pgfault_upcall(envid_t env, void *upcall);
int	sys_page_alloc(envid_t env, void *pg, int perm);
int	sys_page_map(envid_t src_env, void *src_pg,
		     envid_t dst_env, void *dst_pg, int perm);
int	sys_page_unmap(envid_t env, void *pg);
int	sys_ipc_try_send(envid_t to_env, uint32_t value, void *pg, int perm);
int	sys_ipc_recv(void *rcv_pg);
unsigned int sys_time_msec(void);
int sys_tx_pkt(char* buf,size_t nbytes);
```

## User library
```c
// exit.c
void	exit(void);

// pgfault.c
void	set_pgfault_handler(void (*handler)(struct UTrapframe *utf));

// readline.c
char*	readline(const char *buf);

// ipc.c
void	ipc_send(envid_t to_env, uint32_t value, void *pg, int perm);
int32_t ipc_recv(envid_t *from_env_store, void *pg, int *perm_store);
envid_t	ipc_find_env(enum EnvType type);

// fork.c
#define	PTE_SHARE	0x400
envid_t	fork(void);

// fd.c
int	close(int fd);
ssize_t	read(int fd, void *buf, size_t nbytes);
ssize_t	write(int fd, const void *buf, size_t nbytes);
int	seek(int fd, off_t offset);
void	close_all(void);
ssize_t	readn(int fd, void *buf, size_t nbytes);
int	dup(int oldfd, int newfd);
int	fstat(int fd, struct Stat *statbuf);
int	stat(const char *path, struct Stat *statbuf);

// file.c
int	open(const char *path, int mode);
int	ftruncate(int fd, off_t size);
int	remove(const char *path);
int	sync(void);

// pageref.c
int	pageref(void *addr);

// sockets.c
int     accept(int s, struct sockaddr *addr, socklen_t *addrlen);
int     bind(int s, struct sockaddr *name, socklen_t namelen);
int     shutdown(int s, int how);
int     connect(int s, const struct sockaddr *name, socklen_t namelen);
int     listen(int s, int backlog);
int     socket(int domain, int type, int protocol);

// nsipc.c
int     nsipc_accept(int s, struct sockaddr *addr, socklen_t *addrlen);
int     nsipc_bind(int s, struct sockaddr *name, socklen_t namelen);
int     nsipc_shutdown(int s, int how);
int     nsipc_close(int s);
int     nsipc_connect(int s, const struct sockaddr *name, socklen_t namelen);
int     nsipc_listen(int s, int backlog);
int     nsipc_recv(int s, void *mem, int len, unsigned int flags);
int     nsipc_send(int s, const void *buf, int size, unsigned int flags);
int     nsipc_socket(int domain, int type, int protocol);

// spawn.c
envid_t	spawn(const char *program, const char **argv);
envid_t	spawnl(const char *program, const char *arg0, ...);

// console.c
void	cputchar(int c);
int	getchar(void);
int	iscons(int fd);
int	opencons(void);

// pipe.c
int	pipe(int pipefds[2]);
int	pipeisclosed(int pipefd);

// wait.c
void	wait(envid_t env);

//sleep.c
void sleep(int ms);

int transpackt(char* buf,size_t nbytes);
int recvpackt(char* buf,size_t max_bytes,int* lenth_store);
```
# 资料
[Intel 80386 Reference Manual](https://pdos.csail.mit.edu/6.828/2018/readings/i386/toc.htm)
[Intel's Software Developer's Manual for the E1000](https://pdos.csail.mit.edu/6.828/2018/readings/hardware/8254x_GBe_SDM.pdf)