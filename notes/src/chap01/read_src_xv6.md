# XV6的源码——引导部分（bootloader）
bootloader包括两个源文件：

 - bootasm.S（由16位和32位汇编混合编写）
 - bootmain.c


## bootasm.S
**一、bootloader启动的第一步**
```bash
# Start the first CPU: switch to 32-bit protected mode, jump into C.
# The BIOS loads this code from the first sector of the hard disk into
# memory at physical address 0x7c00 and starts executing in real mode
# with %cs=0 %ip=7c00.

.code16                       # Assemble for 16-bit mode
.globl start
start:
  cli                         # BIOS enabled interrupts; disable

  # Zero data segment registers DS, ES, and SS.
  xorw    %ax,%ax             # Set %ax to zero
  movw    %ax,%ds             # -> Data Segment
  movw    %ax,%es             # -> Extra Segment
  movw    %ax,%ss             # -> Stack Segment
```
- **第一条指令cli：屏蔽处理器中断**
BIOS作为一个小型操作系统，为了初始化硬件设备，可能设置了自己的中断处理程序。
但由于现在BIOS没有控制权，所以允许中断不合理且不安全。当xv6准备好后，会重新允许中断。
-  屏蔽中断后，**将%ax置零，并把这个零值拷贝到三个段寄存器中：%dx（data segment），%es（extra segment），%ss（stack segment）**
BIOS需要通过实模式启动，但当引入XV6的内核时，需要从实模式转换到保护模式。
BIOS完成工作后，%ds，%es，%ss的值未知，但实模式刚运行完，寄存器可能存在遗留的值，所以需要清零

> 我们的操作系统有8个16位通用寄存器可用，但实际上处理器发送给内存的是20位的地址。这时，多出来的4位其实是由段寄存器%cs, %ds, %es, %ss提供的。当程序用到一个内存地址时，处理器会自动在该地址上加上某个16位段寄存器值的16倍。因此，内存引用中其实隐含地使用了段寄存器的值：取值会用到 %cs，读写数据会用到 %ds，读写栈会用到 %ss
> ![在这里插入图片描述](https://img-blog.csdnimg.cn/7d78741d81d94351a57ef8e0250fe82a.png)


**二、打开A20开关**
```bash
 # Physical address line A20 is tied to zero so that the first PCs 
  # with 2 MB would run software that assumed 1 MB.  Undo that.
seta20.1:
  inb     $0x64,%al               # Wait for not busy
  testb   $0x2,%al
  jnz     seta20.1

  movb    $0xd1,%al               # 0xd1 -> port 0x64
  outb    %al,$0x64

seta20.2:
  inb     $0x64,%al               # Wait for not busy
  testb   $0x2,%al
  jnz     seta20.2

  movb    $0xdf,%al               # 0xdf -> port 0x60
  outb    %al,$0x60
```
- seta20.1：
**向 804x 键盘控制器的 0x64 端口发送命令**。这里传送的命令是 0xd1，这个命令的意思是要向键盘控制器的 P2 写入数据。
- seta20.2：
**向键盘控制器的 P2 端口写数据**。写数据的方法是把数据通过键盘控制器的 0x60 端口写进去。写入的数据是 0xdf，因为 A20 gate 就包含在键盘控制器的 P2 端口中，随着 0xdf 的写入，A20 gate 就被打开了。

 A20开关是？
 出现80286以后，为了保持和8086的兼容，PC机在设计上在第21条地址线（也就是A20）上做了一个开关
 当这个开关打开时，这条地址线和其它地址线一样可以使用
 当这个开关关闭时，第21条地址线（A20）恒为0
 这个开关就叫做A20 Gate

 打开A20开关的原因？
 在保护模式下，使用32位地址线
 如果A20被禁止，即恒等于0，那么系统只能访问奇数兆的内存，即只能访问0--1M、2-3M、4-5M......，这显然是不行的；
 A20打开，可以访问的内存才是连续的
 所以在保护模式下，这个开关也必须打开

>  虚拟地址 segment:offset 可能产生21位物理地址，但 Intel 8088 只能向内存传递20位地址，所以它截断了地址的最高位：0xffff0 + 0xffff = 0x10ffef，但在8088上虚拟地址 0xffff:0xffff 则是引用物理地址 0x0ffef。早期的软件依赖硬件来忽略第21位地址位，所以当 Intel 研发出使用超过20位物理地址的处理器时，IBM 就想出了一个技巧来保证兼容性。那就是，如果键盘控制器输出端口的第2位是低位，则物理地址的第21位被清零；否则，第21位可以正常使用。引导加载器用 I/O 指令控制端口 0x64 和 0x60 上的键盘控制器，使其输出端口的第2位为高位，来使第21位地址正常工作。

>对于使用内存超过65536字节的程序而言，实模式的16位寄存器和段寄存器就显得非常困窘了，显然更不可能使用超过 1M 字节的内存。x86系列处理器在80286之后就有了保护模式。保护模式下可以使用更多位的地址，并且（80386之后）有了“32位”模式使得寄存器，虚拟地址和大多数的整型运算都从16位变成了32位。xv6 引导程序依次允许了保护模式和32位模式。


**三、切换到保护模式，设置GDT**

```bash
  # Switch from real to protected mode.  Use a bootstrap GDT that makes
  # virtual addresses map directly to physical addresses so that the
  # effective memory map doesn't change during the transition.
  lgdt    gdtdesc
  movl    %cr0, %eax
  orl     $CR0_PE, %eax
  movl    %eax, %cr0

//PAGEBREAK!
  # Complete the transition to 32-bit protected mode by using a long jmp
  # to reload %cs and %eip.  The segment descriptors are set up with no
  # translation, so that the mapping is still the identity mapping.
  ljmp    $(SEG_KCODE<<3), $start32
  
.code32  # Tell assembler to generate 32-bit code now.
```
- 引导加载器执行下面的指令来把**指向gdt的指针gdtdesc加载到全局描述符表（GDT）寄存器中**
```bash
 lgdt    gdtdesc
```
全局描述符表是？
进入保护模式后，数据段、代码段等内存段不再是通过段寄存器获得段基址就可以使用，我们需要把段定义、登记好，全局描述符表便是用于记录这些段信息的数据结构。
GDT 表里的每一项叫做“段描述符”，用来记录每个内存分段的一些属性信息，每个“段描述符”占 8 字节。
GDT可以被放在系统内存中的任何位置，寄存器GDTR用来存放GDT的入口地址。在设定好GDT的位置后，通过lgdt指令将GDT的入口地址装入寄存器。此后，CPU可以通过寄存器中的内容作为GDT的入口来访问GDT。


- **开启保护模式**
将 %cr0 中的 CR0_PE 位置为1
  
- **将16位模式切换到32位**
SEG_KCODE是？
宏定义，定义在mmu.h中，为1
>允许保护模式并不会马上改变处理器把逻辑地址翻译成物理地址的过程；只有当处理器读取 GDT 地址的段寄存器的值被改变为内部段的设置后，才会发生变化。
>我们没法直接修改 %cs，所以使用了一个 ljmp 指令。跳转指令会接着在下一行执行，但这样做实际上将 %cs 指向了 gdt 中的一个代码描述符表项。该描述符描述了一个32位代码段，这样处理器就切换到了32位模式下。




**四、保护模式下初始化数据**

```bash
 start32:
  # Set up the protected-mode data segment registers
  movw    $(SEG_KDATA<<3), %ax    # Our data segment selector
  movw    %ax, %ds                # -> DS: Data Segment
  movw    %ax, %es                # -> ES: Extra Segment
  movw    %ax, %ss                # -> SS: Stack Segment
  movw    $0, %ax                 # Zero segments not ready for use
  movw    %ax, %fs                # -> FS
  movw    %ax, %gs                # -> GS
```
- **用SEG_KDATA（值为2）初始化数据段寄存器（%ds，%es，%ss）**。
逻辑地址现在是直接映射到物理地址的。
- **用0初始化%fs，%gs**。
 
**五、运行bootmain.c**

```bash
  # Set up the stack pointer and call into C.
  movl    $start, %esp
  call    bootmain
```
- 运行 C 代码之前的最后一个步骤是**在空闲内存中建立一个栈**。
>内存 0xa0000 到 0x100000 属于设备区，而 xv6 内核则是放在 0x100000 处。引导加载器自己是在 0x7c00 到 0x7d00。本质上来讲，内存的其他任何部分都能用来存放栈。引导加载器选择了 0x7c00（在该文件中即 $start）作为栈顶；栈从此处向下增长，直到 0x0000，不断远离引导加载器代码。

- 最后**加载器调用 C 函数 bootmain**
>bootmain 的工作就是加载并运行内核。只有在出错时该函数才会返回，这时它会向端口 0x8a00（8470-8476）输出几个字。
>在真实硬件中，并没有设备连接到该端口，所以这段代码相当于什么也没有做。
>如果引导加载器是在 PC 模拟器上运行，那么端口 0x8a00 则会连接到模拟器并把控制权交还给模拟器本身。
>无论是否使用模拟器，这段代码接下来都会执行一个死循环。而一个真正的引导加载器则应该会尝试输出一些调试信息。

## bootmain.c
目的是在磁盘的第二个扇区开头找到内核程序，将elf格式的内核从硬盘加载到内存中，然后跳转到入口处执行。
```bash
// Boot loader.
//
// Part of the boot block, along with bootasm.S, which calls bootmain().
// bootasm.S has put the processor into protected 32-bit mode.
// bootmain() loads an ELF kernel image from the disk starting at
// sector 1 and then jumps to the kernel entry routine.

#include "types.h"
#include "elf.h"
#include "x86.h"
#include "memlayout.h"

#define SECTSIZE  512

void readseg(uchar*, uint, uint);

void
bootmain(void)
{
  struct elfhdr *elf;
  struct proghdr *ph, *eph;
  void (*entry)(void);
  uchar* pa;

  elf = (struct elfhdr*)0x10000;  // scratch space

  // Read 1st page off disk
  readseg((uchar*)elf, 4096, 0);
```
- 为了读取 ELF 头，**bootmain 载入 ELF 文件的前4096字节（8514），并将其拷贝到内存中 0x10000 处**。
```bash
 // Is this an ELF executable?
  if(elf->magic != ELF_MAGIC)
    return;  // let bootasm.S handle error
```
- 通过 ELF 头**检查这是否的确是一个 ELF 文件**。

```bash
  // Load each program segment (ignores ph flags).
  ph = (struct proghdr*)((uchar*)elf + elf->phoff);
  eph = ph + elf->phnum;
  for(; ph < eph; ph++){
    pa = (uchar*)ph->paddr;
    readseg(pa, ph->filesz, ph->off);
    if(ph->memsz > ph->filesz)
      stosb(pa + ph->filesz, 0, ph->memsz - ph->filesz);
  }
```
- bootmain **从磁盘中 ELF 头之后 off 字节处读取扇区的内容，并写到内存中地址 paddr 处**。
- **调用 readseg 将数据从磁盘中载入**。
- **调用 stosb 将段的剩余部分置零**。
 stosb使用 x86 指令 rep stosb 来初始化内存块中的每个字节。
```bash
  // Call the entry point from the ELF header.
  // Does not return!
  entry = (void(*)(void))(elf->entry);
  entry();
}
```
- **调用内核的入口指令，即内核第一条指令的执行地址，跳转到内核的入口代码处执行**

**函数实现**
- readseg：将数据从磁盘载入
```bash
// Read 'count' bytes at 'offset' from kernel into physical address 'pa'.
// Might copy more than asked.
void
readseg(uchar* pa, uint count, uint offset)
{
  uchar* epa;

  epa = pa + count;

  // Round down to sector boundary.
  pa -= offset % SECTSIZE;

  // Translate from bytes to sectors; kernel starts at sector 1.
  offset = (offset / SECTSIZE) + 1;

  // If this is too slow, we could read lots of sectors at a time.
  // We'd write more to memory than asked, but it doesn't matter --
  // we load in increasing order.
  for(; pa < epa; pa += SECTSIZE, offset++)
    readsect(pa, offset);
}
```
根据起始地址pa和大小count基算结束地址epa
pa -= offset % SECTSIZE：计算出从硬盘拷贝扇区到内存中的实际地址
offset = (offset / SECTSIZE) + 1：计算offset在实际哪个硬盘分区。内核是从硬盘的第一个扇区开始，所以要加1。
循环读取数据。

- readsect：读取硬盘数据

```bash
void
waitdisk(void)
{
  // Wait for disk ready.
  while((inb(0x1F7) & 0xC0) != 0x40)
    ;
}
```
```bash
// Read a single sector at offset into dst.
void
readsect(void *dst, uint offset)
{
  // Issue command.
  waitdisk();
  outb(0x1F2, 1);   // count = 1
  outb(0x1F3, offset);
  outb(0x1F4, offset >> 8);
  outb(0x1F5, offset >> 16);
  outb(0x1F6, (offset >> 24) | 0xE0);
  outb(0x1F7, 0x20);  // cmd 0x20 - read sectors

  // Read data.
  waitdisk();
  insl(0x1F0, dst, SECTSIZE/4);
}
```
>![在这里插入图片描述](https://img-blog.csdnimg.cn/1e0f0a2e5ab84b10b100e8564ffe3bd1.png)各个端口的解释

