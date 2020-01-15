---
title: 给 PCB 添加优先级
---

1. 给 PCB （进程控制块）添加优先级属性

   ```shell
vim proc.h
   ```
   
   在 `struct proc` 中添加一行
   
   ```c
   int priority;      // Process priority (0-20); lower value, higher priority
   ```
   
2. ps 实现

   实现 ps 系统调用用来查看我们的进程信息

   ```c
   // cps designed by myself
   int 
   cps(){
   	struct proc *p;
   
   	// Enable interrupts on this processor
   	sti();
   
   	// loop over process table looking for process with pid
   	acquire(&ptable.lock);
   	cprintf("name \t pid \t state \t \t priority \n");
   	for(p = ptable.proc; p < &ptable.proc[NPROC]; p++) {
   		if( p->state == SLEEPING)
   			cprintf("%s \t %d  \t SLEEPING \t %d\n ", p->name, p->pid, p->priority);
   		else if( p->state == RUNNING )
   			cprintf("%s \t %d  \t RUNNING \t %d\n ", p->name, p->pid, p->priority);
   		else if( p->state == RUNNABLE )
   			cprintf("%s \t %d  \t RUNNABLE \t %d\n ", p->name, p->pid, p->priority);
   	}
   
   	release(&ptable.lock);
   
   	return 22;
   }
   
   ```

3. 修改 `allocproc` 给 `priority` 提供默认值

   ```c
   found:
     p->state = EMBRYO;
     p->pid = nextpid++;
     p->priority = 10;           //default priority
     release(&ptable.lock);
   
   ```

4. 创建一个用户程序，该程序会创建一些子进程进行实验

   ```c
   #include "types.h"
   #include "stat.h"
   #include "user.h"
   #include "fcntl.h"
   
   int 
   main(int argc, char* argv[]){
   	int k, n, id;
   	double x = 0, z, d;
   
   	if( argc < 2 )
   		n = 1;        //default value
   	else 
   		n = atoi ( argv[1] ); //from command line
   	if ( n < 0 || n > 20 )
   		n = 2;
   
   	if ( argc < 3)
   		d = 1.0;
   	else
   		d = atoi ( argv[2] );
   
   	x = 0;
   	id = 0;
   	for ( k = 0; k < n; k++) {
   		id = fork();
   		if ( id < 0 ) {
   			printf(1, "%d failed in fork!\n", getpid());
   		} else if ( id > 0 ) { //parent
   			printf(1,"Parent %d creating child %d\n", getpid(), id );
   		} else { //child 
   			printf(1, "Child %d created\n", getpid());
   			for ( z = 0; z < 8000000.0; z += d ){
   				x = x + 3.14 * 89.64;  // useless calculations to consume CPU time  
   			}
   			break;
   		}
   	}
   	exit();
   }
   
   ```

   运行 `foo` 程序的时候你会看见僵尸进程，因为子进程没有运行完，占用着 PCB，而父母必须在进程表中等待，一旦检测到子进程结束，父进程将清除相应的 PCB。你可以在父进程结束前加 wait() 等待。

5. 修改进程的优先级

   修改优先级的函数

   vim proc.c

   ```c
   // change priority
   int 
   chpr( int pid, int priority ) {
   	struct proc *p;
   
   	acquire(&ptable.lock);
   	for ( p = ptable.proc; p < &ptable.proc[NPROC]; p++){
   		if ( p->pid == pid ) {
   			p->priority = priority;
   			break;
   		}
   	}
   	release(&ptable.lock);
   
   	return pid;
   }
   ```

6. 将前面两个函数添加到系统调用中。

   vim sysproc.c

   ```c
   int
   sys_cps ( void )
   {
   	return cps ();
   }
   
   int
   sys_chpr ( void )
   {
   	int pid, pr;
   	if ( argint(0, &pid) < 0 )
   		return -1;
   	if ( argint(1, &pr) < 0 )
   		return -1;
   
   	return chpr ( pid, pr );
   }
   
   ```

   

7. 添加系统调用

   vim syscall.h

   ```c
   #define SYS_cps    22
   #define SYS_chpr   23
   ```

8. 添加头文件声明

   vim defs.h

   ```c
   int             cps(void);
   int             chpr( int pid, int priority );
   ```

   在 swtch.S 之前添加

   vim user.h

   ```c
   int             cps(void);
   int             chpr( int pid, int priority );
   ```

   在 ulib,c 之前添加

   vim usys.S

   ```
   SYSCALL(cps)
   SYSCALL(chpr)
   ```

   vim syscall.c

   ```
   extern int sys_cps(void);
   extern int sys_chpr(void);
   ```

   在 *syscalls[] 数组前添加

   ```
   [SYS_cps]     sys_cps,
   [SYS_chpr]    sys_chpr,
   ```

   在 *syscalls[] 数组内添加

9. 调用 chpr 系统调用，利用下面的 nice 程序改变优先级。

   vim nice.c

   ```c
   #include "types.h"
   #include "stat.h"
   #include "user.h"
   #include "fcntl.h"
   
   int
   main(int argc, char* argv[]){
       int priority, pid;
   
       if (argc < 3) {
           printf(2, "Usage: nice pid priority\n" );
           exit();
       }
       pid = atoi ( argv[1] );
       priority = atoi ( argv[2] );
       if ( priority < 0 || priority > 20 ) {
           printf(2, "Invaild priority (0-20)!\n" );
           exit();
       }
       printf(1, " pid=%d, pr=%d\n", pid, priority );
       chpr ( pid, priority );
   
       exit();
   }
   ```

      

## 总结

为了实现优先级修改，修改和创建了如下文件

```
Makefile   添加应用
defs.h     添加声明
proc.c     实现函数
proc.h     添加优先级属性
syscall.c  添加系统调用到系统调用数组里
syscall.h  添加系统调用
sysproc.c  添加系统函数调用
user.h     添加声明
usys.S     添加声明
foo.c      生成子进程进行实验
nice.c     修改优先级
ps.c       添加查看进程信息功能
```

