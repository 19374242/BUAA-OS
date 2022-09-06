注意`(x<<24)>>24`可能导致结果为负数，所以最好`x&(0x000000ff)`

## lab3-2

题目要求： 在本次课上测试中，我们对env_pri的含义进行如下约定：

1. env_pri的第0-7位仍然表示进程的优先级（PRI）
2. env_pri的第8-15位为功能字段1（FUNC_1）
3. env_pri的第16-23位为功能字段2（FUNC_2）
4. env_pri的第24-31位为功能字段3（FUNC_3）
5. FUNC_1, FUNC_2, FUNC_3均为无符号数

### exam

#### PART1

在这次的课上测试中，我们需要你实现一个新的调度算法，具体要求如下： 1.修改sched_yield函数，使得在进程调度时高优先级（用PRI表示）的进程永远比低优先级（用PRI表示）的进程先调度。 2.为了简化问题，不需要考虑两个进程优先级相同的情况。 例如：如果有两个进程A和B，优先级分别为1和2，且env_status都为ENV_RUNNABLE，则只有B会得到执行；当B的状态不为ENV_RUNNABLE时，A才会得到执行。

#### PART2

在PART1中的sched_yield上继续修改，要求如下： 在当前进程（记为进程X）被时钟中断暂停执行时，进行如下操作： 1.根据FUNC_1的值更新进程X的优先级： 若FUNC_1的值为0，则不修改进程X的优先级 若FUNC_1的值为正，则将进程X的优先级减少FUNC_1的值（如果减少之后优先级小于0则优先级置零） 2.完成PART1中的进程调度

#### 解答

有一些坑：

1. `(x<<24)>>24`可能导致结果为负数，所以最好`x&(0x000000ff)`
2. 没有时间片的概念，所以不存在一个进程的时间片用完这种说法，所以不用在两个list之间倒腾来倒腾去
3. 本地测试时，永远输出一个数字，不存在进程切换

### Extra

#### 题目要求

Extra部分要求在exam部分的基础上继续修改sched_yield： 在当前进程（记为进程X）被时钟中断暂停执行时，进行如下操作： 1.如果当前的时钟中断是第FUNC_2个（从1开始计数）时钟中断，那么在接下来的FUNC_3个时间片里，将进程X的env_status改为ENV_NOT_RUNNABLE，不进行调度，FUNC_3个时间片之后（无论这时是哪一个进程被中断），将进程X的env_status改为ENV_RUNNABLE，参与调度。 2.否则，不进行处理（注意，为了简化问题，我们保证这一部分测试数据中FUNC_1始终为0） 3.按照PART1中的策略进行调度

#### 解答

题干中虽然提到了“时间片”的概念，其实一个时间片就是一次中断。

测试样例很弱，挺多人过Extra的，所以不能保证自己的程序是否正确。

我觉得简单的做法应该是修改env.h里的env结构体，置一个time_to_run，每个中断更新下，更新到0的话就置为RUNNABLE。然后遍历list取RUNNABLE里pri最大的就行。

一个list就够了。

```
#include <env.h>
#include <pmap.h>
#include <printf.h>
#define PRI(X) (((X)->env_pri) & 0xff)
#define FUNC_1(X) ((((X)->env_pri) >> 8) & 0xff)
#define FUNC_2(X) ((((X)->env_pri) >> 16) & 0xff)
#define FUNC_3(X) ((((X)->env_pri) >> 24) & 0xff)

/* Overview:
 *  Implement simple round-robin scheduling.
 *
 *
 * Hints:
 *  1. The variable which is for counting should be defined as 'static'.
 *  2. Use variable 'env_sched_list', which is a pointer array.
 *  3. CANNOT use `return` statement!
 */
/*** exercise 3.14 ***/
void sched_yield(void)
{
    static int count = 0; // remaining time slices of current env
    static int point = 0; // current env_sched_list index
	static struct Env *e = NULL;
	struct Env *tempe;
	int maxpri = 0;
    
    /*  hint:
     *  1. if (count==0), insert `e` into `env_sched_list[1-point]`
     *     using LIST_REMOVE and LIST_INSERT_TAIL.
     *  2. if (env_sched_list[point] is empty), point = 1 - point;
     *     then search through `env_sched_list[point]` for a runnable env `e`, 
     *     and set count = e->env_pri
     *  3. count--
     *  4. env_run()
     *
     *  functions or macros below may be used (not all):
     *  LIST_INSERT_TAIL, LIST_REMOVE, LIST_FIRST, LIST_EMPTY
     */
// 	if (count <= 0) {
// 		do {
// 			if (LIST_EMPTY(&env_sched_list[point])) {
// 				point = 1 - point;
// 			}
// 
// 			e = LIST_FIRST(&env_sched_list[point]);
// 
// 			if (e != NULL) {
// 				LIST_REMOVE(e, env_sched_link);
// 				LIST_INSERT_TAIL(&env_sched_list[1 - point], e, env_sched_link);
// 				count = e->env_pri;
// 			}
// 		} while (e == NULL || e->env_status != ENV_RUNNABLE);
// 	}
// 	count --;
	 while (!LIST_EMPTY(&env_sched_list[1])) { // abandon list[1]
	 	tempe = LIST_FIRST(&env_sched_list[1]);
	 	LIST_REMOVE(tempe, env_sched_link);
	 	LIST_INSERT_HEAD(&env_sched_list[0], tempe, env_sched_link);
	 	
	 }
	 if (e != NULL) {
		 if (FUNC_1(e) > 0) {
			 if (PRI(e) >= FUNC_1(e)) {
				 e->env_pri -= FUNC_1(e);
			 } else {
				 e->env_pri = 0;
			 }
		 }
	 }
	 maxpri = 0;
	 LIST_FOREACH(tempe, &env_sched_list[0],env_sched_link) {
	 	if (PRI(tempe) > maxpri && tempe->env_status == ENV_RUNNABLE) {
	 		e = tempe;
	 		maxpri = PRI(tempe);
	 	}
	 }
	
	env_run(e);
}
```

