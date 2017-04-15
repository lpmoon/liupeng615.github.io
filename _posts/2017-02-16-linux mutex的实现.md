## mutex的结构
```
struct mutex {
/* 1: unlocked, 0: locked, negative: locked, possible waiters */
atomic_t	 count;
spinlock_t	 wait_lock;
struct list_head	wait_list;
#if defined(CONFIG_DEBUG_MUTEXES) || defined(CONFIG_MUTEX_SPIN_ON_OWNER)
struct task_struct	*owner;
#endif
#ifdef CONFIG_MUTEX_SPIN_ON_OWNER
struct optimistic_spin_queue osq; /* Spinner MCS lock */
#endif
#ifdef CONFIG_DEBUG_MUTEXES
void	 *magic;
#endif
#ifdef CONFIG_DEBUG_LOCK_ALLOC
struct lockdep_map	dep_map;
#endif
};
```
* count
用于计数，如果为1表示未加锁。0表示已经加锁。负数表示可能存在等待
* wait_lock
自旋锁，用于等待。
* wait_list
等待队列

## mutex的加锁操作
mutex调用mutex_lock完成加锁操作。
```
void __sched mutex_lock(struct mutex *lock)
{
might_sleep();
/*
* The locking fastpath is the 1->0 transition from
* 'unlocked' into 'locked' state.
*/
__mutex_fastpath_lock(&lock->count, __mutex_lock_slowpath);
mutex_set_owner(lock);
}
```
在mutex_lock中先调用__mutex_fastpath_lock进行快加锁，如果加锁失败则调用__mutex_lock_slowpath进行慢加锁。这里先看一下快加锁的步骤，

### __mutex_lock_slowpath
```
static inline void
__mutex_fastpath_lock(atomic_t *count, void (*fail_fn)(atomic_t *))
{
if (unlikely(atomic_dec_return_acquire(count) < 0))
fail_fn(count);
}
```
__mutex_fastpath_lock最终会调用atomic_dec_return完成count的操作。
```
#define  atomic_dec_return_acquire	atomic_dec_return
```
而这个函数是与体系结构相关的，不同的体系结构有不同的实现，下面的代码以x86结构为例，该函数位于**arch/x86/include/asm/atomic.h**
```
/**
 * atomic_add_return - add integer and return
 * @i: integer value to add
 * @v: pointer of type atomic_t
 *
 * Atomically adds @i to @v and returns @i + @v
 */
static __always_inline int atomic_add_return(int i, atomic_t *v)
{
return i + xadd(&v->counter, i);
}
/**
 * atomic_sub_return - subtract integer and return
 * @v: pointer of type atomic_t
 * @i: integer value to subtract
 *
 * Atomically subtracts @i from @v and returns @v - @i
 */
static __always_inline int atomic_sub_return(int i, atomic_t *v)
{
return atomic_add_return(-i, v);
}

#define atomic_inc_return(v)  (atomic_add_return(1, v))
#define atomic_dec_return(v)  (atomic_sub_return(1, v))
```
```
#define __xchg_op(ptr, arg, op, lock)	 \
({	 \
        __typeof__ (*(ptr)) __ret = (arg);	 \
        switch (sizeof(*(ptr))) {	 \
        case __X86_CASE_B:	 \
        asm volatile (lock #op "b %b0, %1\n"	 \
              : "+q" (__ret), "+m" (*(ptr))	\
              : : "memory", "cc");	 \
              break;	 \
        case __X86_CASE_W:	 \
        asm volatile (lock #op "w %w0, %1\n"	 \
              : "+r" (__ret), "+m" (*(ptr))	\
              : : "memory", "cc");	 \
            break;	 \
        case __X86_CASE_L:	 \
        asm volatile (lock #op "l %0, %1\n"	 \
              : "+r" (__ret), "+m" (*(ptr))	\
              : : "memory", "cc");	 \
        break;	 \
        case __X86_CASE_Q:	 \
        asm volatile (lock #op "q %q0, %1\n"	 \
              : "+r" (__ret), "+m" (*(ptr))	\
              : : "memory", "cc");	 \
        break;	 \
        default:	 \
        __ ## op ## _wrong_size();	 \
}	 \
__ret;	 \
})

/*
 * xadd() adds "inc" to "*ptr" and atomically returns the previous
 * value of "*ptr".
 *
 * xadd() is locked when multiple CPUs are online
 */
#define __xadd(ptr, inc, lock)	__xchg_op((ptr), (inc), xadd, lock)
#define xadd(ptr, inc)	 __xadd((ptr), (inc), LOCK_PREFIX)
```
最终调用xadd命令完成操作。xadd命令的语义是(http://x86.renejeschke.de/html/file_module_x86_id_327.html):
> 
Exchanges the first operand (destination operand) with the second operand (source operand), then loads the sum of the two values into the destination operand. The destination operand can be a register or a memory location; the source operand is a register.
This instruction can be used with a LOCK prefix to allow the instruction to be executed atomically.

用伪代码描述就是，
```
Temporary = Source + Destination;
Source = Destination;
Destination = Temporary;
```
回到上面的代码
```
asm volatile (lock #op "w %w0, %1\n"	 \
      : "+r" (__ret), "+m" (*(ptr))	\
      : : "memory", "cc"); 
```
__ret对应的是-1，ptr对应的mutex->count。上面的代码执行后，__ret指向的值变为mutex-> count, ptr指向的值变为mutex->count - 1，也就是说值递减了1。代码中的volatile保证指令的顺序，lock保证了操作的原子性。xadd "w %w0, %1"中的w表示将操作数转为2个字节宽，同理l表示4字节宽。为了验证这个结论我们可以写一个简单的例子，

```
#include<stdio.h>
#include <stdlib.h>
main() {

    int i=10,j=11;

    asm("xadd %0, %1\n": "+q" (i), "+m" (j): : "memory", "cc");

    printf("i=%d j=%d\n",i,j);
}

```
使用gcc编译上面的代码，并且使用gdb进行调试，
```
gcc -g -o xadd xadd.c
gdb xadd
```
使用disas看代码与汇编的对应关系，
```
(gdb) disas /m main
Dump of assembler code for function main:
3	main() {
   0x0000000000400596 <+0>:	push   %rbp
   0x0000000000400597 <+1>:	mov    %rsp,%rbp
   0x000000000040059a <+4>:	sub    $0x10,%rsp
   0x000000000040059e <+8>:	mov    %fs:0x28,%rax
   0x00000000004005a7 <+17>:	mov    %rax,-0x8(%rbp)
   0x00000000004005ab <+21>:	xor    %eax,%eax

4	
5	    int i=10,j=11;
   0x00000000004005ad <+23>:	movl   $0xa,-0xc(%rbp)
   0x00000000004005b4 <+30>:	movl   $0xb,-0x10(%rbp)

6	
7	    asm("xadd %0, %1\n": "+q" (i), "+m" (j): : "memory", "cc");
   0x00000000004005bb <+37>:	mov    -0xc(%rbp),%eax
   0x00000000004005be <+40>:	xadd   %eax,-0x10(%rbp)
=> 0x00000000004005c2 <+44>:	mov    %eax,-0xc(%rbp)

8	
9	    printf("i=%d j=%d\n",i,j);
   0x00000000004005c5 <+47>:	mov    -0x10(%rbp),%edx
   0x00000000004005c8 <+50>:	mov    -0xc(%rbp),%eax
   0x00000000004005cb <+53>:	mov    %eax,%esi
   0x00000000004005cd <+55>:	mov    $0x400684,%edi
   0x00000000004005d2 <+60>:	mov    $0x0,%eax
   0x00000000004005d7 <+65>:	callq  0x400470 <printf@plt>
   0x00000000004005dc <+70>:	mov    $0x0,%eax

10	}
   0x00000000004005e1 <+75>:	mov    -0x8(%rbp),%rcx
   0x00000000004005e5 <+79>:	xor    %fs:0x28,%rcx
   0x00000000004005ee <+88>:	je     0x4005f5 <main+95>
   0x00000000004005f0 <+90>:	callq  0x400460 <__stack_chk_fail@plt>
   0x00000000004005f5 <+95>:	leaveq 
   0x00000000004005f6 <+96>:	retq 
```
 asm("xadd %0, %1\n": "+q" (i), "+m" (j): : "memory", "cc"); 这个调用被编译成了三条指令，
```
   ## 赋值
   0x00000000004005ad <+23>:	movl   $0xa,-0xc(%rbp)     %rbp - 0xc 中存放10(变量i)
   0x00000000004005b4 <+30>:	movl   $0xb,-0x10(%rbp)   %rbp - 0x10中存放11(变量j)
   ## 开始执行
   0x00000000004005bb <+37>:	mov    -0xc(%rbp),%eax    %eax中存放10  第一个操作数  
   0x00000000004005be <+40>:	xadd   %eax,-0x10(%rbp)  %eax存放11，%rbp - 0x10存放的21(变量j)
   0x00000000004005c2 <+44>:	mov    %eax,-0xc(%rbp)    %rbp - 0xc 中存放的11(变量i)
   ## 结束执行
```
再回到__xchg_op函数，该函数最终返回的就是mutex->count的原始值。

### __mutex_lock_slowpath

