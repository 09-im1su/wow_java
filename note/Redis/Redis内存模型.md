# Redis内存模型

---

## 前言

了解Redis内存模型，对使用Redis有很大帮助

- 估算Redis内存使用量
- 优化内存占用
- 分析解决问题

## 内存统计

- 执行info memory命令(memory为参数，info命令可查看Redis服务器的许多信息，如服务器基本信息、CPU、内存、持久化、客户端连接信息等)，查看Redis的内存的情况

![](D:\Note\note\Redis\image\内存结构\info_memory.jpg)

- 参数介绍
  - **used_memory**：**Redis分配器分配的**内存总量(单位为字节)，包括使用的虚拟内存(即swap)；used_memory_human表示的与used_memory类似，只是为显示更友好，以K为单位显示。
  - **used_memory_rss**：Redis进程占据操作系统的内存(单位为字节)。包括Redis分配器分配的内存以及Redis进程本身运行锁所需要的内存、内存碎片等，但不包括虚拟内存。
  
  - **used_memory可能比used_memory_rss大，也可能比它小**：used_memory是从Redis角度得到的内存占据量，used_memory_rss是从操作系统角度得到的内存占据量。二者之所以不同是因为一方面Redis进程本身运行需要占据内存、Redis产生的内存碎片等，使得前者可能比后者小；另一方面虚拟内存的存在使得前者可能比后者大。
  
  - **mem_fragmentation_ratio**：内存碎片比例(used_memory_rss / used_memory)。
  
    - 在实际应用中，由于Redis存储的数据量很大，此时Redis进程本身所占用的内存与Redis数据量和虚拟内存相比，会显得很小，因此可用mem_fragmentation_ratio来衡量Redis的内存碎片比例。
  
    - mem_fragmentation_ratio一般大于1，且该值越大，说明内存碎片比例越大；当mem_fragmentation_ratio < 1时，说明Redis使用了虚拟内存，由于虚拟内存的媒介是磁盘，比内存速度慢很多，因此，当出现这种情况时需要及时排查，若内存不足需要及时处理，如增加Redis节点、增加Redis服务的内存、优化应用等。
  
  - **mem_allocator**：Redis使用的内存分配器，在编译时指定。可以是libc、jemalloc、tcmalloc，默认是jemalloc。

## 内存划分

### 自身内存

Redis自身进程运行所需要的内存。

### 对象内存(数据内存)

对象内存是Redis内存中占用最大的一块，存储用户的所有数据，还包括慢查询日志等Redis帮我们维护的一些内存数据。

### 缓冲内存

缓冲内存主要包括：客户端缓冲、复制积压缓冲、AOF重写缓冲。

- **客户端缓冲**

  - 客户端缓冲是指所有连到Redis服务器的TCP连接的输入缓冲和输出缓冲。
  - Redis为每个客户端设置了输入缓冲区，用于临时保存客户端发送过来的命令，同时Redis服务端会从输入缓冲区中拉取命令进行执行，输入缓冲区为客户端向服务端发送的命令提供了缓冲功能。输入缓冲区的大小无法控制，每个客户端的输入缓冲区的最大空间为1G，超出则会断开连接。
  - Redis也为每个客户端设置了输出缓冲区，用于临时保存Redis服务端的命令执行结果，同时Redis服务端会从输出缓冲区拉取执行结果返回给客户端，输出缓冲区为服务端向客户端返回结果提供了缓冲。输出缓冲区根据客户端的类型被分为三种：普通客户端、发布订阅客户端、从客户端(Redis复制的slave客户端)，每种客户端的输出规则也不一样。

- **复制积压缓冲**

  Redis在2.8版本后提供了一个可重用的固定大小的缓冲区用于实现增量复制的功能，根据repl-backlog-size参数来控制，默认大小为1MB。

- **AOF重写缓冲**

  - AOF重写缓冲区用于在AOF重写期间保存写命令，所以AOF重写缓冲区的大小取决于AOF的时间以及AOF重写期间的写命令的数量。

  - AOF重写完成后会将AOF重写缓冲区的数据写到AOF文件，从而清空AOF重写缓冲区。

### 内存碎片

- 内存碎片是Redis在分配、回收物理内存过程中产生的。例如，如果对数据的更改频繁，而且数据间的大小相差很大，可能导致Redis释放的空间在物理内存中没有被释放，但Redis又无法有效利用这部分内存，这就形成了内存碎片。
- 内存碎片的产生与对数据进行的操作、数据的特点有关；此外，与使用的内存分配器也有关系。如果内存分配器设计合理，可以尽可能的减少内存碎片的产生。
- 如果Redis服务器中的内存碎片已经很大，可以通过安全重启的方式减小内存碎片。因为重启之后，Redis重新从备份文件中读取数据，在内存中进行排查，为每个数据重新选择合适的内存单元，减小内存碎片。

**used_memory = 自身内存 + 对象内存 + 缓冲内存 + 虚拟内存**

**used_memory_rss = 自身内存 + 对象内存 + 缓冲内存 + 内存碎片**

## 数据存储

### 概述

Redis的数据存储涉及到内存分配器(如jemalloc)、简单动态字符串(SDS)、5种对象类型及其内部编码、redisObject等。

- 执行set hello world时所涉及到的数据模型

  ![](D:\Note\note\Redis\image\内存结构\set_hello_world.jpg)
  - **dictEntry**：Redis是key-value类型的数据库，因此对每个键值都会有一个dictEntry，里面存储了指向key和value的指针，next指针指向下一个dictEntry，与本对象的key-value无关。
  - **key**：图中右上角"hello"，可见Redis中的key不是直接以字符串形式存储，而是存储在SDS结构体中。
  - **value**：图中右下角"world"，可见Redis中的value是存储在redisObject结构体中(由于world是字符串类型，因此存储在SDS中，而redisObject中有指向该SDS的指针)。
  - **redisObject**：Redis中五种数据类型的value都是通过redisObject来存储的。redisObject中的type字段指明了value对象的类型，ptr指针指向该对象所在的地址(图中指向存储了"world"的SDS)。
  - **jemalloc**：无论是dictEntry对象、还是redisObject对象、SDS对象都需要内存分配器分配内存来存储。

### Jemalloc

- Redis在编译时会指定内存分配器，内存分配器可以是libc、jemalloc或者tcmalloc，默认时jemalloc。

- jemalloc作为Redis的默认内存分配器，在处理内存碎片方面做的相对较好。jemalloc在64位系统中将内存空间划分为小(small)、大(large)、巨大(huge)三个范围；每个范围又划分了很多小的内存单位；当Redis存储数据时，会选择大小最合适的内存块进行存储。

  ![](D:\Note\note\Redis\image\内存结构\jemalloc_memory_divide.jpg)

  例如：若需要存储130字节的对象，jemalloc会将其放入160字节的内存单元中，而多余的30字节的内存空间将成为内存碎片。

### RedisObject

Redis中的五种类型的对象都不会直接被存储，都是通过redisObject对象进行存储的。redisObject对象非常重要，Redis对象的类型、内部编码、内存回收、共享对象等功能，都需要redisObject的支持。

~~~ c++
typedef struct redisObject {
　　unsigned type:4;
　　unsigned encoding:4;
　　unsigned lru:REDIS_LRU_BITS; /* lru time (relative to server.lruclock) */
　　int refcount;
　　void *ptr;
} robj;
~~~

![](D:\Note\note\Redis\image\内存结构\redisObject.png)

- **type**

  - type字段表示对象的类型，占4个bit。Redis中有5种数据类型，2^4 = 16 足以表示这些类型。

  ~~~ c
  /* Object types */
  #define REDIS_STRING 0
  #define REDIS_LIST 1
  #define REDIS_SET 2
  #define REDIS_ZSET 3
  #define REDIS_HASH 4
  ~~~

- **encoding**

  - encoding字段表示对象的内部编码方式，占4个bit，同一种数据类型可能有多种不同的编码方式。

  ~~~ c
  /* Objects encoding. Some kind of objects like Strings and Hashes can be
   * internally represented in multiple ways. The 'encoding' field of the object
   * is set to one of this fields for this object. */
  #define OBJ_ENCODING_RAW 0     /* Raw representation */
  #define OBJ_ENCODING_INT 1     /* Encoded as integer */
  #define OBJ_ENCODING_HT 2      /* Encoded as hashtable */
  #define OBJ_ENCODING_ZIPMAP 3  /* Encoded as zipmap */ // 已废弃
  #define OBJ_ENCODING_LINKEDLIST 4 /* Encoded as regular linked list */
  #define OBJ_ENCODING_ZIPLIST 5 /* Encoded as ziplist */
  #define OBJ_ENCODING_INTSET 6  /* Encoded as intset */
  #define OBJ_ENCODING_SKIPLIST 7  /* Encoded as skiplist */
  #define OBJ_ENCODING_EMBSTR 8  /* Embedded sds string encoding */
  #define OBJ_ENCODING_QUICKLIST 9 /* Encoded as linked list of ziplists */
  ~~~

  - 对于Redis支持的每种类型，都有至少两种内部编码(如字符串类型有int、embstr、raw三种编码方式)，通过encoding属性，Redis可以根据不同的使用场景来为对象设置不同的编码，大大提高了Redis的灵活性和效率。如列表类型有压缩列表和双端列表两种编码方式，如果列表中元素较少，Redis倾向于使用压缩列表来进行存储，因为压缩列表占用内存更少，而且比双端链表更快载入；当列表元素较多时，压缩列表就会转化为更适合存储大量数据的双端列表。
  - 通过object  encoding命令可以查看对象采用的编码方式。

  ![](D:\Note\note\Redis\image\内存结构\object_encoding.png)

- **lru**

  - lru字段记录对象最后一次被命令程序访问的时间，占据的bit数根据版本有所不同(4.0版本占据24bit，2.6版本占据22bit)。

  - 通过对比lru时间与当前时间，可以计算出某个对象的空转时间；通过object  idletime命令可以查看该空转时间(单位为秒)，且object  idletime命令不会更新对象的lru时间值。

    ![](D:\Note\note\Redis\image\内存结构\object_idletime.jpg)

  -  lru值除了通过object idletime命令打印之外，还与Redis的内存回收有关系：如果Redis打开了maxmemory选项，且内存回收算法选择的是volatile-lru或allkeys—lru，那么当Redis内存占用超过maxmemory指定的值时，Redis会优先选择空转时间最长的对象进行释放。 

- **refcount**

  - refcount字段记录对象被引用的次数，类型为整型。refcount的作用，主要在于对象的引用计数和内存回收。当创建新对象时，refcount初始化为1；当有新的程序使用到该对象时，refcount加一；当对象不再被一个新程序使用时，refcount减一；当refcount变为0时，对象占用的内存就会被释放。

  - Redis中多次被使用到的对象(refcount > 1)被称为共享对象。Redis为节省内存，当有一些对象出现重复时，新的程序不会创建新的对象，而是仍然使用原来的对象，这个被重复使用的对象就是共享对象。目前共享对象仅支持整型值的字符串对象。

    **共享对象**

    - Redis的共享对象目前只支持整型值的字符串类型对象。之所以如此，是从内存和CPU(时间)的平衡来考虑的：共享对象虽然能够降低内存消耗，但是判断两个对象是否相等需要消耗额外的时间。对于整数值，判断相等的操作复杂度为O(1)；对于普通字符串，判断复杂度为O(n)；而对于列表、哈希、集合和有序集合，判断的复杂度为O(n^2)。

    - 对然共享对象目前只支持整数值的字符串对象，但Redis的五种类型都可能使用共享对象(如哈希、列表等的元素可以是整数值的字符串对象)

    - 目前，Redis服务器在初始化时，会创建10000个字符串对象，值分别为0~9999；当Redis需要使用值为0~9999的字符串对象时，可以直接使用这些共享对象。10000这个数字不能通过配置修改，但可以通过调整参数REDIS_SHARED_INTEGERS(4.0中改为OBJ_SHARED_INTEGERS)的值进行改变。

    - 共享对象的引用次数可以通过object  refcount命令查看。

      ![](D:\Note\note\Redis\image\内存结构\redis_share_integer.jpg)

      图中设置k1的值为9999，使用的是共享对象，因此查看k1的refcount时值为4；而设置k4的值为10000，Redis中没有存储值为10000的共享对象，因此会创建新对象，查看k4的refcount时值为1。

- **ptr**

  - ptr字段是一个指针，指向存储value的具体的数据结构。如例子中的"world"字符串对象即被存储在SDS中，而ptr指针指向了该SDS。

- **总结**
  
  - redisObject中的结构与对象类型、内部编码、内存回收、共享对象都有关系；一个redisObject对象的大小为16字节：4bit (type) + 4bit (encoding) + 24bit (lru) + 4Byte (refcount) + 8Byte (ptr) = 16Byte

### SDS(Simple Dynamic String)

Redis中没有直接使用c中字符串(即以空字符 '\0' 结尾的字符数组)作为默认的字符串表示，而是使用了SDS。SDS是简单动态字符串(Simple Dynamic String)的缩写。

- **SDS结构**

  - SDS在正常维护字符数组之外引入了两个变量free和len，len用于记录当前存储的字符串长度(不包括末尾的 '\0')，free表示SDS结构中剩余可以分配的内存大小

    ~~~ c
    struct sdshdr {
    
        // 记录 buf 数组中已使用字节的数量
        // 等于 SDS 所保存字符串的长度
        int len;
    
        // 记录 buf 数组中未使用字节的数量
        int free;
    
        // 字节数组，用于保存字符串
        char buf[];
    
    };
    ~~~

    ![](D:\Note\note\Redis\image\内存结构\SDS_datastruction.png)

    从上图可以看出，字符数组buf的长度 = free + len + 1 (1 表示字符串结尾的空字符)；所以，一个SDS结构占据的空间为：free长度 + len长度 + buf数组长度 = 4Byte + 4Byte + (free + len + 1)Byte = (free + len + 9)Byte。

- **SDS与c字符串比较**

  SDS在c字符串的基础上加入了free和len字段，带来了很多好处：

  - **获取字符串长度**

    SDS的操作复杂度为O(1)，c字符串的复杂度为O(n)。

  - **避免缓冲区溢出**

    使用c字符串的api时，如果对字符串长度增加(如strcat操作)而忘记重新分配内存，很容易造成缓冲区的溢出

    当SDS的api需要对SDS进行修改时，api会先检查SDS的空间是否满足修改所需的要求，如果不满足的话，api会自动将SDS的空间扩展至执行修改所需的大小，然后才执行实际的修改操作。所以，使用SDS既不需要手动修改SDS的空间大小，也不会出现缓冲区溢出的问题。

  - **内存分配**

    - **空间预分配**

      空间预分配用于优化SDS的字符串增长操作：当SDS的api对一个SDS进行修改，并且需要对SDS进行空间扩展时，程序不仅会为SDS分配修改所必须的内存空间，还会为SDS分配额外的未使用空间。

      额外未使用空间分配数量：

      - 如果对SDS进行修改之后，字符串的长度(即len属性的值)将小于1 MB，那么程序会分配和len属性同样大小的未使用空间，这时SDS的len属性值将和free属性值相等。例如：如果对一个原本len = 5， free = 0的SDS进行修改之后，SDS的len变为13，那么程序也会分配13字节的未使用空间，SDS变为len = 13， free = 13，此时SDS的buf数组的实际长度将变成 13 + 13 + 1 = 27byte。

        ![](D:\Note\note\Redis\image\内存结构\space_preAllocation.png)

      - 如果对SDS进行修改之后，字符串的长度(len的值)将大于1MB，那么程序会分配额外1MB的未使用空间。例如：对一个SDS进行修改后，SDS的len变为30MB，那么程序会分配1MB的未使用空间，SDS的buf数组的实际长度将变为 30MB + 1MB + 1byte。

      通过预分配策略，Redis可以减少连续执行字符串增长操作所需的内存重新分配次数。

    - **惰性空间释放**

      惰性空间释放用于优化SDS的字符串缩减操作：当SDS的api需要缩短SDS保存的字符串时，程序并不立即使用内存重分配来回收缩短后多出来的空间，而是使用free属性将这些字节的数量记录起来，并待将来使用。例如：sdstrim函数接受一个SDS和一个c字符串作为参数，从SDS左右两端分别移除所有在c字符串中出现过的字符

      ~~~ c
      sdstrim(sds, "XY"); // 移除 SDS 字符串中的所有 'X' 和 'Y'
      ~~~

      ![](D:\Note\note\Redis\image\内存结构\inert_space_release.png)

      如图，执行sdstrim后的SDS并没有释放多出来的8byte的空间，而是将这8byte作为未使用的空间保留在SDS中，并有free记录，如果将来要对SDS进行增长操作的话，这些未使用的空间可能就会派上用场。

  - **存取二进制数据**

    SDS可以存储二进制的数据，而c字符串不行。因为c字符串以空字符 '\0' 作为字符串结束的标识，而对于一些二进制文件(如图片等)，内容可能会包含空字符，因此c字符串无法正确存取；而SDS以字符串长度len来标识字符串结束位置，因此可以存取二进制数据。

  - **补充**

    - 由于SDS中的buf仍然使用的是c字符串(即以 '\0' 结尾)，因此SDS可以使用c字符串库中的部分函数，但只有当SDS是用来存储文本数据时才可以，存储二进制数据时不能使用。
    - Redis在存储对象时，一律使用SDS代替c字符串。例如 set hello world 命令，hello和world都是以SDS的形式存储的。而 sadd wowset number1 number2 number3 命令，不论是键("wowset")，还是集合中的元素("number1"、"number2"、"number3")，都是以SDS的形式存储。除了存储对象，SDS还用于存储各种缓冲区。只有在字符串不会改变的情况下，如打印日志时，才会使用c字符串。

## 对象类型及编码

Redis支持5种对象类型，而每种结构都有至少两种内部编码；这样做的好处在于：一方面接口与实现分离，当需要增加内部编码时，可以不影响用户；另一方面是可以根据不同的业务场景选择不同的编码方式，提高效率。

![](D:\Note\note\Redis\image\内存结构\redis_encoding_type.png)

Redis内部编码的转换符合以下规律：编码转换在Redis写入数据时完成，且转换过程不可逆，只能从小内存编码向大内存编码转换。

### 字符串

- **概况**

  - 字符串是最基础的类型，因为所有的键都是字符串类型，且字符串之外的其他几种复杂类型的元素也是字符串。字符串长度不能超过512MB。

- **内部编码**

  字符串类型的内部编码有三种

  - int：8byte的长整型。字符串是整型时，该值使用long整型表示。
  - embstr：长度 <= 39byte 的字符串。embstr与raw都使用redisObject和SDS保存数据，区别在于：embstr的使用只分配一次内存空间(因此redisObject和SDS是连续的)；而raw需要分配两次内存空间(分别为redisObject和SDS分配空间)。因此，与raw相比，embstr的好处在于创建时少分配一次内存空间，删除时少释放一次内存空间，以及对象的所有数据连在一起，寻找方便。而embstr的坏处也很明显，如果字符串长度增加需要重新分配内存，整个redisObject和SDS都要重新分配空间吗，因此redis中的embstr实现为只读。
  - raw：长度 > 39byte 的字符串。

- **编码转换**

  - int -> raw ：当int数据不再是整数或大小超过了long的范围，自动转化为raw。

  - embstr -> raw ：对于embstr，由于其实现是只读的，因此在对embstr对象进行修改时，都会先转化为raw在进行修改。因此，只要是修改embstr对象， 无论修改后的对象是否达到39byte，内部编码都一定转化为了raw。

    ![](D:\Note\note\Redis\image\内存结构\encoding_string.jpg)

- **补充**

  - 3.2版本后，Redis中String内部编码的embstr与raw的分界点由39变为44。

  - 原因：新版本中SDS的结构发生了变化

    ~~~ c
    struct __attribute__ ((__packed__)) sdshdr5 {
        unsigned char flags; /* 3 lsb of type, and 5 msb of string length */
        char buf[];
    };
    struct __attribute__ ((__packed__)) sdshdr8 {
        uint8_t len; /* used */
        uint8_t alloc; /* excluding the header and null terminator */
        unsigned char flags; /* 3 lsb of type, 5 unused bits */
        char buf[];
    };
    struct __attribute__ ((__packed__)) sdshdr16 {
        uint16_t len; /* used */
        uint16_t alloc; /* excluding the header and null terminator */
        unsigned char flags; /* 3 lsb of type, 5 unused bits */
        char buf[];
    };
    struct __attribute__ ((__packed__)) sdshdr32 {
        uint32_t len; /* used */
        uint32_t alloc; /* excluding the header and null terminator */
        unsigned char flags; /* 3 lsb of type, 5 unused bits */
        char buf[];
    };
    struct __attribute__ ((__packed__)) sdshdr64 {
        uint64_t len; /* used */
        uint64_t alloc; /* excluding the header and null terminator */
        unsigned char flags; /* 3 lsb of type, 5 unused bits */
        char buf[];
    ~~~

    - `len`：字符串的长度（实际使用的长度）
    - `alloc`：分配内存的大小
    - `flags`：标志位，低三位表示类型，其余五位未使用
    - `buf`：字符数组

    新版本的SDS中，len字段、alloc字段以及flags字段均只占 1byte，合计占用3byte大小的内存，因此相应的用于存储字符串的内存空间就会变大。当内存分配器分配64byte内存时，字符串最大长度 = 64 - 3 - 1 - 16 = 44 (1 为字符串末尾的结束符，16为redisObject的长度)

### 列表

- **概况**
  
  - 列表可以用来存放多个有序的字符串，每个字符串成为元素。
  - 一个列表可以存放 2^32 - 1 个元素。
  - Redis中的列表是String类型的双向链表，支持两端插入和弹出，并可以获得指定位置(或范围)的元素，可以充当数组、队列、栈等结构。
  
- **内部编码**

  - 3.2版本以前，列表的内部编码可以是压缩列表(ziplist)或双端列表(linkedlist)。

  - **双端列表**

    - 双端列表由一个list结构和多个listNode结构构**成：

      ![](D:\Note\note\Redis\image\内存结构\list_linkedlist.jpg)

      ~~~ c
      typedef struct list{
          // 表头节点
          listNode *head;
          // 表尾节点
          listNode *tail;
          // 链表所包含的节点数量
          unsigned long len;
          // 节点值复制函数
          void *(*dup)(void *ptr);
          // 节点值释放函数
          void *(*free)(void *ptr);
          // 节点值对比函数
          int (*match)(void *ptr,void *key);
      }list;
      ~~~

    - 双端列表同时保存了表头指针和表尾指针，并且每个节点都有指向前驱和后继的指针，且data存的也是一个指针，指向保存的元素的地址(链表里保存的元素是String)。

    - list中保存了列表的长度， dup 、 free 和 match 成员则是用于实现多态链表所需的类型特定函数：
      - dup 函数用于 复制 链表节点所保存的值
      - free 函数用于 释放 链表节点所保存的值
      - match 函数则用于 对比 链表节点所保存的值和另一个输入值 是否相等

    - 链表节点使用 void* 指针来保存节点值， 并且可以通过 list 结构的 dup 、 free 、 match 三个属性为节点值设置类型特定函数， 所以链表可以用于保存各种不同类型的值。

  - **压缩列表**

    - 压缩列表(ziplist)是列表键和哈希键的底层实现之一。当列表或哈希中只包含少量元素，且元素都是小整数值或者长度较短的字符串，那么Redis就会使用压缩列表来作为其键的底层实现。

    - 压缩列表是Redis为节约内存而开发的，是由一系列特殊编码的**连续内存块**组成的顺序型数据结构。

    - 压缩列表有点儿类似数组，通过一片连续的内存空间，来存储数据。不过，它跟数组不同的一点是，它允许存储的数据大小不同。每个节点上增加一个length属性来记录这个节点的长度，这样比较方便地得到下一个节点的位置 。

      ![](D:\Note\note\Redis\image\内存结构\list_ziplist.jpg)

      - `zlbytes`：列表的总长度
      - `zltail`：指向最末元素
      - `zllen`：元素的个数
      - `entry`：元素的内容，里面记录了前一个entry的长度，用于方便双向遍历
      - `zlend`：恒为0xFF，作为ziplist的定界符

  - **3.2版本后，Redis使用快速列表(quicklist作为列表的底层实现**

  - **快速列表**

    - quicklist是ziplist和linkedlist的混合体，它将linkedlist按段切分，每一段使用ziplist来紧凑存储，多个ziplist之间使用双向指针串接起来。

      ![](D:\Note\note\Redis\image\内存结构\list_quicklist.jpg)

      ~~~ c
      // 快速列表
      struct quicklist {
          quicklistNode* head;
          quicklistNode* tail;
          long count; // 元素总数
          int nodes; // ziplist 节点的个数
          int compressDepth; // LZF 算法压缩深度
          ...
      }
      // 快速列表节点
      struct quicklistNode {
          quicklistNode* prev;
          quicklistNode* next;
          ziplist* zl; // 指向压缩列表
          int32 size; // ziplist 的字节总数
          int16 count; // ziplist 中的元素数量
          int2 encoding; // 存储形式 2bit，原生字节数组还是 LZF 压缩存储
          ...
      }
      
      struct ziplist_compressed {
          int32 size;
          byte[] compressed_data;
      }
      
      struct ziplist {
          ...
      }
      ~~~

- **编码转换**

  - 只有同时满足以下两个条件时，才会使用压缩列表：列表中元素数量小于512个；列表中所有字符串对象都不足64byte。如果有一个条件不满足，则使用双端列表；且编码只能由压缩列表转换为双端列表，不能反向转换。

    ![](D:\Note\note\Redis\image\内存结构\encoding_list.jpg)

  - 其中，单个字符串的长度不超过64byte，是为了便于统一分配每个节点的长度；此时的64byte是指字符串的长度，不包括SDS结构，因为压缩列表使用连续、定长内存存储字符串，不需要SDS结构指明长度。

### 哈希

- **概况**

  - 哈希是Redis作为Key-Value数据库所使用的数据结构。
  -  Redis底层也是使用的**散列表**作为字典的实现，解决hash冲突使用的是链表法。 

- **内部编码**

  - 内层的哈希使用的内部编码可以是压缩列表(ziplist)和哈希表(hashtable)两种；Redis外层的哈希则只使用了hashtable。

  - **哈希表**

    - 一个hashtable由一个dic结构，2个dictht结构，1个dictEntry指针数组(成为bucket)和多个dictEntry结构组成。

      ![](D:\Note\note\Redis\image\内存结构\hash_hashtable.jpg)

    - **dictEntry**

      ~~~ c
      typedef struct dictEntry{
          // 键值对中的键
          void *key;
          // 键值对中的值
          union{
              void *val;
              uint64_tu64;
              int64_ts64;
          }v;
          // 指向下一个dictEntry，用于解决哈希冲突问题
          struct dictEntry *next;
      }dictEntry;
      ~~~

      - dictEntry结构用于保存键值对
        - `key`：键值对的键
        - `value`：键值对的值，值为union(共用体)实现，value存储的内容可能是一个指向值的指针，也可能是64为整型值，或无符号64位整型值
        - `next`：指向下一个dictEntry，用于解决哈希冲突问题
      - 在64位系统中，一个dictEntry对象占24字节(key：8byte， value：8byte， next：8byte)。

    - **bucket**

      - bucket是一个指针数组(dictEntry*)，数组的每个元素都是指向dictEntry结构的指针。Redis中的bucket数组的大小计算规则为：大于dictEntry数量的、最小的2^n。例如：若由1000个dictEntry，则bucket大小为1024。

    - **dictht**

      ~~~ c
  typedef struct dictht{
          // 指向bucket的指针
          dictEntry **table;
          // 记录哈希表的大小，即bucket的大小
          unsigned long size;
          // sizemask = size - 1，用于计算元素落在bucket中的位置
          unsigned long sizemask;
          // 记录bucket中已使用的dictEntry的数量
          unsigned long used;
  }dictht;
      ~~~
    
      
    
    - **dict**
    
      - 一般来说，通过dictht和dictEntry结构即可实现普通哈希表的功能，但在Redis的实现中，在dictht结构上又加了一层dict结构，用于适应不同类型的键值对和帮助rehash
      
      ~~~ c
      typedef struct dict{
          // 用于适应不同类型的键值对，丰富哈希表元素类型
          dictType *type;
          // 用于适应不同类型的键值对，丰富哈希表元素类型
          void *privdata;
        // 包含"两个"dictht结构体(哈希表主体)
          dictht ht[2];
          // 用于哈希表扩容或收缩
          int rehashidx;
      } dict;
      
      typedef struct dictType {
          unsigned int (*hashFunction)(const void *key);
          void *(*keyDup)(void *privdata, const void *key);
          void *(*valDup)(void *privdata, const void *obj);
          int (*keyCompare)(void *privdata, const void *key1, const void *key2);
          void (*keyDestructor)(void *privdata, void *key);
          void (*valDestructor)(void *privdata, void *obj);
      } dictType;
      ~~~
      
      - type属性和privdata属性是为了适应不同类型的键值对，用于创建多态字典。
      - ht属性和trehashidx属性主要用于rehash，即当哈希表需要进行扩容或收缩时使用。ht是一个包含**两个**项的数组，每一项都是一个dictht结构。通常情况下，所有的数据都是存放在dict的ht[0]中，ht[1]只在进行rehash的时候使用。dict进行rehash时，将ht[0]中的数据rehash到ht[1]中，然后将ht[1]赋值给ht[0]，并清空ht[1]。
      - 当哈希表中元素太多时，为解决一次扩容耗时过多的情况，可以将扩容操作穿插在插入操作中分批完成。这种操作成为**渐进式rehash**：
        - 当负载因子触达阈值后，为ht[1]分配空间，让字典同时持有ht[0]和ht[1]两个哈希表
        - 将rehashidx置为0，表示rehash工作正式开始
        - 在rehash期间，每次对哈希表执行添加、删除、查找或更新操作时，程序除了执行指定的操作外，还会将ht[0]哈希表中处在rehashidx索引上的所有键值对放到ht[1]上，当此次操作完成以后，程序将rehashidx属性的值增一
        - 随着哈希表操作的不断执行，最终在某个时间点ht[0]中的所有键值对全都被rehash到ht[1]上，此时程序将rehashidx属性的值设置为-1，表示rehash操作已完成
  
- **编码转换**

  - Redis中的hash内部编码既可以使用哈希表也可以使用哈希表，也可以使用压缩列表。

  - 只有同时满足以下两个条件才会使用压缩列表：哈希表中元素数量小于512个；哈希表中所有键值对的键和值字符串长度都小于64byte。若有一个条件不满足，则会使用哈希表进行存储，且编码只能由压缩列表转换为哈希表，不能反向转换。

    ![](D:\Note\note\Redis\image\内存结构\encoding_hash.jpg)

### 集合

- **概况**

  - 集合与列表类似，都是用来保存多个字符串，但集合与列表有两点不同：集合中的元素是无序的(插入顺序与遍历顺序不同)，因此不能通过索引来操作元素；集合中的元素不能有重复。
  - 一个集合最多可以存储 2^32 - 1 个元素。
  - Redis中的集合除了支持常规的增删改查，还支持取交集、并集、差集。

- **内部编码**

  - 当set中添加的元素都是整型且元素数目较少时，set使用intset作为底层数据结构；否则，set使用dict作为底层数据结构（当使用dict时，key为添加的集合元素，value为null）。

  - **intset**

    - intset是由整数组成的集合，实际上，intset是一个由整数组成的有序集合，从而可以使用二分查找快速判断一个元素是否属于这个集合。
    - intset在内存分配上与ziplist类似，是一块连续的内存空间，而且对于大整数和小整数采取不同的编码，尽量对内存的使用进行优化。
    - 与哈希表相比，intset的优势在于集中存储，节省空间。

    ~~~ c
    typedef struct intset {
        uint32_t encoding;
        uint32_t length;
        int8_t contents[];
    } intset;
    
    #define INTSET_ENC_INT16 (sizeof(int16_t))
    #define INTSET_ENC_INT32 (sizeof(int32_t))
    #define INTSET_ENC_INT64 (sizeof(int64_t))
    ~~~

    - `encoding`：数据编码，表示contents中存储的元素的类型(intset中每个数据元素使用几个字节来存储)。

      encoding的取值有三种：

      - INTSET_ENC_INT16  表示每个元素使用2byte来存储
      - INTSET_ENC_INT32  表示每个元素使用4byte来存储
      - INTSET_ENC_INT64  表示每个元素使用8byte来存储

    - `length`：表示intset中元素的个数。encoding属性和length属性构成了intset的头部(header)。

    - `contents`：contents是一个柔性数组，表示intset的header后紧跟着的数据元素。contents数组的总长度(字节总数) = encoding * length。

    - **contents需要单独分配内存空间，这部分内存不包含在intset结构中**

    - 柔性数组在Redis的很多数据结构的定义中出现过，例如SDS、quicklist、skiplist，用于表示一个偏移量。

- **编码转换**

  - 只有同时满足以下两个条件，集合才会使用intset作为内部编码：集合元素数量小于512个；集合中所有元素都是整数值。如果有一个条件不满足，则会使用哈希表作为内部编码进行存储，且不能反向转换。

    ![](D:\Note\note\Redis\image\内存结构\encoding_set.jpg)

### 有序集合

- **概况**
  - 有序集合与集合一样，元素不能重复；但与集合不同的是，有序集合中的元素是有顺序的。
  - 与列表使用索引作为排序依据不同，有序集合为每个元素设置一个分数(score)作为排序依据。
- **内部编码**
  
  - 有序集合的内部编码可以是压缩列表(ziplist)或跳跃表(skiplist)。
  - 如果集合内元素不多且不大，则使用压缩列表ziplist来存储元素，否则使用跳表skiplist。不过由于zset包含了score的排序信息，所以在ziplist内部，是按照score排序递增来存储的，意味着每次插入元素后都要移动之后的元素。
  - **skiplist**
    
    - 跳跃表是一种有序数据结构，是对链表的一个增强，通过在每个节点中维持多个指向其他节点的指针，从而达到快速访问节点的目的。除了跳跃表，实现有序数据结构的另一种典型方式是平衡树；大多数情况下，跳跃表的效率可以和平衡树媲美，但跳跃表实现比平衡树简单很多。跳跃表支持平均O(log n)，最坏O(n)的查找复杂度，并支持顺序操作。
    
    - Redis的跳表实现由zskiplist和zskipNode两个结构组成，前者用于保存跳表信息(如头节点，尾节点，长度等)，后者用于表示跳表节点。
    
      ~~~ c
      #define ZSKIPLIST_MAXLEVEL 32
      #define ZSKIPLIST_P 0.25
      
      typedef struct zskiplistNode {
          robj *obj;
          double score;
          struct zskiplistNode *backward;
          struct zskiplistLevel {
              struct zskiplistNode *forward;
              unsigned int span;
          } level[];
      } zskiplistNode;
      
      typedef struct zskiplist {
          struct zskiplistNode *header, *tail;
          unsigned long length;
          int level;
      } zskiplist;
      ~~~
    
    - 开头定义了两个常量，ZSKIPLIST_MAXLEVEL和ZSKIPLIST_P，分别对应我们前面讲到的skiplist的两个参数：一个是MaxLevel，一个是p。
    
    - zskiplistNode定义了skiplist的节点结构。
    
      - obj字段存放的是节点数据，它的类型是一个string robj。本来一个string robj可能存放的不是sds，而是long型，但zadd命令在将数据插入到skiplist里面之前先进行了解码，所以这里的obj字段里存储的一定是一个sds。这样做的目的应该是为了方便在查找的时候对数据进行字典序的比较，而且，skiplist里的数据部分是数字的可能性也比较小。
      - score字段是数据对应的分数。
      - backward字段是指向链表前一个节点的指针（前向指针）。节点只有1个前向指针，所以只有第1层链表是一个双向链表。
      - level[]存放指向各层链表后一个节点的指针（后向指针）。每层对应1个后向指针，用forward字段表示。另外，每个后向指针还对应了一个span值，它表示当前的指针跨越了多少个节点。span用于计算元素排名(rank)，这正是前面我们提到的Redis对于skiplist所做的一个扩展。需要注意的是，level[]是一个柔性数组，因此它占用的内存不在zskiplistNode结构里面，而需要插入节点的时候单独为它分配。也正因为如此，skiplist的每个节点所包含的指针数目才是不固定的，我们前面分析过的结论——skiplist每个节点包含的指针数目平均为1/(1-p)——才能有意义。
    
    - zskiplist定义了真正的skiplist结构，它包含：
    
      - 头指针header和尾指针tail。
      - 链表长度length，即链表包含的节点总数。注意，新创建的skiplist包含一个空的头指针，这个头指针不包含在length计数中。
      - level表示skiplist的总层数，即所有节点层数的最大值。













