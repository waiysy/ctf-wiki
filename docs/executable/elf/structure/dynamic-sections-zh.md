[EN](./dynamic-sections.md) | [ZH](./dynamic-sections-zh.md)

# Dynamic Sections

## .interp

这个节包含了程序对应的解释器。

参与动态链接的可执行文件会具有一个 PT_INTERP 类型的程序头元素，以便于来加载程序中的段。在 exec (BA_OS) 过程中，系统会从该节中提取解释器的路径，并根据解释器文件的段创建初始时的程序镜像。也就是说，系统并不使用给定的可执行文件的镜像，而会首先为解释器构造独立的内存镜像。因此，解释器需要从系统处获取控制权，然后为应用程序提供执行环境。

解释器可能有两种方式获取控制权。

1. 它可以接收一个指向文件头的文件描述符，以便于读取可执行文件。它可以使用这个文件描述符来读取并将可执行文件的段映射到内存中。

2. 有时候根据可执行文件格式的不同，系统有可能不会把文件描述符给解释器，而是会直接将可执行文件加载到内存中。

解释器本身可能不需要再有一个解释器。解释器本身可能是一个共享目标文件或者是一个可执行文件。

- 共享目标文件（正常情况下）被加载为地址独立的。也就是说，对于不同的进程来说，它的地址会有所不同。系统通过 mmap (KE_OS) 以及一些相关的操作来创建动态段中的内容。因此，共享目标文件的地址通常来说不会和原来的可执行文件的原有地址冲突。
- 可执行文件一般会被加载到固定的地址。系统通过程序头部表的虚拟地址来创建对应的段。因此，一个可执行文件的解释器的虚拟地址可能和第一个可执行文件冲突。解释器有责任来解决相应的冲突。

## .dynamic

如果一个目标文件参与到动态链接的过程中，那么它的程序头部表将会包含一个类型为 PT_DYNAMIC 的元素。这个段包含了 .dynamic 节。ELF 使用 _DYNAMIC 符号来标记这个节。它的结构如下

```c
/* Dynamic section entry.  */
typedef struct
{
    Elf32_Sword d_tag; /* Dynamic entry type */
    union
    {
        Elf32_Word d_val; /* Integer value */
        Elf32_Addr d_ptr; /* Address value */
    } d_un;
} Elf32_Dyn;
extern Elf32_Dyn_DYNAMIC[];
```

其中，d_tag 的取值决定了该如何解释 d_un。

-   d_val
    -   这个字段表示一个整数值，可以有多种意思。
-   d_ptr
    -   这个字段表示程序的虚拟地址。正如之前所说的，一个文件的虚拟地址在执行的过程中可能和内存的虚拟地址不匹配。当解析动态节中的地址时，动态链接器会根据原始文件的值以及内存的基地址来计算真正的地址。为了保持一致性，文件中并不会包含重定位入口来"纠正"动态结构中的地址。

可以看出，其实这个节是由若干个键值对构成的。

**下表总结了可执行文件以及共享目标文件中的 d_tag 的需求**。如果一个 tag 被标记为"mandatory"，那么对于一个 TIS ELF conforming 的文件来说，其动态链接数组必须包含对应入口的类型。同样的，“optional”意味着可以有，也可以有没有。

| 名称                  | 数值                   | d_un   | 可执行 | 共享 目标 | 说明                                                         |
| --------------------- | ---------------------- | ------ | ------ | --------- | ------------------------------------------------------------ |
| DT_NULL               | 0                      | 忽略   | 必需   | 必需      | 标志着 _DYNAMIC 数组的末端。                                 |
| DT_NEEDED             | 1                      | d_val  | 可选   | 可选      | 包含以NULL 结尾的字符串的字符串表偏移，该字符串给出某个需要的库的名称。所使用的索引为DT_STRTAB的下标。动态数组中可以包含很多个这种类型的标记。这些项在这种类型标记中的相对顺序比较重要。但是与其它的标记之前的顺序倒无所谓。对应的段为.gnu.version_r。 |
| DT_PLTRELSZ           | 2                      | d_val  | 可选   | 可选      | 给出与过程链接表相关的重定位项的总的大小。如果存在DT_JMPREL类型的项，那么DT_PLTRELSZ也必须存在。 |
| DT_PLTGOT             | 3                      | d_ptr  | 可选   | 可选      | 给出与过程链接表或者全局偏移表相关联的地址，对应的段.got.plt |
| DT_HASH               | 4                      | d_ptr  | 必需   | 必需      | 此类型表项包含符号哈希表的地址。此哈希表指的是被 DT_SYMTAB 引用的符号表。 |
| DT_STRTAB             | 5                      | d_ptr  | 必需   | 必需      | 此类型表项包含动态字符串表的地址。符号名、库名、和其它字符串都包含在此表中。对应的节的名字应该是.dynstr。 |
| DT_SYMTAB             | 6                      | d_ptr  | 必需   | 必需      | 此类型表项包含动态符号表的地址。对 32 位的文件而言，这个符号表中的条目的类型为 Elf32_Sym。 |
| DT_RELA               | 7                      | d_ptr  | 必需   | 可选      | 此类型表项包含重定位表的地址。此表中的元素包含显式的补齐，例如 32 位文件中的 Elf32_Rela。目标文件可能有多个重定位节区。在为可执行文件或者共享目标文件创建重定位表时，链接编辑器将这些节区连接起来，形成一个表。尽管在目标文件中这些节区相互独立，但是动态链接器把它们视为一个表。在动态链接器为可执行文件创建进程映像或者向一个进程映像中添加某个共享目标时，要读取重定位表并执行相关的动作。如果此元素存在，动态结构体中也必须包含 DT_RELASZ 和 DT_RELAENT 元素。如果对于某个文件来说，重定位是必需的话，那么 DT_RELA 或者 DT_REL 都可能存在。 |
| DT_RELASZ             | 8                      | d_val  | 必需   | 可选      | 此类型表项包含 DT_RELA 重定位表的总字节大小。                |
| DT_RELAENT            | 9                      | d_val  | 必需   | 可选      | 此类型表项包含 DT_RELA 重定位项的字节大小。                  |
| DT_STRSZ              | 10                     | d_val  | 必需   | 必需      | 此类型表项给出字符串表的字节大小，按字节数计算。             |
| DT_SYMENT             | 11                     | d_val  | 必需   | 必需      | 此类型表项给出符号表项的字节大小。                           |
| DT_INIT               | 12                     | d_ptr  | 可选   | 可选      | 此类型表项给出初始化函数的地址。                             |
| DT_FINI               | 13                     | d_ptr  | 可选   | 可选      | 此类型表项给出结束函数（Termination Function）的地址。       |
| DT_SONAME             | 14                     | d_val  | 忽略   | 可选      | 此类型表项给出一个以 NULL 结尾的字符串的字符串表偏移，对应的字符串是某个共享目标的名称。该偏移实际上是 DT_STRTAB 中的索引。 |
| DT_RPATH              | 15                     | d_val  | 可选   | 忽略      | 此类型表项包含以 NULL 结尾的字符串的字符串表偏移，对应的字符串是搜索库时使用的搜索路径。该偏移实际上是 DT_STRTAB 中的索引。 |
| DT_SYMBOLIC           | 16                     | 忽略   | 忽略   | 可选      | 如果这种类型表项出现在共享目标库中，那么这将会改变动态链接器的符号解析算法。动态连接器将首先选择从共享目标文件本身开始搜索符号，只有在搜索失败时，才会选择从可执行文件中搜索相应的符号。 |
| DT_REL                | 17                     | d_ptr  | 必需   | 可选      | 此类型表项与 DT_RELA类型的表项类似，只是其表格中包含隐式的补齐，对 32 位文件而言，就是 Elf32_Rel。如果ELF文件中包含此元素，那么动态结构中也必须包含 DT_RELSZ 和 DT_RELENT 类型的元素。 |
| DT_RELSZ              | 18                     | d_val  | 必需   | 可选      | 此类型表项包含 DT_REL 重定位表的总字节大小。                 |
| DT_RELENT             | 19                     | d_val  | 必需   | 可选      | 此类型表项包含 DT_REL 重定位项的字节大小。                   |
| DT_PLTREL             | 20                     | d_val  | 可选   | 可选      | 此类型表项给出过程链接表所引用的重定位项的地址。根据具体情况， d_val 对应的地址可能包含 DT_REL 或者  DT_RELA。过程链接表中的所有重定位都必须采用相同的重定位方式。 |
| DT_DEBUG              | 21                     | d_ptr  | 可选   | 忽略      | 此类型表项用于调试。ABI 未规定其内容，访问这些条目的程序可能与 ABI 不兼容。 |
| DT_TEXTREL            | 22                     | 忽略   | 可选   | 可选      | 如果文件中不包含此类型的表项，则表示没有任何重定位表项能够造成对不可写段的修改。如果存在的话，则可能存在若干重定位项请求对不可写段进行修改，因此，动态链接器可以做相应的准备。 |
| DT_JMPREL             | 23                     | d_ptr  | 可选   | 可选      | 该类型的条目的 d_ptr 成员包含了过程链接表的地址，并且索引时应该会把该地址强制转换为对应的重定位表项类型的指针。把重定位表项分开有利于让动态链接器在进程初始化时忽略它们（开启了延迟绑定）。如果存在此成员，相关的 DT_PLTRELSZ 和  DT_PLTREL 必须也存在。 |
| DT_BIND_NOW           | 24                     | 忽略   | 可选   | 可选      | 如果可执行文件或者共享目标文件中存在此类型的表项的话，动态链接器在将控制权转交给程序前，应该将该文件的所有需要重定位的地址都进行重定位。这个表项的优先权高于延迟绑定，可以通过环境变量或者dlopen(BA_LIB)来设置。 |
| DT_LOPROC  ~DT_HIPROC | 0x70000000 ~0x7fffffff | 未指定 | 未指定 | 未指定    | 这个范围的表项是保留给处理器特定的语义的。                   |

没有出现在此表中的标记值是保留的。此外，除了数组末尾的 DT_NULL 元素以及 DT_NEEDED 元素的相对顺序约束以外， 其他表项可以以任意顺序出现。

## Global Offset Table

GOT 表在 ELF 文件中分为两个部分

-   .got，存储导入变量的地址。
-   .got.plt，存储导入函数的地址。

通常来说，地址独立代码不能包含绝对虚拟地址。GOT 表中包含了隐藏的绝对地址，这使得在不违背位置无关性以及程序代码段共享的情况下，得到相关符号的绝对地址。一个程序可以使用位置独立代码来引用它的 GOT 表，然后提取出来绝对的数值，以便于将位置独立的引用重定向到绝对的地址。 

初始时，got 表中包含重定向所需要的信息。当一个系统为可加载的目标文件创建内存段时，动态链接器会处理重定位项，其中的一些项的类型可能是 R_386_GLOB_DAT，这会指向 got 表。动态链接器会决定相关符号的值，计算它们的绝对地址，然后将合适的内存表项设置为相应的值。尽管在链接器建立目标文件时，绝对地址还处于未知状态，动态链接器知道所有内存段的地址，因为可以计算所包含的符号的绝对地址。

如果一个程序需要直接访问一个符号的绝对地址，那么这个符号将会有一个 got 表项。由于可执行文件以及共享目标文件都有单独的表项，所以一个符号的地址可能会出现在多个表中。动态链接器在把权限给到进程镜像中的代码段前，会处理所有的 got 表中的重定位项，以便于确保所有的绝对地址在执行过程中是可以访问的。

GOT 表中的第 0 项包含动态结构（_DYNAMIC）的地址。这使得一个程序，例如动态链接器，在没有执行其重定向前可以找到对应的动态节。这对于动态链接器来说是非常重要的，因为它必须在不依赖其它程序的情况下可以重定位自己的内存镜像。

在不同的程序中，系统可能会为同一共享目标文件选择不同的内存段地址；甚至对于同一个程序，在不同的执行过程中，也会有不同的库地址。然而，一旦进程镜像被建立，内存段的地址就不会再改变，只要一个进程还存在，它的内存段地址将处于固定的位置。

GOT 表的形式以及解释依赖于具体的处理器，对于 Intel 架构来说，`_GLOBAL_OFFSET_TABLE_` 符号可能被用来访问这个表。

```
extern Elf32_Addr _GLOBAL_OFFSET_TABLE[];
```

`_GLOBAL_OFFSET_TABLE_`可能会在 .got 节的中间，以便于可以使用正负索引来访问这个表。

在 Linux 的实现中，.got.plt 的前三项的具体的含义如下

-   GOT[0]，.dynamic 的地址。
-   GOT[1]，指向 link_map 的指针，只会在动态装载器中使用，包含了进行符号解析需要的当前 ELF 对象的信息。每个 link_map 都是一条双向链表的一个节点，而这个链表保存了所有加载的 ELF 对象的信息。
-   GOT[2]，指向动态装载器中 _dl_runtime_resolve 函数的指针。

.got.plt 后面的项则是程序中不同 .so 中函数的引用地址。下面给出一个相应的关系。

![img](./figure/got.png)

## Procedure Linkage Table

PLT 表将导入函数重定向到绝对地址。主要包括两部分

-   **.plt**，与常见导入的函数有关，如 read 等函数。
-   **.plt.got**，与动态链接有关系。

准确的说，plt 表不是查询表，而是一块代码。这一块内容是与代码相关的。

在动态链接下，程序模块之间包含了大量的函数引用。如果在程序开始执行前就把所有的函数地址都解析好，动态链接就会耗费不少时间用于解决模块之间的函数引用的符号查找以及重定位。但是，在一个程序运行过程中，可能很多函数在程序执行完时都不会用到，因此一开始就把所有函数都链接好可能会浪费大量资源，所以 ELF 采用了一种延迟绑定的做法，其基本思想是函数第一次被用到时才进行绑定（符号查找，重定位等），如果没有用则不进行绑定。所以程序开始执行前，模块间的函数调用都没有进行绑定，而是需要用到时才由动态链接器负责绑定。

在惰性绑定的情况下，总体流程如下图所示，蓝线表示首次执行的流程图，红线表示第二次以后调用的流程图：

![](./figure/lazy-plt.png)

LD_BIND_NOW 环境变量可以改变动态链接器的行为。如果它的值非空的话，动态链接器在将控制权交给程序之前会执行 PLT 表项。也就是说，动态链接器在进程初始化过程中执行类型为 R\_3862\_JMP_SLOT 的重定位表项。否则的话，动态链接表会对过程链接表项进行延迟绑定，直到第一次执行对应的表项时，才会今次那个符号解析以及重定位。

注意

>   惰性绑定通常来说会提高应用程序的性能，因为没有使用的符号并不会增加动态链接的负载。然而，有以下两种情况将会使得惰性绑定出现未预期的情况。首先，对于一个共享目标文件的函数的初始引用一般来说会超过后续调用的时间，因为动态链接器需要拦截调用以便于去解析符号。一些应用并不能够忍受这种不可预测性。其次，如果发生了错误，并且动态链接器不能够解析符号。动态链接器将会终止程序。在惰性绑定的情况下，这种情况可能随时发生。当关闭了惰性绑定的话，动态链接器在进程初始化的过程中就不会出现相应的错误，因为这些都是在应用获得控制权之前执行的。

链接编辑器不能够解析执行流转换（比如函数调用），即从一个可执行文件或者共享目标文件到另一个文件。链接器安排程序将控制权交给过程链接表中的表项。在 Intel 架构中，过程链接表存在于共享代码段中，但是他们会使用在 GOT 表中的数据。动态链接器会决定目标的绝对地址，并且会修改相应的 GOT 表中的内存镜像。因此，动态链接器可以在不违背位置独立以及程序代码段兼容的情况下，重定向 PLT 项。可执行文件和共享目标文件都有独立的 PLT 表。

绝对地址的过程链接表如下

```assembly
.PLT0:  pushl	got_plus_4
        jmp	*got_plus_8
	      nop; nop
	      nop; nop
.PLT1:  jmp	*name1_in_GOT
        pushl	$offset@PC
	      jmp	.PLT0@PC
.PLT2:  jmp	*name2_in_GOT
        push	$offset
	     jmp	.PLT0@PC
	     ...
```

位置无关的过程链接表的地址如下

```assembly
.PLT0:  pushl	4(%ebx)
        jmp	*8(%ebx)
	      nop; nop
	      nop; nop
.PLT1:  jmp	*name1_in_GOT(%ebx)
        pushl	$offset
	      jmp	.PLT0@PC
.PLT2:  jmp	*name2_in_GOT(%ebx)
        push	$offset
	      jmp	.PLT0@PC
	      ...
```

可以看出过程链接表针对于绝对地址以及位置独立的代码的处理不同。但是动态链接器处理它们时，所使用的接口是一样的。









动态链接器和程序按照如下方式解析过程链接表和全局偏移表的符号引用。

1.  当第一次建立程序的内存镜像时，动态链接器将 GOT 表的第二个和第三个项设置为特殊的值，下面的步骤会仔细解释这些数值。
2.  如果过程链接表是位置独立的话，那么 GOT 表的地址必须在 ebx 寄存器中。每一个进程镜像中的共享目标文件都有独立的 PLT 表，并且程序只在同一个目标文件将控制流交给 PLT 表项。因此，调用函数负责在调用 PLT表项之前，将全局偏移表的基地址设置为寄存器中。
3.  这里举个例子，假设程序调用了name1，它将控制权交给了 lable .PLT1。
4.  那么，第一条指令将会跳转到全局偏移表中 name1 的地址。初始时，全局偏移表中包含 PLT 中下一条 pushl 指令的地址，并不是 name1 的实际地址。
5.  因此，程序将相应函数在 `rel.plt` 中的偏移（重定位偏移，reloc_index）压到栈上。重定位偏移是 32 位的，并且是非负的数值。此外，重定位表项的类型为 R\_386\_JMP_SLOT，并且它将会说明在之前 jmp 指令中使用的全局偏移表项在 GOT 表中的偏移。重定位表项也包含了一个符号表索引，因此告诉动态链接器什么符号目前正在被引用。在这个例子中，就是 name1了。
6.  在压入重定位偏移后，程序会跳转到 .PLT0，这是过程链接表的第一个表项。pushl 指令将 GOT 表的第二个表项(got_plus_4 或者4(%ebx)，**当前ELF对象的信息**)压到栈上，然后给动态链接器一个识别信息。此后，程序会跳转到第三个全局偏移表项(got_plus\_8 或者8(%ebx)，**指向动态装载器中 `_dl_runtime_resolve` 函数的指针**) 处，这将会将程序流交给动态链接器。
7.  当动态链接器接收到控制权后，他将会进行出栈操作，查看重定位表项，解析出对应的符号的值，然后将 name1 的地址写入到全局偏移表项中，最后将控制权交给目的地址。
8.  过程链接表执行之后，程序的控制权将会直接交给 name1 函数，而且此后再也不会调用动态链接器来解析这个函数。也就是说，在 .PLT1 处的 jmp 指令将会直接跳转到 name1 处，而不是再次执行 pushl 指令。

## .rel(a).dyn & .rel(a).plt

.rel.dyn 包含了动态链接的二进制文件中需要重定位的变量的信息，而 .rel.plt 包含了需要重定位的函数的信息。这两类重定位节都使用如下的结构（以 32 位为例）

```
typedef struct {
    Elf32_Addr        r_offset;
    Elf32_Word       r_info;
} Elf32_Rel;

typedef struct {
    Elf32_Addr     r_offset;
    Elf32_Word    r_info;
    Elf32_Sword    r_addend;
} Elf32_Rela;
```

Elf32_Rela 类型的表项包含明确的补齐信息。 Elf32_Rel 类型的表项在将被修改的位置保存隐式的补齐信息。由于处理器体系结构的原因，这两种形式都存在，甚至是必需的。因此，对特定机器的实现可以仅使用一种形式，也可以根据上下文使用两种形式。

其中，每个字段的说明如下

| 成员     | 说明                                                         |
| -------- | ------------------------------------------------------------ |
| r_offset | **此成员给出了需要重定位的位置。**对于一个可重定位文件而言，此值是从需要重定位的符号所在节区头部开始到将被重定位的位置之间的字节偏移。对于可执行文件或者共享目标文件而言，其取值是需要重定位的**虚拟地址**，一般而言，也就是说我们所说的 GOT 表的地址。 |
| r_info   | **此成员给出需要重定位的符号的符号表索引，以及相应的重定位类型。**  例如一个调用指令的重定位项将包含被调用函数的符号表索引。如果索引是 STN_UNDEF，那么重定位使用 0 作为“符号值”。此外，重定位类型是和处理器相关的。 |
| r_addend | 此成员给出一个常量补齐，用来计算将被填充到可重定位字段的数值。 |

关于 r_info 更加具体的字段信息如下面的代码所示

- r_info 的高三个字节对应的值表示这个动态符号在 `.dynsym` 符号表中的位置
- r_info 的最低字节表示的是重定位类型类型

```c
#define ELF32_R_SYM(i)    ((i)>>8)
#define ELF32_R_TYPE(i)   ((unsigned char)(i))
// 用于构造 r_info
#define ELF32_R_INFO(s,t) (((s)<<8)+(unsigned char)(t))
```

重定位节区会引用两个其它节区：**符号表、要修改的节区**。节区头部的 sh_info 和 sh_link 成员给出相应的关系。

这里，我们具体讨论可能的重定位类型。在下面的计算中，我们假设是把一个可重定位文件转换为可执行文件或者共享目标文件。从概念上讲，链接器会把一个或者多个可重定位文件合并起来得到输出文件。它首先要决定如何结合并放置这些输入文件，然后更新符号表的值，最后才进行重定位。可执行文件或者共享目标文件的重定位方法是相似的，并且结果几乎一样。在后面的描述中我们将会采用如下记号。

-   A(addend) 用来计算可重定位字段的取值的补齐。
-   B(base)  表示共享目标文件在执行过程中被加载到内存中的基地址。一般来说，共享目标文件的虚拟基地址为 0，但是在执行时，其地址却会发生改变。
-   G(Global) 表示在执行时重定位项的符号在全局偏移表中的偏移。
-   GOT (global offset table) 表示全局偏移表（GOT）的地址。
-   L (linkage) 表示过程链接表项中一个符号的节区偏移或者地址。过程链接表项会把函数调用重定位到正确的目标位置。链接编辑器会构造初始的过程链接表，然后动态链接器在执行过程中会修改这些项目。
-   P (place) 表示被重定位（用 r_offset 计算）的存储单元的位置（节区偏移或者地址）。
-   S  (symbol) 表示索引位于重定位项中的符号的取值。

重定位项的 r_offset 取值为受影响的存储单元的第一个字节的偏移或者虚拟地址。重定位类型给出需要修改的比特位以及如何计算它们的值。其中，Intel 架构只使用 ELF32_REL 重定位表项，将要被重定位的成员保留对应的补齐数值。在所有的情况下，补齐的数值与计算的结果使用相同的字节序。

重定位类型以及部分含义如下

| 名称           | 值   | 域     | 计算        | 含义                                                         |
| -------------- | ---- | ------ | ----------- | ------------------------------------------------------------ |
| R_386_NONE     | 0    | 无     | 无          |                                                              |
| R_386_32       | 1    | word32 | S + A       |                                                              |
| R_386_PC32     | 1    | word32 | S + A - P   |                                                              |
| R_386_GOT32    | 1    | word32 | G + A - P   | 该重定位类型计算从全局偏移表基址到符号的全局偏移表项的距离。另外，它还命令链接器创建一个全局偏移表。 |
| R_386_PLT32    | 1    | word32 | L + A - P   | 该重定位类型计算符号的过程链接表项地址。另外，它还命令链接器创建一个过程链接表。 |
| R_386_COPY     | 5    | 无     | 无          | 该重定位类型由链接器为动态链接过程创建。它的偏移项指向可写段中的位置。符号表规定这种符号应既存在于当前目标文件又该存在于共享目标文件中。在执行过程中，动态链接器将与该共享目标符号相关的数据复制到由上述偏移量指定的位置。 |
| R_386_GLOB_DAT | 6    | word32 | S           | 该重定位类型用于把一个全局偏移表中的符号设置为指定符号的地址。这个特殊的重定位类型允许确定符号和全局偏移表项之间的关系。 |
| R_386_JMP_SLOT | 7    | word32 | S           | 该重定位类型由链接器为动态链接过程创建。它的偏移项给出了相应过程链接表项的位置。动态链接器修改过程链接表，从而把程序控制权转移到上述指出的符号地址。 |
| R_386_RELATIVE | 8    | word32 | B + A       | 该重定位类型由链接器为动态链接过程创建。它的偏移项给出了共享目标中的一个包含了某个代表相对地址的值的位置。动态链接器通过把共享目标文件装载到的虚拟地址与上述相对地址相加来计算对应虚拟地址。这种类型的重定位项设置符号表索引为0。 |
| R_386_GOTOFF   | 9    | word32 | S + A - GOT | 该重定位类型计算符号值与全局偏移表地址之间的差。此外，它还通知链接器创建一个全局偏移表。 |
| R_386_GOTPC    | 10   | word32 | S + A - P   | 该重定位类型与`R_386_PC32` 类似，只不过它在计算时使用全局偏移表的地址。正常情况下，该重定位表项中被引用的符号是`_GLOBAL_OFFSET_TABLE_` ，它会命令链接器创建一个全局偏移表。 |

## .dynsym

动态链接的 ELF 文件具有专门的动态符号表，其使用的结构也是 Elf32_Sym，但是其存储的节为 .dynsym。这里再次给出 Elf32_Sym 的结构

```c
typedef struct
{
  Elf32_Word    st_name;   /* Symbol name (string tbl index) */
  Elf32_Addr    st_value;  /* Symbol value */
  Elf32_Word    st_size;   /* Symbol size */
  unsigned char st_info;   /* Symbol type and binding */
  unsigned char st_other;  /* Symbol visibility under glibc>=2.2 */
  Elf32_Section st_shndx;  /* Section index */
} Elf32_Sym;
```

需要注意的是 `.dynsym` 是运行时所需的，ELF 文件中 export/import 的符号信息全在这里。但是，`.symtab` 节中存储的信息是编译时的符号信息，它们在 `strip` 之后会被删除掉。

我们主要关注动态符号中的两个成员

-   st_name，该成员保存着动态符号在 .dynstr 表（动态字符串表）中的偏移。
-   st_value，如果这个符号被导出，这个符号保存着对应的虚拟地址。

动态符号与指向它的 Elf_Verdef 之间的关联性保存在 `.gnu.version` 节中。这个节是由 Elf_Verneed 结构体构成的数组。其中，每个表项对应动态符号表的一项。其实，这个结构体就只有一个域：那就是一个 16 位的整数，表示在 `.gnu.verion_r` 段中的下标。

除此之外，动态链接器使用 Elf_Rel 结构体成员 r_info 中的下标同时作为 .dynsym 段和 .gnu.version 段的下标。这样就可以一一对应到每一个符号到底是那个版本的了。

## .dynstr

这个节包含了动态链接所需要的字符串。

## Misc 

### version releated sections

ELF 文件不仅可以导入外部的符号，而且还可以导入指定版本的符号。例如，当我们可以从 GLIBC_2.2.5 中导入其中的一些标准库函数，比如 printf。其中，.gnu.version_r 保存了版本的定义，对应的结构体是 Elf_Verdef。

#### .gnu.version

该节与 .dynsym 中的符号信息一一对应，即两者的元素个数一样。.gnu.version 中每一个元素的类型是 `Elfxx_Half`，指定了对应符号的版本信息。`Elfxx_Half` 中有两个值是保留的

- 0，表示这个符号是本地的，对外不公开。下面的`__gmon_start__` 就是一个本地符号。
- 1，表示这个符号在当前这个目标文件中定义，并且是全局可以访问的。下面的 `_IO_stdin_used` 就是一个全局符号。

```assembly
LOAD:080482D8 ; ELF GNU Symbol Version Table
LOAD:080482D8                 dw 0
LOAD:080482DA                 dw 2                    ; setbuf@@GLIBC_2.0
LOAD:080482DC                 dw 2                    ; read@@GLIBC_2.0
LOAD:080482DE                 dw 0                    ; local  symbol: __gmon_start__
LOAD:080482E0                 dw 2                    ; strlen@@GLIBC_2.0
LOAD:080482E2                 dw 2                    ; __libc_start_main@@GLIBC_2.0
LOAD:080482E4                 dw 2                    ; write@@GLIBC_2.0
LOAD:080482E6                 dw 2                    ; stdin@@GLIBC_2.0
LOAD:080482E8                 dw 2                    ; stdout@@GLIBC_2.0
LOAD:080482EA                 dw 1                    ; global symbol: _IO_stdin_used
...
.rodata:0804866C                 public _IO_stdin_used
.rodata:0804866C _IO_stdin_used  db    1                 ; DATA XREF: LOAD:0804825C↑o
.rodata:0804866D                 db    0
.rodata:0804866E                 db    2
.rodata:0804866F                 db    0
.rodata:0804866F _rodata         ends
```

#### .gnu.version_d

Version definitions of symbols.

#### .gnu.version_r

Version references (version needs) of symbols.

## 参考

- https://refspecs.linuxfoundation.org/LSB_3.0.0/LSB-PDA/LSB-PDA.junk/symversion.html