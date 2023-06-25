# runc用到的技术小结

## 1.XDG

XDG是linux中的一种技术，该规范定义了一套指向应用程序的环境变量，这些变量指明的就是这些程序应该存储的基准目录。而变量的具体值取决于用户，若用户未指定，将由程序本身指向一个默认目录，该默认目录也应该遵从标准，而不是用户主目录。[^1]

这段话挺绕的，大概看起来就是应用程序应该基于XDG指定的一系列环境变量来存储临时文件、下载的内容等。在runc中使用了环境变量\$XDG_RUNTIME_DIR。而\$XDG_RUNTIME_DIR定义了应存储用户特定的非重要性运行时文件和一些其他文件对象。[^1]

在我的服务器上，$XDG_RUNTIME_DIR的值为：

```
[karkli@VM-12-11-opencloudos bundle]$ echo $XDG_RUNTIME_DIR
/run/user/1001
```

网上对于/run目录的解释为：是运行时需要的，重启时应该抛弃的内容。有点像tmpfs的感觉。并且这里面的内容不一定会被交换到磁盘（如果是重启就丢失的话，除非内存不足，否则大概率不会被换页到内存里）。[^2]

/run其实是/var/run的别名，它在定义上就是tmpfs，而/run/lock也是/var/lock mount来的，这已经是linux的一种标准了。[^3]

runc会将自己复制到$XDG_RUNTIME_DIR/runc下，并且给自己设定os.ModeSticky标志位。什么是ModeSticky？ModeSticky就是设置了该标志位的用户才可以删除该文件，保证了runc不会被其他的用户误删掉。[^4]

[^1]: https://winddoing.github.io/post/ef694e1f.html 
[^2]: https://www.zhihu.com/question/20262336 
[^3]: https://lwn.net/Articles/436012/
[^4]: https://blog.csdn.net/weixin_31090341/article/details/116544073

## 2. github.com/urfave/cli库

### 2.1 cli库构建主程序

cli这个库是用于构建一个命令行应用的，它封装了很多细节，使得你可以定制你的子命令、命令行参数、帮助等一系列信息。非常好用。

首先通过NewApp函数生成一个*cli.App对象：

```go
app := cli.NewApp()
```

然后就可以配置它的Name、Usage、Version参数，这些参数都是string，可高度定制化。

Commands参数是制定子命令的，对应的数据结构是cli.Command。

Flags参数是定制命令行参数的，对应的数据结构是cli.Flag。可以通过看flag_xxx.go看有多少种可以指定的参数类型，比如bool,duration,int32等。一般无论是什么Flag，设定Name、Usage和Value一般就可以了。Name可以用逗号分隔，代表全称和缩写都可以作为参数，比如：

```go
cli.StringFlag{
    Name:  "bundle, b",
    Value: "",
    Usage: `path to the root of the bundle directory, defaults to the current directory`,
}
```

Value是不填的时候的默认值，Usage属于帮助文档。

Before参数是一个类型为func (context *cli.Context) error的函数，Before会在Context就绪后，任意子命令运行前运行，如果Before本身抛出了错误，将不会运行任何子命令。

ErrWriter参数是一个io.Writer，它允许应用程序自行重定向错误输出流，你可以将其输出到文件里、socket里之类的，如果不设定，应该就是os.Stderr。

最后使用该命令运行程序：

```go
if err := app.Run(os.Args); err != nil {
    fatal(err)
}
```

### 2.2 cli库构建子命令

cli库构建子命令的时候，需要填充cli.Command，Name、Usage、ArgsUsage、Description参数：

```go
	Name:  "create",
	Usage: "create a container",
	ArgsUsage: `<container-id>

Where "<container-id>" is your name for the instance of the container that you
are starting. The name you provide for the container instance must be unique on
your host.`,
	Description: `The create command creates an instance of a container for a bundle. The bundle
is a directory with a specification file named "` + specConfig + `" and a root
filesystem.

The specification file includes an args parameter. The args parameter is used
to specify command(s) that get run when the container is started. To change the
command(s) that get executed on start, edit the args parameter of the spec. See
"runc spec --help" for more explanation.`,
```

然后你可以通过在Action中设置产生err时调用cli的ShowCommandHelp函数格式化并输出这些参数：

```
NAME:
   runc create - create a container

USAGE:
   runc create [command options] <container-id>

Where "<container-id>" is your name for the instance of the container that you
are starting. The name you provide for the container instance must be unique on
your host.

DESCRIPTION:
   The create command creates an instance of a container for a bundle. The bundle
is a directory with a specification file named "config.json" and a root
filesystem.

The specification file includes an args parameter. The args parameter is used
to specify command(s) that get run when the container is started. To change the
command(s) that get executed on start, edit the args parameter of the spec. See
"runc spec --help" for more explanation.
```

同样地，子命令也可以设置Flags参数，和主程序的Flags是一样的效果。

Action是执行子命令时会被触发的函数，如果Action返回了error，就会输出这个error到ErrWriter。

## 3.seccomp库

seccomp，全称叫secure computing mode（安全计算模式），是linux内核的一个功能，用于限制系统调用的调用。seccomp是一种沙箱功能，可以通过过滤的方式写程序，从而有条件地拒绝某些系统调用。[^5]

seccomp可以通过prctl（定义在sys/prctl.h中）或者seccomp族函数（定义在seccomp.h中）进行启用，seccomp有两种模式：SECCOMP_MODE_STRICT和SECCOMP_MODE_FILTER，前者只可以调用read、write、_exit和sigreturn四个系统调用，后者可以通过BPF模式来写程序裁决该系统调用是否可被执行。[^6][^7][^11]

一个严格模式下的例子：

```c
#include <stdio.h>
#include <sys/prctl.h>
#include <unistd.h>
#include <linux/seccomp.h>

int main() {
    // MODE_STRICT会禁止所有在白名单以外的系统调用
    prctl(PR_SET_SECCOMP,SECCOMP_MODE_STRICT);
    char *buf = "hello, world\n";
    write(0,buf,14);
    return 0;
}
```

这段代码执行会SIGKILL，但我不知道是什么系统调用使它SIGKILL了。用strace看了一下，原来是return 0;这句是使用的exit_group系统调用而不是exit系统调用，exit_group不在白名单内，所以会被kill掉。

**SECCOMP_MODE_STRICT一度被linus希望从内核中删除。**

但是bpf模式太过麻烦，跟写汇编的效率差不多，于是seccomp官方推出了libseccomp，用于高阶seccomp编程[^8]。同时也有golang对应的封装版本，以支持云计算应用使用seccomp进行安全计算[^9]。

这是一个例子：

```c
#include <unistd.h>
#include <seccomp.h>
#include <linux/seccomp.h>

int main()
{
    // 黑名单机制，拒绝所有execve系统调用
    // 使用seccomp的高阶封装函数使得不用写BPF包规则
    // 编译时加上-lseccomp
    scmp_filter_ctx ctx;
    ctx = seccomp_init(SCMP_ACT_ALLOW);
    seccomp_rule_add(ctx, SCMP_ACT_KILL, SCMP_SYS(execve), 0);
    seccomp_load(ctx);
    char *str = "/bin/sh";
    write(1, "hello world\n", sizeof("hello world\n"));
    execve(str, NULL, NULL);
    return 0;
}
```

首先使用seccomp_init来初始化，SCMP_ACT_ALLOW是指没有匹配规则时，允许所有的系统调用。你也可以使用SCMP_ACT_KILL，来在没有匹配到规则时，拒绝所有的系统调用。总之，SCMP_ACT_ALLOW是黑名单机制，而SCMP_ACT_KILL是白名单机制。

通过使用seccomp_rule_add来编写规则，来确定是放过还是拒绝某个系统调用：

![image-20230606235750075](pic/image-20230606235750075.png)

arg_cnt是指需要判断的系统调用的参数的个数，0代表不校验参数，一律放过/拒绝。否则判断前arg_cnt个参数是否符合规则。

比如以下这段，就是使得dup2(1,2)可以通过，而其他的dup2系统调用都会被拒绝[^10]：

```c
seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(dup2), 2,
        SCMP_A0(SCMP_CMP_EQ, 1),
        SCMP_A1(SCMP_CMP_EQ, 2));
```

最后需要用seccomp_load才能使接下来的代码都处于seccomp的保护之下[^6][^10]。

[^5]: https://en.wikipedia.org/wiki/Seccomp
[^6]: https://www.anquanke.com/post/id/208364
[^7]: http://just4coding.com/2022/04/03/seccomp/
[^8]: https://github.com/seccomp/libseccomp
[^9]: https://github.com/seccomp/libseccomp-golang
[^10]: http://just4coding.com/2022/04/03/seccomp/
[^11]: https://www.kernel.org/doc/Documentation/prctl/seccomp_filter.txt

## 4.NOTIFY_SOCKET（资料太少，待完善）
NOTIFY_SOCKET是linux系统中，systemd用于接收服务状态变更的通知的套接字。通过给NOTIFY_SOCKET发送字符串（或者调用sd_notify函数），就可以实现对服务的状态进行变更（比如将服务从已停止变为已启动）。

## 5.linux的身份标识符
Linux中的UID、GID和EUID是三个不同的用户标识符，它们分别代表用户 ID、组 ID 和effective user ID。以下是对这三个标识符的简要概述：

- 用户标识符（UID）：每个用户在系统中都有一个唯一的 UID，它用于标识用户的身份。UID 通常由用户的登录名确定，但也可以由其他方式分配。UID 被用作许多系统资源的访问控制列表（ACL）条目的键，例如文件、目录和设备等。

- 组标识符（GID）：同样，每个用户也有一个或多个组，每个组也有一个唯一的 GID。GID 通常与用户的主组关联，但也可以由其他方式分配。GID 被用作许多系统资源的 ACL 条目的键，例如文件、目录和设备等。

- effective user identifier（EUID）：effective user identifier 是当前执行进程的实际用户标识符。它是在进程创建时由 init 进程继承的，并随后在进程进行任何有效用户身份转换时更新。换句话说，EUID 表示正在执行进程所表示的用户的实际身份。

总之，UID、GID 和 EUID 分别代表用户的身份、用户所属的组和当前执行进程的实际用户身份。它们是 Linux 系统中用于实现用户和组访问控制的重要机制。

如果使用了sudo，那么进程的UID和EUID都会变成0，代表当前是root用户运行的。
```go
package main

import (
	"fmt"
	"os"
)

func main() {
	uid := os.Getuid()
	fmt.Printf("uid: %d\n", uid)
	euid := os.Geteuid()
	fmt.Printf("euid: %d\n", euid)
	if uid == 0 {
		fmt.Println("Are you root?")
	}
	if euid == 0 && uid != 0 {
		fmt.Println("Is this program has setuid bit?")
	}
}
```
如果需要使得UID和EUID不同，我们需要做以下操作：

1. 切换至root用户
2. 编译上述代码，并且在编译后，使用ll -h确定编译后的程序的所属者和所属用户组都是root。否则
3. 执行 `chmod u+s <your-program>`，这里chmod u+s命令是用来设置文件的setuid位的。这个命令的含义是“给文件的所有者（user）添加setuid权限”。
4. 切换回普通用户，执行该程序。

在我的机器上，直接执行会得到以下的结果：
```
[karkli@VM-12-11-opencloudos test-euid]$ ./test-euid 
uid: 1001
euid: 0
Is this program has setuid bit?
```
可以看到，euid此时被设置为了0（因为程序会以拥有者root的身份设置euid）。

如果加上sudo，那么uid就会变成root的uid（即0）：
```
[karkli@VM-12-11-opencloudos test-euid]$ sudo ./test-euid 
uid: 0
euid: 0
Are you root?
```

## 6.cgroups技术
cgroups技术是linux内核提供的一种功能，它是实现容器的核心，它实现了运行资源（如CPU、内存、磁盘I/O等）的限制与隔离，使一组进程（通常被叫作进程组）都只能使用宿主机一部分规定好的资源限额，从而达到资源控制的效果。[^1]

而每种资源的控制都是实现内核定义的cgroupfs这种伪文件系统来达成的，这些子系统通常都由内核直接实现。[^2]

cgroups是采用层级组织的方式进行管理的，也就是说，cgroup之间和进程之间一样，也存在着父子关系的组织层级。而cgroup的管理对象是一个进程组，也就是说不仅是一个进程可以受到cgroup的资源约束，多个进程都可以在一个cgroup下同时被约束，这就给容器这种需要在一个资源限制下执行多个进程的技术提供了可能性。[^2]

cgroups技术在2.6.24版本被引入，但经过这么多年的发展后，cgroups子系统逐渐变得无序，难以管理，因此在4.5版本引入了新的cgroup技术，后者被称之为cgroups v2，相对应地，在2.6.24版本引入的cgroups被称为cgroups v1。[^2]

## 6.1 cgroups v1



[^1]: https://zh.wikipedia.org/wiki/Cgroups
[^2]: https://man7.org/linux/man-pages/man7/cgroups.7.html