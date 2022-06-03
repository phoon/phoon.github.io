# Linux Process Credentials


Process Credentials包含一系列的ID：

- real user ID & real group ID
- effective user ID & effective group ID
- saved set-user-ID & saved set-group-ID
- file-system user ID & file-system group ID
- supplementary group IDs

#### real user ID & real group ID
real user ID和real gourp ID确定了进程所属的用户与组。在登陆过程中（只讨论终端登陆情况），init进程调用getty完成用户名与密码的输入后，login进程会从`/etc/passwd`文件中读取相应用户密码记录的第三字段（UID）和第四字段（GID），并调用`setuid()`设定登陆成功后对应Shell进程的real user ID与real group ID。即real user ID（real group ID）用于标识当前用户（组）是谁。
可使用`logname`命令获取real user ID：
```bash
┌─[peven@Arch] - [~] - [Sat Dec 01, 16:03]
└─[$] <> logname
peven
```
#### effective user ID & effective group ID
一般而言，effective user ID（effective group ID）是与real user ID（real group ID）相同的，但当我们的程序需要额外的权限去访问他们没有权限访问的资源时，他们就需要将自己的user ID或group ID改为一个具有适当权限的ID（effective user/group ID）。这可以通过调用`setuid()`，`setgid()`来实现。
effective user ID 可以通过`whoami`命令来查看：
```bash
┌─[peven@Arch] - [~] - [Sat Dec 01, 16:18]
└─[$] <> whoami
peven
┌─[peven@Arch] - [~] - [Sat Dec 01, 16:18]
└─[$] <> su #切换为root用户
Password: 
[root@Arch peven]# logname #可以看到当前Shell的real user ID仍为peven
peven
[root@Arch peven]# whoami #effctive user ID变成了root
root
```
#### saved set-user-ID & saved set-group-ID
在设计程序时，应当遵守最小权限准则，以减少恶意用户试图欺骗我们的程序以非预期的方式使用其特权而损害安全性的可能性。saved set-user-ID（saved set-group-ID）是与set-user-ID（set-group-ID）程序结合使用的。

>此名称中的`saved`保存的实际是effective user ID（effective group ID）。
>- 若可执行文件的set uid/gid权限位开启（eg. chmod u+s a.out），则将进程的effective user/group ID置为可执行文件的属主。否则，进程的effective user/group ID将保持不变。
>- 无论可执行文件的set /uid/gid权限位是否开启，saved set-user-ID（saved set-group-ID）对effective user/group ID值的复制都会进行。

比如我们有一个程序，其作用是打印real user ID，effective user ID以及saved set-user-ID:
```c
/* res.c */
#define _GNU_SOURCE        
#include <unistd.h>
#include <stdio.h>

int main(void)
{
        uid_t ruid, euid, suid;
        getresuid(&ruid, &euid, &suid);
        printf("real user ID: %d\neffective user ID: %d\nsaved set-user-ID: %d\n", ruid, euid, suid);
        return 0;
}
```
其权限为：
```bash
-rwxr-xr-x 1 peven users 17K Dec  1 17:17 res.o
```

编译运行：
- 在未开启`set uid/gid`权限位时，以普通用户运行：
```bash
real user ID: 1000
effective user ID: 1000
saved set-user-ID: 1000
```
root用户：
```bash
real user ID: 0
effective user ID: 0
saved set-user-ID: 0
```
- 开启`set uid/gid`权限位后（chmod u+s res.o），普通用户：
```bash
real user ID: 1000
effective user ID: 1000
saved set-user-ID: 1000
```
root用户：
```bash
real user ID: 0
effective user ID: 1000
saved set-user-ID: 1000
```
#### file-system user ID & file-system group ID.
file-system user/group ID始见于Linux1.2版本，当时如果某进程甲的effective user ID等同于某进程乙的real user ID或effective user ID，那么甲就可以向乙发送信号。file-system user/group ID就是为了防范这一风险而生的。不过从Linux2.0版本起，Linux开始在信号发送权限方面遵循SUSv3所强制规定的规则，且这些规则不再涉及目标进程的effective user ID。这样看来，file-system user/group ID在今天已经可以废除了。但为了保持软件的兼容性，还是将其保留了下来。
#### supplementary group IDs
supplementary group IDs用于标识进程所属的若干附加的组。supplementary group IDs会配合file-system ID和effective ID来进行进程是否有访问某些资源权限的判定。
