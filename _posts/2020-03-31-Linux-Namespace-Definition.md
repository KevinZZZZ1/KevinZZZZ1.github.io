---
layout: article
title: Linux Namespace 基本概念
mathjax: true
---

# Linux Namespace基本概念

Linux Namespace是Kernel的一个功能，用来**隔离系统资源**（进程树、网络接口、挂载点等）；还可以**实现UID级别的隔离**（也就是说以UID为n的用户虚拟化一个Namespace，在该Namespace中用户具有root权限，而在实际物理机器上还是以UID为n的用户）；再就是**虚拟PID**（从用户角度，每个命名空间像一台单独的Linux计算机有自己的init进程，其他进程依次递增；从父命名空间来看，子命名空间的进程映射到父命名空间的进程上，父命名空间可以知道每个子命名空间的运行状态，而子命名空间之间是隔离的）。

Namespace的API主要使用3个系统调用：`clone()`、`unshare()`、`setns()`以及一些`/proc`文件组成。以下是各个Namespace的类型对应的参数，以及实现的内核版本：

|   Namespace类型   | 系统调用参数  | 内核版本 |
| :---------------: | :-----------: | :------: |
|  Mount Namespace  |  CLONE_NEWNS  |  2.4.19  |
|   UTS Namespace   | CLONE_NEWUTS  |  2.6.19  |
|   IPC Namespace   | CLONE_NEWIPC  |  2.6.19  |
|   PID Namespace   | CLONE_NEWPID  |  2.6.24  |
| Network Namespace | CLONE_NEWNET  |  2.6.29  |
|  User Namespace   | CLONE_NEWUSER |   3.8    |

- `clone()`系统调用：

  `clone()`是创建namespace的一种方式，该系统调用会创建出一个新进程，下面描述了`clone()`的原型：

  ```c
  int clone(int (*child_func)(void *), void *child_stack, int flags, void *arg);
  ```

  其实，`clone()`可以说是更通用版本的`fork()`因为其功能可以通过参数flags控制。当调用时指定了flag参数为CLONE_NEW*中的某几个，那么对应的namespace就会被创建，而且新进程也会进入该namespace。在flag参数中可以指定多个CLONE_NEW参数。

  接下来我们使用一个例子来说明`clone()`系统调用的使用和特点：该例子会创建一个UTS namespace，该namespace会隔离系统的hostname和NIS domain name两个系统标识符，分别通过`sethostname()`和`setdomainname()`两个系统调用设置，使用`uname()`系统调用返回设置结果：

  ```c
  #define _GNU_SOURCE
  #include <sys/wait.h>
  #include <sys/utsname.h>
  #include <sched.h>
  #include <string.h>
  #include <stdio.h>
  #include <stdlib.h>
  #include <unistd.h>
  
  /**
   * A simple error-handling function:
   * print an error message based on the value in 'errno' and terminate the calling process
   */
  #define errExit(msg) do {perror(msg); exit(EXIT_FAILURE);} while(0)
  // start function for cloned child
  static int childFunc(void *arg) {
          struct utsname uts;
  
          // change hostname in UTS namespace of child
  
          if (sethostname(arg, strlen(arg)) == -1)
              errExit("sethostname");
          // Retrieve and display hostname
          if (uname(&uts) == -1)
              errExit("uname");
  
          printf("uts.nodename in child: %s\n", uts.nodename);
  
          // keep the namespace open for a while, by sleeping.
          // this allows some experimentation--for example, another process might join the namespace.
          sleep(100);
  
          return 0; // terminates child
  }
  
  #define STACK_SIZE (1024*1024)  // stack size for cloned child
  static char child_stack[STACK_SIZE];
  int main(int argc, char *argv[]) {
          pid_t child_pid;
          struct utsname uts;
  
          if (argc < 2) {
              fprintf(stderr, "Usage: %s <child-hostname>\n", argv[0]);
              exit(EXIT_FAILURE);
          }
  
          // create a child that has its own UTS namespace;
          // the child commences execution in childFunc()
          child_pid = clone(childFunc, child_stack+STACK_SIZE, CLONE_NEWUTS | SIGCHLD, argv[1]);
          if (child_pid == -1) {
              errExit("clone");
          }
          printf("PID of child created by clone() is %ld\n", (long) child_pid);
  
          // parent falls through to here
          sleep(1); // give child time to change its hostname
  
          // display the hostname in parent's UTS namespace.
          // this will be different from the hostname in child's UTS namespace.
          if (uname(&uts) == -1)
              errExit("uname");
          printf("uts.nodename in parent: %s\n", uts.nodename);
  
          if (waitpid(child_pid, NULL, 0) == -1) // wait for child
              errExit("waitpid");
          printf("child has terminated\n");
  
          exit(EXIT_SUCCESS);
  }
  ```

  上面的程序需要输入一个参数，当程序运行时会创建一个子进程在新的UTS namespace中运行，并且在新的UTS namespace中子进程将hostname替换为给定的参数。下面来仔细分析上面的代码：

  ```c
  child_pid = clone(childFunc, child_stack+STACK_SIZE, CLONE_NEWUTS | SIGCHLD, argv[1]);
  printf("PID of child created by clone() is %ld\n", (long) child_pid);
  ```

  新创建的子进程将会在新的UTS namespace中执行`childFunc()`函数，该函数使用`clone()`的argv[1]作为参数；

  ```c
  sleep(1);
  uname(&uts);
  printf("uts.nodename in parent: %s\n", uts.nodename);
  ```

  主程序将会自行阻塞一段时间，以便使子进程能在其UTS namespace中改变其hostname，主程序使用`uname()`寻找其对应的UTS namespace并展示出来；

  ```c
  sethostname(arg, strlen(arg));
  
  uname(&uts);
  printf("uts.nodename in child: %s\n", uts.nodename);
  ```

  与此同时，由`clone()`函数创建的子进程将执行`childFunc()`函数：首先，将hostname替换为给定的参数，然后使用`uname()`函数进行检索并展示出来修改后的hostname；

  上面解释了程序中部分重点代码，现在我们来编译程序：

  ```shell
  gcc -o demo_uts_namespaces demo_uts_namespace.c
  ```

  执行程序：

  ```
  kevin@kevin-pc:~/namespace$ su						# need privilege to create a UTS namespace
  密码： 
  root@kevin-pc:/home/kevin/namespace# uname -n
  kevin-pc
  root@kevin-pc:/home/kevin/namespace# ./demo_uts_namespaces zyn
  PID of child created by clone() is 6549
  uts.nodename in child: zyn
  uts.nodename in parent: kevin-pc
  ```

  和创建大部分其他namespace相同，创建UTS namespace需要管理员权限，这时就要尽力避免`set-user-ID`应用程序误操作而导致系统出现意外的hostname；

- `/proc/PID/ns`目录：

  每个进程都拥有一个对应的`/proc/PID/ns`目录，该目录为该进程的每个namespace都有对于文件

  ```shell
  root@kevin-pc:/home/kevin/namespace# ls -l /proc/$$/ns           # 这里的$$表示shell的PID
  总用量 0
  lrwxrwxrwx 1 root root 0 2月   8 17:41 cgroup -> 'cgroup:[4026531835]'
  lrwxrwxrwx 1 root root 0 2月   8 17:41 ipc -> 'ipc:[4026531839]'
  lrwxrwxrwx 1 root root 0 2月   8 17:41 mnt -> 'mnt:[4026531840]'
  lrwxrwxrwx 1 root root 0 2月   8 17:41 net -> 'net:[4026532008]'
  lrwxrwxrwx 1 root root 0 2月   8 17:41 pid -> 'pid:[4026531836]'
  lrwxrwxrwx 1 root root 0 2月   8 17:41 pid_for_children -> 'pid:[4026531836]'
  lrwxrwxrwx 1 root root 0 2月   8 17:41 user -> 'user:[4026531837]'
  lrwxrwxrwx 1 root root 0 2月   8 17:41 uts -> 'uts:[4026531838]'
  ```

  root@kevin-pc:/home/kevin# touch ~/uts
  root@kevin-pc:/home/kevin# mount --bind /proc/8312/ns/uts ~/uts我们可以通过`类型:[inode_numbers]`字符串来判断两个进程是否在同一个namespace中，inode_numbers可以通过`stat()`系统调用获得；

  ```shell
  ^Z																												# stop parent and child
  [1]+  已停止               ./demo_uts_namespaces zyn				 
  root@kevin-pc:/home/kevin/namespace# jobs -l  			   # show PID of parent process
  [1]+  8311 停止                  ./demo_uts_namespaces zyn
  root@kevin-pc:/home/kevin/namespace# readlink /proc/8311/ns/uts  # show parent UTS namespace
  uts:[4026531838]
  root@kevin-pc:/home/kevin/namespace# readlink /proc/8312/ns/uts  # show child UTS namespace
  uts:[4026532825]
  ```

  从上面我们可以看出，不同进程的`/proc/PID/ns/uts`中的namespace是不同的，说明它们在不同的UTS namespace中；需要特别指出的是，**如果我们打开了这些表示namespace的文件，只要保证文件描述符保证一直打开的状态，那么namespace将会继续存在，即使该namespace中的进程全部终止；而且除了保证文件描述符打开之外，还可以通过绑定挂载到文件系统中特定的挂载点上实现**，具体如下所示：

  ```shell
  root@kevin-pc:/home/kevin# touch ~/uts
  root@kevin-pc:/home/kevin# mount --bind /proc/8312/ns/uts ~/uts
  ```

- `setns()`系统调用：`setns()`系统调用的主要作用就是使得调用该函数的进程进入某个已存在的namespace中，`setns()`的原型如下所示：

  ```c
  int setns(int fd, int nstype);
  ```

  具体来说，`setns()`就是将调用进程和之前某个namespace实例解除关联，并将该进程和给定的namespace实例建立关联；fd参数用来指定加入的namespace，它是一个文件描述符表示`/proc/PID/ns`目录下的symbolic links，该文件描述符可以通过直接打开symbolic links或者打开对应的绑定挂载点获得；nstype参数用来检查fd参数对应的namespace是否一致，如果nstype值为0，表示不需要检查；

  接下来我们使用一个程序来完成一个小工具：使用`setns()`和`execve()`系统调用让某个进程加入特定namespace并且在该namespace执行命令；

  ```c
  // join a namespace and execute a command in the namespace
  #define _GNU_SOURCE
  #include <fcntl.h>
  #include <sched.h>
  #include <unistd.h>
  #include <stdlib.h>
  #include <stdio.h>
  
  // a simple error-handling function:
  // print an error message based on the value in 'errno' and terminate the calling process
  #define errExit(msg) do {perror(msg); exit(EXIT_FAILURE); } while (0)
  
  int main(int argc, char *argv[]) {
          int fd;
  
          if (argc < 3) {
              fprintf(stderr, "%s /proc/PID/ns/FILE cmd [arg...]\n", argv[0]);
              exit(EXIT_FAILURE);
          }
          int pid = getpid();
          fd = open(argv[1], O_RDONLY); // get descriptor for namespace
          if (fd == -1) {
              errExit("open");
          }
  
          if (setns(fd, 0) == -1) // join the namespace
              errExit("setns");
  
          execv(argv[2], &argv[2]); // execute a command in namespace
          errExit("execvp");
  }
  ```

  该程序接受两个以上的参数，第一个参数是namespace对应的symbolic link路径或者绑定symbolic link的挂载文件，其他的参数是要在该namespace中执行的程序名称，以及该程序可选的命令行参数；

  接下来，我们将详细介绍代码中的重要部分：

  ```c
  fd = open(argv[1], O_RDONLY); 			// get descriptor for namespace
  setns(fd, 0); 												  // join that namespace
  execvp(argv[2], &argv[2]);					 // execute a command in namespace
  ```

  上面的代码中，第一步是使用`open()`函数打开namespace对应的文件描述符，然后再使用`setns()`函数将当前进程加入文件描述符对应的namespace中，最后在该namespace执行命令行；

  现在我们来编译代码：

  ```shell
  gcc -o ns_exec ns_exec.c
  ```

  执行程序：

  ```shell
  ./ns_exec ~/uts /bin/bash                                       # ~/uts is bound to /proc/8312/ns/uts
  ```

  在上面我们使用`/proc/8312/ns/uts`对应的挂载文件表示namespace，并且执行了shell命令行，接下来我们验证该shell下的UTS namespace是否和`demo_uts_namespace`子进程创建的UTS namespace相同：

  ```shell
  root@zyn:/home/kevin/namespace# hostname
  zyn
  root@zyn:/home/kevin/namespace# readlink /proc/8312/ns/uts
  uts:[4026532825]
  root@zyn:/home/kevin/namespace# readlink /proc/$$/ns/uts
  uts:[4026532825]
  ```

  从结果可以看出，它们的UTS namespace相同，说明使用`setns()`将`ns_exec`创建的进程加入到由`demo_uts_namespace`子进程创建的UTS namespace中；

- `unshare()`系统调用：

  `unshare()`系统调用主要提供类似与`clone()`的功能，但是该函数会根据CLONE_NEW*创建一个新namespace并且把调用进程变成该namespace的一部分，**和clone()最大的不同就是创建namespace的同时，不创建新的进程或线程**，下面是`unshare()`的原型：

  ```c
  int unshare(int flags);
  ```

  flags参数用于指定创建namespace的类型；

  接下来通过一个例子来介绍`unshare()`的使用：

  ```c
  // a simple implementation of the unshare(1) command: unshare namespaces and execute a command
  
  #define _GNU_SOURCE
  #include <sched.h>
  #include <unistd.h>
  #include <stdlib.h>
  #include <stdio.h>
  
  // a simple error-handling function:
  // print an error message based on the value in 'errno' and terminate the calling process
  #define errExit(msg) do { perror(msg); exit(EXIT_FAILURE); } while (0)
  
  static void usage(char *pname) {
      fprintf(stderr, "Usage: %s [options] program [arg...]\n", pname);
      fprintf(stderr, "Options can be:\n");
      fprintf(stderr, " -i unshare IPC namespace\n");
      fprintf(stderr, " -m unshare mount namespace\n");
      fprintf(stderr, " -n unshare network namespace\n");
      fprintf(stderr, " -p unshare PID namespace\n");
      fprintf(stderr, " -u unshare UTS namespace\n");
      fprintf(stderr, " -U unshare user namespace\n");
      exit(EXIT_FAILURE);
  }
  
  int main (int argc, char *argv[]) {
      int flags, opt;
      flags = 0;
      while ((opt = getopt(argc, argv, "imnpuU")) != -1) {
          switch (opt) {
              case 'i': flags |= CLONE_NEWIPC;    break;
              case 'm': flags |= CLONE_NEWNS;     break;
              case 'n': flags |= CLONE_NEWNET;    break;
              case 'p': flags |= CLONE_NEWPID;    break;
              case 'u': flags |= CLONE_NEWUTS;    break;
              case 'U': flags |= CLONE_NEWUSER;   break;
              default: usage(argv[0]);
          }
      }
  
      if (optind >= argc)
          usage(argv[0]);
      if (unshare(flags) == -1)
          errExit("unshare");
  
      execvp(argv[optind], &argv[optind]);
      errExit("execvp");
  }
  ```

  上面代码中最重要的部分就是：

  ```c
  // code to initialize 'flags' according to command-line options omitted
  unshare(flags);
  // now execute 'program' with 'arguments'; 'optind' is the index of the next command-line argument after options
  execvp(argv[optind], &argv[optind]);
  ```

  `unshare()`函数会根据flags参数创建对应的namespace，并且将调用该函数的进程加入新创建的namespace中；

  现在我们来编译程序：

  ```shell
  gcc -o unshare unshare.c
  ```

  接着来执行程序：

  ```shell
  root@kevin-pc:/home/kevin/namespace# echo $$
  6537
  root@kevin-pc:/home/kevin/namespace# cat /proc/6537/mounts | grep mq
  mqueue /dev/mqueue mqueue rw,relatime 0 0
  root@kevin-pc:/home/kevin/namespace# readlink /proc/6537/ns/mnt
  mnt:[4026531840]
  root@kevin-pc:/home/kevin/namespace# ./unshare -m /bin/bash
  root@kevin-pc:/home/kevin/namespace# readlink /proc/$$/ns/mnt
  mnt:[4026532824]
  ```

  从上面的readlink的结果可以看出，两个shell的mount namespace是不同的，说明`unshare()`将该进程加入到新的namespace中；
