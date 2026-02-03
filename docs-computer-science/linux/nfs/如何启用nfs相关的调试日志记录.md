# 如何启用NFS相关的调试日志记录
# 如何启用NFS相关的调试日志记录

> https://blog.csdn.net/QTM_Gitee/article/details/143935475

## 场景

NFS服务器或NFS客户端不能正常工作。日志并没有提供任何关于什么地方出了问题的细节。

### 解决方法

可以为NFS相关的功能打开额外的调试日志记录。但是，需要注意的是，日志记录非常冗长且非常神秘。除了开发人员，它可能对任何人都没有帮助。一个更好的方法（比这里的方法）可能是通过网络数据包捕获（即tcpdump）来分析问题。当然，这两种方法（调试日志或包捕获）都需要一定的技能和经验。如果获得这些调试信息的目的是将其提供给其他人进行分析，那么可能需要该人提前提供有关针对您的特定症状使用何种故障排除和数据收集方法的输入。如果在没有别人输入的情况下为他们收集数据，可能只是在浪费时间。

如果要收集NFS问题的调试日志，可以使用以下方法：

1. 设置环境，以便可以立即重现有问题的行为。最好在直接重现问题并且不需要读取和分析其他分散注意力的日志时启用额外的调试。还要注意，调试日志记录可能非常密集和冗长，可能会降 低系统的性能。让这些调试方法长时间处于开启状态是不太可取的。所以基本步骤应该是：

   a. 启用调试。

   b. 及时复现问题。

   c. 禁用调试。

2. rpcdebug是一个命令行工具，它可以启用或禁用与 NFS 相关的各种模块的调试功能，以及这些模块中的各种类别的调试日志。 -m标志标识要激活调试的模块， -s 选项标识要设置的调试标志,-c 选项标识要清除的调试标志。

要查看可用的 rpcdebug 模块，请运行：

```
# rpcdebug -vh

usage: rpcdebug [-v] [-h] [-m module] [-s flags...|-c flags...]
       set or cancel debug flags.
Module     Valid flags
rpc        xprt call debug nfs auth bind sched trans svcsock svcdsp misc cache all
nfs        vfs dircache lookupcache pagecache proc xdr file root callback client mount fscache pnfs pnfs_ld state all
nfsd       sock fh export svc proc fileop auth repcache xdr lockd all
nlm        svc client clntlock svclock monitor clntsubs svcsubs hostcache xdr all
以下是可以使用 rpcdebug 命令为其设置内核调试标志的模块列表。
```

| 命令 | 描述                   |
| ---- | ---------------------- |
| nfs  | NFS客户端              |
| nfsd | NFS服务器              |
| nlm  | 网络锁管理器协议 (NLM) |
| rpc  | 远程过程调用           |

## 启用调试

注意，nfs 服务器和nfs 客户端的区别分别是nfsd vs nfs。

1. 要生成NFS 客户端功能的调试日志。

   ```bash
   # rpcdebug -m nfs -s all
   ```

2. 要生成NFS 服务器lock的调试日志:

   ```bash
   # rpcdebug -m nfsd -s lockd
   ```

3. 生成NFS 服务器完整功能的调试日志。

    ```bash
    # rpcdebug -m nfsd -s all
    ```

4. 在许多情况下，RPC协议也需要调试;要开启RPC调用调试：

    ```bash
    # rpcdebug -m rpc -s call
    ```

附加的日志信息将出现在/var/log/messages中，或者可以使用dmesg命令查看。

## 禁用调试

要禁用相应的调试选项，请使用相同的命令，但使用-c选项来清除相关标志。

```bash
# rpcdebug -m nfsd -c all
# rpcdebug -m nfs -c all
# rpcdebug -m rpc -c all
```
在非常确定问题所在的情况下（nfs服务器vs nfs客户端），可能不需要在两端启用调试。但是在某些情况下，在nfs客户端系统上启用rpc/nfs，在nfs服务器系统上启用rpc/nfsd可能是有益的。

注意：确保在完成调试后禁用调试。

启用调试后，调试会在日志上创建大量输出，可能会影响系统性能。
