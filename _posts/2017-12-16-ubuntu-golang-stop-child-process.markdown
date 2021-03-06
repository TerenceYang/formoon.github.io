---
layout:         post
title:          golang子进程的启动和停止
subtitle:       mac与linux的区别
card-image:     https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1513447113487&di=c73f02fddca8c7917d825fd8f90aa69e&imgtype=0&src=http%3A%2F%2Fimgsrc.baidu.com%2Fimgad%2Fpic%2Fitem%2Fd000baa1cd11728b7e303b08c3fcc3cec3fd2cfe.jpg
date:           2017-12-16
tags:           linux mac golang
post-card-type: image
---
![](https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1513447113487&di=c73f02fddca8c7917d825fd8f90aa69e&imgtype=0&src=http%3A%2F%2Fimgsrc.baidu.com%2Fimgad%2Fpic%2Fitem%2Fd000baa1cd11728b7e303b08c3fcc3cec3fd2cfe.jpg)
今天接到一个任务是将原来运行在mac的应用移植到linux，原因当然是因为客户那边当前是linux环境，也不想再采购mac电脑。  
通常来说，这个工作并不难，因为我选用的服务器端技术是c或者golang,这两种技术具有很好的可移植性，而且大多是重新编译即可运行，所以接到任务的开始并没有把这个当一回事。  
跟想象中的也差不多，搭建好linux测试服务器，在mac上把运行很久的应用重新交叉编译了一遍，部署到linux实验环境，启动、测试，看起来一切正常。准备打包交活，这时候发现一个问题，程序无法终止。  
简单调试后就找到了原因，在系统中启动的子进程，发出终止信号之后居然仍在运行，导致父进程也一直无法退出，尴尬了。  
列一下采用的代码(代码为简化版仅供示例）：  
```go
func startChild1() {
	cmd := exec.Command("/bin/sh", "-c", "sleep 1000")
	time.AfterFunc(10*time.Second, func() {
		fmt.Println("PID1=", cmd.Process.Pid)
		syscall.Kill(-cmd.Process.Pid, syscall.SIGQUIT)
		fmt.Println("killed")
	})
	fmt.Println("begin run")
	cmd.Run()
}
```
示例代码首先启动一个sleep的子进程，表示某个子业务开始工作，然后延时10秒钟之后，把这个子进程杀死。这段代码启动子进程和关闭子进程在mac电脑的原有系统上工作都很正常，但是到了linux，启动子进程仍然没有问题，关闭子进程不成功。  
检查了一下在linux的工作过程，发现启动子进程之后，实际上是启动了两个进程，一个进程是`/bin/sh`,随后sh又启动了一个子进程自身的子进程sleep。而发出退出命令的时候，只有sh退出了，sleep进程仍然继续运行。对比同样的mac电脑上，sh进程是没有出现的，只有一个sleep进程，所以发出退出命令的时候，sleep正常关闭，系统表现正常。  
使用/bin/sh来启动另外的命令行程序是有原因的，这源于golang本身的设计，golang的exec.Command,后面第一个参数是命令行程序本身，之后的每一个exec.Command参数，都代表命令行程序的一个参数,而不是我们常用的，命令行程序路径和参数都可以写在一个字符串，用空格隔开即可。所以有的时候我们是为了省事，也有的时候是顺手移植了别的语言的代码，就使用/bin/sh来启动需要的命令行程序，就如同上面示例代码一样，这样情况下，除了-c参数要单独占用一个字符串，我们原本要启动的字符串程序及其参数，就可以如同常见语言处理方式那样，放在一个字符串了。  
我们可以尝试一下这个代码：  
```go
func startChild2() {
	cmd := exec.Command("sleep", "1000")
	time.AfterFunc(10*time.Second, func() {
		fmt.Println("PID2=", cmd.Process.Pid)
		syscall.Kill(-cmd.Process.Pid, syscall.SIGQUIT)
		fmt.Println("killed")
	})
	fmt.Println("begin run")
	cmd.Run()
}
```
测试一下，这段代码因为没有经过/bin/sh程序，在linux上也只有sleep这一个进程被建立，直接向其发出退出指令是可以正常工作的。这从进程的观察中及实验的结果中，都可以证实我们的判断。 
知道了原因，处理起来也很容易，一是把程序改成类似上面这样的方式启动进程。另外一个办法则是直接为/bin/sh及我们的命令行进程建立一个进程组，这样最后发出的指令退出这个进程组，同样可以同时退出/bin/sh及sleep两个进程，达到我们的需求。写代码测试一下：  
```go
func startChild3() {
	cmd := exec.Command("/bin/sh", "-c", "sleep 1000")
	cmd.SysProcAttr = &syscall.SysProcAttr{Setpgid: true}
	time.AfterFunc(10*time.Second, func() {
		fmt.Println("PID3=", cmd.Process.Pid)
		syscall.Kill(-cmd.Process.Pid, syscall.SIGQUIT)
		fmt.Println("killed")
	})
	fmt.Println("begin run")
	cmd.Run()
}
```
经过实际测试，这段代码在不改变原有的命令行参数传递习惯的基础上，可以正常在linux及mac电脑顺利执行。  

最后再说一下命令`cmd.Process.Signal`,golang文档上说的很清楚，这是向进程发送消息信号，比如同样的syscall.SIGQUIT,这也是告诉子进程退出的意思。所以大多的应用中，我们希望一个进程退出，直接用：
```go
		cmd.Process.Signal(syscall.SIGQUIT)
```
也是可以正常执行的，但对于我们上面说的情况，如果先使用/bin/sh启动了另外一个子进程，这种方法就无效了（指在linux无效，mac测试是一样可以用的,关键区别同样是在mac,/bin/sh进程不会保留并等待我们启动的子进程退出，所以退出消息可以正常的发送到正常的子进程）。所以为了跨平台的通用性，建议还是使用Process.Kill或者syscall.Kill来杀死子进程。

本文参考及引用了以下链接，在此致谢：  
<https://studygolang.com/articles/10083>


