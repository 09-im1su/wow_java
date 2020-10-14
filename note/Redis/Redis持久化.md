# Redis持久化

---

## Redis高可用概述

在Web服务器中，高可用指的是服务器可以正常访问的时间，衡量的标准是多长时间内可以提供正常的服务(99.9%、99.99%、99.999%等)。

在Redis语境中，高可用除了要保证Redis服务器能提供正常服务(如主从复制、快速容灾技术)外，还需要考虑数据容量的扩展，数据安全不丢失等问题。

在Redis中，实现高可用的技术包括：持久化、主从复制、哨兵和集群。

- **持久化**：持久化是最简单的高可用方法，主要的作用是数据备份，即将数据存储在硬盘中，保证数据不会因进程退出而丢失。
- **主从复制**：主从复制是Redis高可用的基础，哨兵和集群都是在复制的基础上实现高可用的。复制主要实现了数据的多机备份，以及对于读操作的负载均衡和简单的故障恢复。
  - 缺陷：故障恢复无法自动化；写操作无法负载均衡；存储能力受到单机的限制
- **哨兵**：哨兵是在复制的基础上实现了自动化的故障恢复。
  - 缺陷：写操作无法负载均衡；存储能力受到单机的限制
- **集群**：通过集群，Redis解决了写操作无法负载均衡、存储能力受单机限制的问题，实现了较为完善的高可用方案。

## Redis持久化概述

持久化的功能：Redis是内存数据库，数据都存储在内存中，为了避免出现进程退出导致数据丢失的情况，需要定期将Redis中的数据以某种形式(数据或命令)从内存保存到硬盘中；当下次Redis重启时，就能利用持久化文件进行数据恢复。并且，为了进行容灾备份，还可以将持久化文件拷贝到远程位置保存起来。

Redis的持久化由RDB持久化和AOF持久化两种模式

- **RDB持久化**：RDB持久化将Redis的数据直接保存到硬盘。
- **AOF持久化**：将每次执行的写命令保存到硬盘(类似于MySQL的binlog)

由于aof持久化的实时性更好，即当进程意外退出的时候丢失的数据更少，因此aof是目前主流的持久化方式，不过rdb持久化仍然有其应用场景。

## RDB持久化

RDB持久化是将当前进程中的数据生成数据快照保存到硬盘中(因此也被称为快照持久化)，保存的文件后缀是 .rdb。当Redis重启时，可以读取快照文件进行数据恢复。

### 触发条件

RDB持久化的触发分为手动触发和自动触发:

- **手动触发**

  - 手动执行save命令和bgsave命令
  - **save命令**
    - save命令会阻塞Redis服务器进程，使得Redis能不能处理任何命令，知道rdb文件创建完毕为止。
  - **bgsave命令**
    - bgsave命令会fork一个子进程，由子进程来负责创建rdb文件，父进程(即Redis主进程)继续处理请求。
    - fork() 是由操作系统提供的函数，作用是创建当前进程的一个副本作为子进程。
  - bgsave命令执行过程中，只有fork子进程时会阻塞服务器，而对于save命令，整个创建rdb文件的过程都会阻塞服务器，因此，save命令已基本被废弃，线上环境要杜绝save的使用。
  - 此外，在自动触发rdb持久化时，Redis也会选择bgsave而不是save来进行持久化。

- **自动触发**

  - **save m n 命令**

    - 自动触发Redis的rdb持久化最常见的情况是在配置文件中配置 save m n ，指定当 n 秒内发生了 m 次数据的变化则触发bgsave进行数据的持久化。

      ~~~ properties
      # Save the DB on disk:
      #
      #   save <seconds> <changes>
      #
      #   Will save the DB if both the given number of seconds and the given
      #   number of write operations against the DB occurred.
      #
      #   In the example below the behaviour will be to save:
      #   after 900 sec (15 min) if at least 1 key changed
      #   after 300 sec (5 min) if at least 10 keys changed
      #   after 60 sec if at least 10000 keys changed
      #
      #   Note: you can disable saving completely by commenting out all "save" lines.
      #
      #   It is also possible to remove all the previously configured save
      #   points by adding a save directive with a single empty string argument
      #   like in the following example:
      #
      #   save ""
      
      save 900 1
      save 300 10
      save 60 10000
      ~~~

      其中 save 900 1 的含义是：自上一次bgsave执行完成时起，经过900秒后，如果Redis数据发生了至少一次变化，则执行bgsave；save 300 10 和 save 60 10000 同理，且当三个save条件中任意一个满足时就会引起 bgsave 的调用。

  - **save m n 实现原理**

    - Redis的 save m n 是通过 ServerCron 函数、dirty 计数器和 lastsave 时间戳来实现的
    - **ServerCron**
      - ServerCron函数是Redis服务器的周期性操作函数，默认每隔100ms执行一次；ServerCron函数会对Redis服务器的状态进行维护，其中一项工作就是检查 save m n 是否满足，如果满足就执行bgsave。
    - **dirty**
      - dirty 计数器是Redis服务器维护的一个状态，记录上一次执行bgsave/save命令后，服务器进行了多少次数据的修改(包括增、删、改)，而当save/bgsave执行完成后，就会将dirty重置为0。
        - dirty 计数器记录的是Redis数据被修改的次数，而不是修改命令的执行次数。例如sadd myset v1 v2 v3 命令会使 dirty的值加3。
    - **lastsave**
      - lastSave 时间戳也是Redis服务器维护的一个状态，记录了上一次成功执行bgsave/save的时间。
    - save m n的执行原理如下：每隔100ms，执行ServerCron函数，在ServerCron函数执行过程中，会遍历 save m n 配置的配置项，只要有一个配置项满足就会执行bgsave；对于每一个 save m n 配置项，只有同时满足下面两个条件时才算满足save m n配置项：
      - 当前时间 - lastSave > m
      - dirty >= n

- **其他自动触发机制**

  - 在主从复制的场景下，如果从节点执行全量复制操作，则主节点会执行 bgsave 命令，并将rdb文件发送给从节点
  - 在执行flushadd命令清空服务器数据时，会自动执行rdb持久化
  - 在执行shutdown命令时，会自动执行rdb持久化

### 执行流程

![](D:\Note\note\Redis\image\持久化\rdb_process.jpg)

![](D:\Note\note\Redis\image\持久化\rdb_process2.jpg)

- Redis父进程首先判断：当前是否有其他子进程正在执行save、bgsave、bgrewriteaof，如果有，则bgsave命令直接返回。bgsave / bgrewriteaof 的子进程不能同时执行，主要是基于性能方面的考虑：两个并发的子进程同时执行大量的磁盘写操作，可能引起严重的性能问题。
- 父进程执行fork操作，创建子进程，这个过程父进程是阻塞的，Redis不能执行来自客户端的命令。
- 父进程fork创建子进程完成后，bgsave命令返回"Background saving started"信息并不在阻塞父进程，父进程即可继续相应客户端的命令。
- 子进程创建rdb文件，根据父进程的内存快照把数据集写入临时快照文件，完成后对原有的rdb文件进行原子替换操作，rdb文件使用二进制压缩存储。
- 子进程发送信号给父进程表示rdb文件生成完成，父进程更新统计信息。
  - **压缩**：Redis默认采用LZF算法对rdb文件进行压缩，虽然压缩耗时但可以大大减小rdb文件的体积，因此压缩是默认开启的，当然也可以通过命令 config set edbcompression no 关闭压缩。需要注意的是，rdb文件的压缩并不是针对整个文件进行的，而是对数据库中的字符串进行的，且只有在字符串达到一定长度(20字节)时才会进行。

### 启动时加载

- RDB文件的载入工作是在Redis服务器启动时自动执行的，并没有专门的命令。但由于AOF的优先级更高，因此，当AOF开启时，Redis会优先载入aof文件来恢复数据；只有当AOF关闭时，才会在Redis服务器启动时检测rdb文件并自动载入。服务器载入rdb文件时会进入阻塞状态，直到载入完成为止。
- Redis载入rdb文件时，会对文件进行检测，若文件损坏，则会在日志中打印错误，Redis启动失败。

## AOF持久化

RDB持久化是将进程数据写入文件，再将文件保存起来用于数据恢复。而AOF持久化(Append Only File持久化)则是将Redis每次执行的写命令记录到单独的日志文件中(类似于MySQL的binlog)；当Redis重启时再次执行aof文件中的命令来恢复数据。

与RDB持久化相比，AOF的实时性更好，因此已成为主流的持久化方案。

### 开启AOF

- Redis服务器默认开启RDB，关闭AOF。要开启AOF，需要在配置文件中配置：appendonly yes

### 执行流程

由于需要记录Redis的每条写命令，因此AOF不需要触发，会自动执行。

**执行流程：**

- 命令追加(append)：将Redis的写命令追加到缓冲区aof_buf；

- 文件写入(write)与文件同步(sync)：根据不同的同步策略将aof_buf中的内容同步到硬盘。

- 文件重写(rewrite)：定期重写aof文件，达到压缩的目的。

  #### 命令追加

  - Redis并不是将每次的写命令直接写到aof文件，而是先写到缓冲区aof_buf的末尾，然后再将缓冲区的数据追加到aof文件中。这样做主要是为了避免每次有写命令都直接写入硬盘，导致硬盘I/O成为Redis负载的瓶颈。
  - 命令追加的格式是Redis命令请求的协议格式，它是一种纯文本格式，具有兼容性好、可读性强、容易处理、操作简单、避免二次开销等特点；在AOF中，除了用于指定数据库的select命令，(如 select 0 为选中 0 号数据库)是由Redis添加的，其他都是客户端发送来的写命令。

  #### 文件写入和文件同步

  - Redis提供了多种AOF缓冲区的同步文件策略，策略涉及到操作系统的write函数和sync函数。

  - **write 与 sync**

    > 为了提高文件写入效率，在现代操作系统中，当用户调用write函数将数据写入到文件中时，操作系统通常会将数据暂存到一个内存缓冲区中，当缓冲区被填满或超过了指定时限后，才真正将缓冲区的数据写入到硬盘里。
    >
    > 这样的操作虽然提高了效率，但也带来了安全问题：如果计算机停机，内存缓冲区中的数据就会丢失。因此，系统同时提供了 fsync、fdatasync 等同步函数，可以强制操作系统立刻将缓冲区中的数据写入到硬盘里，从而确保数据的安全性。

    

  - AOF缓冲区的文件同步策略由参数 appendfsync 控制

    - **always：**命令写入aof_buf后立即调用系统 fsync 函数同步写命令到aof文件中，fsync完成后线程返回。这种情况下，每次由写命令都要同步到aof文件中，数据安全性最高，但硬盘I/O成为性能瓶颈，Redis只能支持大约几百TPS写入，严重降低了Redis 的性能；即便是是哟个固态硬盘，每秒大约也只能处理几万个命令，而且会大大降低固态硬盘的寿命。
    - **no：**命令写入aof文件后调用系统的 write 操作，不对aof文件做fsync同步操作；同步由操作系统负责， 通常同步周期为30秒。这种情况下，文件同步的时间不可控，且缓冲区中堆积的数据会很多，数据安全性无法保证。
    - **everysec：**命令写入aof文件后调用系统 write 操作，write完成后线程返回；fsync同步文件操作由专门的线程每秒调用一次。**everysec算是对前两种策略的折中，是性能和数据安全性的平衡，因此是Redis的默认配置，也是推荐的配置**

  #### 文件重写

  - 随着时间流逝，Redis执行的写命令越来越多，aof文件也会越来越大，过大的aof文件不仅会影响服务器的正常运行，也会导致数据恢复所需要的时间过长。

  - 文件重写是指定期重写aof文件，减小aof文件的体积。需要注意的是，**AOF重写是把Redis进程内的数据转化为写命令，同步到新的aop文件中，不会对旧的aof文件进行任何读取、分享、写入操作，而是通过读取服务器当前的数据库状态来实现**

  - 对于AOF持久化来说，文件重写是强烈推荐的，但并不是必须的；即便没有文件重写，数据也可以被持久化并在Redis启动时导入，因此，在一些情境下会关闭自动的文件重写，然后通过定时任务在每天的某一时刻定时的执行重写。

  - 文件重写之所以能压缩aof文件，原因在于：

    - 过期的数据不在写入文件

    - 无效的命令不在写入文件：例如有些数据被重复设置值(set wowkey v1, set wowkey v2)、有些数据被删除了(sadd wowset v1,  del wowset)等等。

    - 多条命令可以合并为一个：例如sadd wowset v1, sadd wowset v2, sadd wowset v3 可以合并为 sadd wowset v1, v2, v3。不过为了防止单条命令过大造成客户端缓冲区溢出，对于list、set、hash、zset类型的key，并不一定只使用一条命令，而是以某个常量为界将命令拆分为多条。这个常量在 `redis.h/REDIS_AOF_ITEMS_PER_CMD`中定义，不可更改，3.0版本中的值为64。

    - **文件重写的触发**

      - 文件重写的触发分为手动触发和自动触发

      - **手动触发：**直接调用 bgrewriteaof 命令，该命令的执行与 bgsave 类似，都是主进程fork子进程进行具体的操作，且都只在fork时阻塞

      - **自动触发：**根据 auto-aof-rewrite-min-size 和 auto-aof-rewrite-percentage 参数，以及 aof_current_size 和 aof_base_size 状态确定触发时机。

        - **auto-aof-rewrite-min-size：**执行AOF重写时，文件的最小体积，默认值为64MB。

        - **auto-aof-rewrite-percentage：**执行AOF重写时，当前aof文件的大小(即aof_current_xize)和上一次重写时aof文件的大小(aof_base_size)的比值。

          - 其中，参数可以通过 config get 命令查看
        
          ![](D:\Note\note\Redis\image\持久化\aof_auto_rewrite_param.jpg)
        
          - 状态可以通过 info persistence 命令查看(开启AOF后才能看见aof状态信息)
        
          ![](D:\Note\note\Redis\image\持久化\info_persistence.jpg)
        
        - 只有当 auto-aof-rewrite-min-size 和 auto-aof-percentage 两个参数同时满足时，才会自动触发AOF重写，即 bgrewriteaof 操作。
      
    - **文件重写流程**
    
      ![](D:\Note\note\Redis\image\持久化\aof_process.jpg)
    
      > AOF重写由父进程fork子进程进行
      >
      > 重写期间Redis执行的写命令会追加到新的aof文件中，为此Redis引入了aod_rewrite_buf缓存
    
      - **重写流程**
        - Redis父进程首先判断当前是由存在正在执行 bgsave / bgrewriteaof 命令的子进程，若存在 bgrewriteaof 命令直接返回，如果存在 bgsave 命令则等待bgsave命令执行完成后在执行。
        - 父进程执行fork操作创建子进程，此过程中父进程是阻塞的。
        - 父进程fork完成后，bgrewriteaof命令会返回"Background append only file rewrite started"信息并不再阻塞父进程，父进程可以响应其他命令。**Redis的所有写命令依然写入aof缓冲区，并根据appendfsync策略同步到硬盘，保证原有AOF机制的正确。**
        - 由于fork操作使用写时复制技术，子进程只能共享fork操作时的共享数据。**由于父进程仍然在相应命令，因此Redis使用aof重写缓冲区(aof_rewrite_buf)保存这部分数据，防止新的aof文件生成期间丢失这部分数据。也就是说，bgrewriteaof执行期间，Redis的写命令同时追加到 aof_buf 和 aof_rewrite_buf 两个缓冲区。**
        - 子进程根据内存快照，按照命令合并规则写入到新的aof文件。
        - 子进程写完新的aof文件后，向父进程发送完成信号，父进程更新统计信息，具体可以通过info persistence查看。
        - 父进程把aof重写缓冲区的数据写入到新的aof文件中，这样就保证了新生成的aof文件所保存的数据库状态和服务器当前的状态一致。
        - 使用新的aof文件替换老文件，完成aof重写。

### 启动时加载

当AOF开启时，Redis启动时会优先载入AOF文件来恢复数据；只有当AOF关闭时，才会载入rdb文件来恢复数据。

当AOF开启，且aof文件存在时，Redis启动日志如下：

![](D:\Note\note\Redis\image\持久化\aof_start.jpg)

当AOF开启，但aof文件不存在时，即使rdb文件存在也不会加载，Redis启动日志如下：

![](D:\Note\note\Redis\image\持久化\aof_start_without_aof_file.jpg)

- **文件校验**
  - 与载入rdb文件类似，Redis载入aof文件时，会对aof文件进行校验，如果文件损坏，则日志中会打印错误，Redis启动失败。但如果aof文件结尾不完整(机器突然宕机等容易导致文件尾部不完整)，且 aof-load-truncated 参数开启，则日志中会输出警告，Redis忽略掉aof文件的尾部，启动成功。aof-load-trunced 参数是默认开启的。
- **伪客户端**
  - 因为Redis的命令只能在客户端上下文中执行，而载入aof文件时命令是直接从文件中读取的，并不是由客户端发送；因此Redis服务器在载入aof文件之前，会创建一个没有网络连接的客户端，之后由它来执行aof文件中的命令，命令执行的效果与带网络连接的客户端完全一样。

### AOF常用配置

**以下是AOF持久化的常用配置以及默认值：**

> **appendonly  no :** 是否开启AOF持久化
>
> **appendfilename  "appendonly.aof" :** aof文件名
>
> **dir  ./ :** rdb文件和aof文件所在的目录
>
> **appendfsync  everysec :** fsync持久化策略
>
> **no-appendfsync-on-rewrite  no :** AOF持久化期间禁止fsync；如果开启该选项，可以减轻文件重写时CPU和硬盘的负载(尤其是硬盘)，但可能会丢失AOF重写期间的数据，因此需要在负载和安全性之间进行平衡
>
> **auto-aof-rewrite-percentage  100 :** 文件重写触发条件之一  -->  文件重写时，当前aof文件与上一次文件重写的aof文件的比值
>
> **auto-aof-rewrite-min-size   64MB :** 文件重写触发条件之一 --> 文件重写时，当前aof文件的最小体积
>
> **aof-load-truncated  yes :** 如果aof文件结尾损坏，Redis启动时是否仍然加载aof文件

## 持久化方案选择

### RDB与AOF的优缺点

- **RDB持久化**
  - 优点：RDB文件紧凑，体积小，网络传输快，适合全量复制，且恢复速度比AOF快很多。当然，与AOF相比，RDB最明显的优势就是占用空间小，对性能影响较小。
  - 缺点：RDB文件的致命缺点在于其数据快照的持久化方式必然做不到实时的持久化，这样会造成大量的数据丢失，因此AOF持久化成为主流。此外，rdb文件需要满足特定格式，兼容性差。
- **AOF持久化**
  - 与RDB持久化相比，AOF持久化的优势在于支持秒级实时持久化、兼容性好，缺点是文件大、恢复速度慢、对性能影响较大。

### 持久化策略选择

在介绍持久化策略前，需要明白无论是RDB还是AOF，持久化的开启都是要付出性能的代价的：

- 对于RDB持久化，一方面是bgsave在进行fork操作时Redis进程会阻塞，另一方面，子进程向硬盘写数据时也会带来I/O压力。
- 对于AOF持久化，向硬盘写数据的频率大大增加(everysec策略下每秒都向硬盘同步数据)，I/O压力更大，甚至可能造成AOF追加阻塞问题。此外，AOF文件的重写与RDB的bgsave类似，都会调用fork创建子进程从而造成Redis服务器的阻塞和子进程的I/O压力问题。

相对来说，由于AOF向硬盘中写数据的频率更高，因此对Redis主进程性能的影响更大。

在实际的生产环境中，应根据数据量、应用对数据的安全性要求、预算限制等不同情况，选择不同的持久化策略：如完全不适用持久化、使用RDB或AOF其中一种、同时开启RDB和AOF持久化等。此外，持久化的选择必须与Redis的主从策略一起考虑，因为主从复制和持久化同样具备数据备份的功能，而且主机master和从机salve可以独立选择持久化方案。

- 如果Redis的数据完全丢弃也没有关系(如Redis完全用作DB层数据的缓存)，那么无论是单机还是主从架构，都可以不进行任何持久化。

- 在单机环境下(个人开发者情况)，如果可以接受十几分钟或更多的数据丢失，选择RDB对Redis的性能更有利；如果只能接受秒级别的数据丢失，应该选择AOF。

- 但在大多数情况下，我们都会配置主从环境，salve的存在既可以实现数据的热备份，也可以进行读写分离分担Redis的读请求，以及在master宕掉后继续提供服务。

  - 在这种情况下，一种可行的持久化方案是：

    - master：完全关闭持久化(包括RDB和AOF)，这样可以让master的性能达到最佳

    - salve：关闭RDB，开启AOF，并定时对持久化文件进行备份(如备份到其他文件夹，并标记好备份时间)；然后关闭AOF的自动重写，然后添加定时任务，在每天Redis闲置时(如凌晨12点)调用bgrewriteaof

  - 为什么开启了主从复制，可以实现热备份了还要设置持久化呢？因为在一些特殊情况下，主从复制仍然不足以保证数据的安全性，例如：

    - master和salve进程同时停止：如果master和salve在同一栋大楼或同一个机房，那么一次停电事故就可能导致master和salve机器同时关机，Redis进程停止；如果没有持久化，则会面临数据的完全丢失。

    - master误重启：master服务因为故障宕掉了，如果系统中有自动拉起机制(即检测到服务停止后自动重启该服务)将master自动重启，由于没有持久化文件，那么master重启后数据是空的，salve同步master后数据也变成空的；如果master和salve都没有设置持久化，同样会面临数据的完全丢失。

      **即使使用了哨兵进行自动的主从切换，也有可能在哨兵轮询到master之前，master就被自动拉起机制重启了，因此，应该避免"自动拉起机制"和"不做持久化"同时出现**

- 异地容灾：上述的几种持久化策略，针对的都是一般的系统故障，如进程异常退出、宕机、断电等，这些故障不会损坏硬盘。但是对于一些可能导致硬盘损坏的灾难情况，如火灾地震，就需要进行异地灾备。例如对于单机的情形，可以定时将RDB文件或重写后的AOF文件，拷贝到远程机器，如阿里云、AWS等；对于主从的情况，可以定时在master上执行bgsave，然后将RDB文件拷贝到远程机器，或者在slave上执行bgrewriteaof重写aof文件并将aof文件拷贝到远程机器上。一般来说，由于RDB文件小、恢复快，因此灾难恢复通常使用RDB文件；异地备份的频率根据数据安全性的需要及其他条件来确定，最少不要低于一天一次。

### fork阻塞：CPU的阻塞

在Redis的实践中，众多因素限制了Redis单机的内存不能过大，例如：

- 当面对Redis请求的暴增，需要从库扩容时，Redis内存过大会导致扩容时间太长

- 当主机宕机时，切换主机后需要挂载从库，Redis内存过大会导致挂载速度过慢

- 持久化过程中的**fork操作：**

  - fork()是操作系统的一个api，而不是Redis 的api

  - fork()用于创建一个子进程(不是子线程)，fork()出来的进程共享其父类的内存数据。仅仅是共享fork()出子进程那一刻的内存数据，后期主进程修改数据对子进程不可见，同理，子进程修改的数据对主进程也不可见。

    - 例如：A进程fork()出了一个子进程B，那么A进程就成为主进程，这时候主进程和子进程指向同一个内存空间，所以他们的数据是一致的。但是A修改了内存上的一条数据，这时候B是看不到的，A新增一条数据，删除一条数据，B都是看不到的。而且B进程出问题了对A进程没有任何影响，A仍然可以对外提供服务，但是如果主进程挂了，子进程必须跟着一起挂。

  - Redis中的fork()

    - Redis巧妙的运用了fork()。当bgsave执行时，Redis主进程会判断当前是否有fork()出来的子进程，若有则忽略，若没有则会fork()出一个子进程来执行rdb文件持久化的工作，子进程与Redis主进程共享同一块内存空间，所以子进程可以进行rdb文件的持久化工作，而主进程可以继续处理请求，二者相互不影响。

  - 写时复制(CopyOnWrite)

    - 主进程fork()出子进程后，内核把主进程中所有的内存页的权限设置为read-only，然后子进程的地址空间指向主进程，这样就共享了主进程的内存。当其中某个进程写内存时(此时肯定时主进程写，因为子进程只负责rdb文件持久化工作，不参与客户端的请求)，CPU硬件检测到内存页是read-only的，于是触发页异常中断(page-fault)，陷入内核的一个中断例程。
    - 中断例程中，内核就会把触发异常的页复制一份(这里仅仅复制异常的页，就是所修改的那个数据页，而不是内存中的全部数据)，于是主、子进程各自持有独立的一份数据。

    **copyOnWrite：fork()出来的子进程共享主进程的物理空间，当主子进程有内存写入操作时，read-only内存页发生中断，将触发异常的内存页复制一份(其余页还是共享主进程的)**

  - 虽然fork()操作时，子进程不会复制主进程的数据空间，但是会复制内存页表(内存页表相当于内存的索引、目录)(用于将子进程的地址空间指向主进程)；主进程的数据空间越大，内存页表越大，fork()时复制操作耗时也会越多。

  - 在Redis中，无论是RDB持久化的bgsave，还是AOF重写的bgrewriteaof，都需要fork出子进程来进行操作。如果Redis内存过大，会导致fork()操作时复制内存页表耗时过多；而Redis主进程在进行fork操作时，是完全阻塞的，也就意味着无法响应客户端的请求，会造成请求延时过大。

  - 为减轻fork()操作带来的阻塞问题，除了控制Redis单机内存的大小以外，还可以适当放宽AOF重写的触发条件、选用物理机或高效支持fork操作的虚拟化技术等。

### AOF追加阻塞：硬盘的阻塞

- 在AOF持久化中，如果AOF缓冲区的文件同步策略为everysec，则在主进程中，命令写入aof_buf后调用系统的write操作，write操作完成后主线程返回，fsync同步文件操作由专门的文件同步线程每秒调用一次。

- 这种做法的问题在于，如果硬盘负载过高，那么fsync操作用时可能会超过1s；如果Redis主线程持续高速向aof_buf写入命令，硬盘的负载会越来越大，I/O资源消耗更快；如果此时Redis进程异常退出，丢失的数据也会越来越多，可能远超过1s。

- 针对这种情况，Redis的处理策略是：主线程每次执行AOF时，先对比此时与上次成功执行fsync的时间，如果距上次不到2s，主线程就直接返回；如果超过2s，则主线程阻塞知道fsync同步完成。因此，如果系统硬盘负载过大导致fsync速度太慢，会导致Redis主线程阻塞。因此，使用everysec配置，AOF最多可能丢失2s的数据，而不是1s。

- AOF追加阻塞问题定位：

  - 监控info persistence中的aof-delayed-fsync：当AOF发生追加阻塞时(即主线程等待fsync执行完成而阻塞)，该指标累加。

  - AOF阻塞时的Redis日志：

     `Asynchronous AOF fsync is taking too long (disk is busy?). Writing the AOF buffer without waiting for fsync to complete, this may slow down Redis. `

  - 如果AOF追加阻塞频繁发生，说明系统的硬盘负载太大；可以考虑更换I/O速度更快的硬盘，或者通过I/O监控分析工具对系统的I/O负载进行分析，如iostat、iotop、pidstat等。

### info命令与持久化

- **info persistence**

![](D:\Note\note\Redis\image\持久化\info_persistence.jpg)

> **rdb_last_bgsave_staus: ok** 	上次执行bgsave的结果，可以用于发现bgsave错误
>
> **rdb_last_bgsave_time_sec: -1** 	上次执行bgsave的执行时间(单位为s)，可以用于发现bgsave是否耗时过长
>
> **aof_enabled: 1**	是否开启AOF
>
> **aof_last_rewrite_time_sec: 1** 	上次文件重写执行时间(单位为s)，可以用于发现文件重写是否耗时过长
>
> **aof_last_write_status: ok** 	上次bgrewrite执行结果，可以用于发现bgrewrite错误
>
> **aof_buffer_length: 0** 	缓冲区大小
>
> **aof_rewrite_buffer_length: 0** 	重写缓冲区大小
>
> **aof_delayed_fsync: 0** 	AOF追加阻塞统计情况

