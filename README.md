# learningOS_blog
2022 OS 训练营每日进度记录. 
慢慢来, 比较快. 不贪多, 力求弄懂眼前遇到的问题. 

## day 23
### 计划
1. 修好昨天遇到的问题

### 成果
1. 今天事情有点多, 只有晚上尝试调试下项目, 没有思路. git 切换到还没重构时的分支, 运行观察log, 竟然此时的entrypoint也都是0! 而且还能正常运行??? 我怀疑自己打的log有问题. 明天恐怕得在gdb里慢慢跟汇编才能找到原因了. 

## day 22
### 计划
1. 尝试把os4的代码贴到os5里, 并运行

### 成果
1. 结合文档, 感觉这次实验需要对我的~~屎山~~代码进行较大力度的重构...
2. 部分重构后, 解析elf竟然读不出entrypoint了, 明天再找找原因.
3. 对于引用borrow的问题, 今天又发现一些小技巧: 当对一较复杂结构体的成员进行多次引用(包括可变和不可变), 常常容易报错, 不妨在冲突的借用之前手动drop较早的引用, 可能可以解决问题.

## day 21
### 计划
1. 一把梭哈完某不愿透露姓名的知识工程课作业. 

### 成果
1. 梭哈完了, 明天回归正轨!

## day 20
### 计划
1. 参加 unicore 培训课.

### 成果
1. 可能是上一轮培训还没结束, 所以这节课似乎并不面向刚加入的新人. 分享的主题是内存分配器和异步运行时内核模块. 内存分配器模块大概是伙伴算法+二叉平衡树.
2. 杨老师分享了一些资料, fast-trap, "200 行 Rust 解读 Futures" 等, 日后一定好好学习一下. 

## day 19
没有进度的一天, 有些杂事, 摸了.

## day 18
### 计划
1. 今天一定要写完实验4.
2. 浅读 实验5 文档.

### 成果
1. 实验4--地址空间, 本地/github测例全部通过, 坚持这么多天实属不易, 撒花🎈~
2. 一点感想: 这个系列的实验难度着实不小, 最近几天能明显感觉自己的焦躁情绪比较重. 接下来的学习过程要端正心态, 戒骄戒躁, 做好长线作战的心理准备. 

## day 17
### 计划
1. 写实验4.

### 成果
1. 内核栈内存没有写权限的问题, 其实简单的加个 mut 就可以解决. 但是可写静态变量在被访问时必须是 unsafe 的. 我想到 sync 模块里有个 UPSafeCell, 或许就是能解决这个问题的? 尝试用了下有问题, 以后可以仔细研究下.
2. 陆陆续续遇到了些小问题, 但是基本都能在文档 "基于地址空间的分时多任务" 一节中找到答案, 例如切换页表后指令地址不连续/切换页表后跳板函数代码无执行权限/保存和恢复上下文sp指针没对齐等等. 也幸亏有文档可以参考, 否则我很难凭自己解决.
3. mmap munmap 的测例有点烦, 区间的开闭我总处理不太好 (算法苦手), 现在(晚11点)已经很晚了, 明天再搞吧.

## day 16
### 计划
1. 写实验4, 最终还是决定想按照昨天的想法尝试实现一下. 拿出勇气, 别畏手畏脚, 想到了, 就去做!

### 成果
1. 在初始化 app 的内核栈时, 我传入一个构造好的 TaskContext, 想把它通过解引用裸指针写入内核栈中. 遇到了奇怪的问题. 
2. 经过漫长的调试, 发现在赋值那行语句会出现异常, gdb 显示异常类型($scause)是15. 我顺手在rust代码中查看 riscv::register::scause::Exception 的枚举类型, 发现对应的异常是 StoreGuestPageFault. 
3. 查阅资料, StoreGuestPageFault是二阶段页表相关的页错误, 这让我百思不得其解. 其实这里我被误导了, rust库中的枚举和cpu的异常号不是对应的. 这里触发的异常的确就是 Store Page Fault. 
4. 在上一次实验中, 我使用相同的办法写入内核栈, 为何这次行不通了呢? 同样与页表有关. 上次实验没有开启页表, 所以我推测当时应该没有读写权限的检查, 而这次实验内核栈被编译器放在了 rodata 段中(我一开始想当然的以为是在 bss 段), 所以没有写权限.
5. 所以现在的想法是, 可能需要用堆分配器动态申请内核栈内存, 明天尝试一下吧. 

## day 15
### 计划
1. 写实验4, 实现昨天的想法.

### 成果
1. 注意到示例代码中, 中断处理函数内, 原先的 LoadFault, StoreFault被改为 Load page fault 和 store page fault. 查阅资料后发现原来前者是 PMP 机制出错的异常, 后者才是常见的缺页异常.
2. 注意到文档中提到 RAII风格 这种说法, 似乎最早是 C++ 中提出的一种设计思想, 主要用于封装资源的申请和释放.  
3. 开启分页后, 原先(os3)中写的很多代码都需要重构. 最主要的一点是, 我原先的实现是 TrapContext 内嵌在 TaskContext 结构体内, 由于前者是后者的第一个成员, 所以在解裸指针时, 可以任意按这两个类型解引用. 但是文档中在开启分页后, TrapContext 需要放置在用户地址空间, 而TaskContext要放在内核地址空间的栈中, 二者无法再复用相同的地址了. 
4. 为了解决刚才的问题, 除了按照文档和示例重新实现相关代码之外, 我还想到一种弥补的办法. 或许可以将内核栈和用户空间的TrapContext段保持相同的大小并且页对齐, 在初始化用户地址空间时, 将TrapContext直接映射到该应用对应的内核栈上. 或许可以和当前的设计保持兼容. 有点纠结要不要花时间按这个思路做实现, 如果行不通的话可能会浪费很多时间.

## day 14
### 计划
1. 实验4的文档太长了, 能读多少读多少, 然后跑一下代码试试水.

### 成果
1. 读文档时的心态很不好. 一方面因为文档很长, 示例代码有非常多繁琐的结构体定义, 读起来相当乏味; 另一方面, riscv的SV39页表机制我已经读过很多次相关文档了: 比如 xv6 和 Writing an OS in rust. 看到熟悉的内容忍不住想跳, 但是又怕错过什么细节.

2. os4 里面居然又是空的, 我还得把 os3 里自己写的部分copy进来, 有点麻烦. 编译通过后运行发现所有 app 的 entrypoint 都变成 0x80400000, 这说明每个应用都得有自己的虚拟地址空间才可以正常运行. 既然 loader 里可以获取到每个app二进制数据的起止地址, 那么应该只需要做两件事(先不考虑释放资源): 
    * 启动app时构造应用页表, 它会将 0x80400000 起一定长度的(取决于具体应用二进制数据的起止地址)的虚拟地址映射到真正放有app的物理地址上. 
    * 运行app时出现 loadFault 或 storeFault, 向页帧分配器申请一个空页, map到用户页表中. 
    
明天尝试实现下目前能想到的思路.

## day 12 & day 13
1. 没有进度的两天. day 12 学校停电检修 (真的不是我摸鱼), day 13 摸了. 也用了一些时间做其它课程的作业.  

## day 11
### 计划
1. 今天一定一定要搞完实验3.
2. 浅读实验4的文档.

### 成果
1. 排查昨天 gettimeofday 出bug的原因在于自己局部重构代码时, 应该向 x[10] 寄存器写入返回值 0, 误写为 x[17]. 出现这种低级错误一方面因为自己粗心大意, 另一方面也说明自己对于riscv的ABI还不是那么了如指掌. 好好反思!

2. 实现 taskinfo 接口时怎么也过不了测例, 发现原来是自己实现获取时间戳部分一直都有问题. time::read() 获取的是 CPU 时钟计数, 它不是秒/微秒等单位, 所以需要手动换算一下. (所以之前怎么过的 sleep 之类的测例啊😅)

3. 本地/github测试全部 passed, 坚持这么多天实属不易, 撒花~ 

## day 10
### 计划
1. 今天一定要搞完实验3. 

### 成果
1. 遇到了各种各样的小问题, 例如: 在实现 write 系统调用的时候, 我一开始这么写的,
```rs
let _user_buf = unsafe { String::from_raw_parts(buf as *mut u8, len, len) };
print!("{}", _user_buf);
```
但是这行代码在第一次执行时没问题, 第二次执行的话就会出错 (哪怕是同一个程序传同样的参数).
正确的实现是:
```rs
let user_buf = {
    let slice = unsafe { core::slice::from_raw_parts(buf as *const u8, len) };
    core::str::from_utf8(slice).unwrap()
};
print!("{}", user_buf);
```
具体原因还不清楚, 后续有机会调查一下.

2. 实现了按时间片轮转调度任务, 实现的过程中在 Rust 引用借用上花了不少时间, 发现个小窍门: 如果对同一个结构体读写访问很复杂, 不妨以元组的形式批量把成员读到局部变量中, 大概率能简化引用借用的问题. 

3. 实现简单调度后, 原本没有问题的 gettimeofday 接口竟然不 work 了, 明天继续 debug.

## day 9
### 计划
1. 今天目测虽然不闲, 但是不搞完实验3不甘心, 一定要弄完. 

### 成果
1. gdb调试看用户态程序的汇编, loadFault Exception 的原因在于尝试向 0x0 地址写入数据. 首先怀疑自己的 TrapContext 构造是否正确.
2. 跟 os-ref 的 TrapContext 做对比, 发现 OS-ref 里是用 TaskContext 做切换的. 看这部分代码有点懵. 最后基本是按照文档里写的代码, 应该没问题.
3. 其次怀疑是否没有正确初始化用户态栈, 同样调试 os3-ref, 发现它的用户态程序刚启动时栈也是空的. 
3. 其次怀疑 __restore 是否正确实现了, gdb 单步调试, 发现确实准确的跳到了 app 的entrypoint, 而且 sp 指针也正确指向了预期的栈地址. 
4. 此时我已经蒙了, 按照我的理解, 程序是状态机, 如果起始状态相同, 那么没有理由会出现不同的执行结果. 然而我已经确认寄存器和栈已经初始化好了, 怎么会出现访问空指针呢... 漫无目的的调试, 甚至怀疑过我这个版本的loader实现错了, 没有把正确的程序链接进来. 
5. 已经快要放弃, 抱着最后的希望在gdb里查看寄存器, 发现自己很多寄存器很多都是非0, os3-ref里基本都是0(除了sp, epc等), 原来是自己实现__restore 时认为反正目前它只作为程序的入口, 所以通用寄存器无需初始化为0 (底层逻辑是认为编译器在用通用寄存器前会初始化), 导致后面应用的控制流截然不同. 修正这部分实现后, 果然没有空指针的问题了.  
6. 得去搞实验室的工作了, 不然师兄非得鲨了我, 溜了溜了, 明天再肝.

## day 8
### 计划
1. 今天目测比较闲, 一定要把 lab1 (test3) 做完.

### 成果
1. 这次实验比想象的难得多. os3 目录给出了一个基本的框架, 但是执行用户态程序/处理中断/上下文的保存与恢复/系统调用分发与实现/时间片等等都需要自己给出实现.
2. 在内联汇编上踩坑花了些时间, 汇编中有中文注释 Rust 就只能读出空字符串.
3. 目前可以将控制流切入用户程序, 但是执行到某个 ld 执行时会触发 loadFault Exception, 原因还不确定. 
4. 深刻体会到: 了解原理与动手实现之间差了十万八千里.

## day 7
### 计划
1. 尝试做 lab1 (test3).

### 成果
1. 浅读实验指导的第三章, 无论是协作式调度或是抢占式调度的原理我都比较熟悉了, 所以读下来障碍并不太多 (也可能是因为我读的不细). 文档中代码部分有些太长了, 所以为了节省时间略过了一些. 
2. 今天来不及做习题了, 留到明天吧.

## day 6
### 计划
1. 继续昨天未完成的 lab0-1. 

### 成果
1. 精读实验指导的第二章. 对于保存上下文和链接用户态程序的汇编代码理解的比较透彻了. 
2. 没改动任何文件, 将仓库 push 后, CI竟然通过了. 仿照着 Test2 实验的 toolchain 配置, 我又重新将 Test1 仓库上传了几次, 但是仍然没有解决 CI 报错的问题. 似乎是 CI 脚本有问题? 
3. 从家回学校了, 想必明天起我将进入高产期 (想peach).

## day 5
### 计划
1. 尝试本地运行 lab0-1 (Test2), 以及阅读相关文档. 

### 成果
1. 浅读了下第二章的"引言"和"实现应用程序"两节. 基本理解了用户态程序的编译机制. (这两天回趟家, 时间比较紧张)

## day 4
1. 今天事情有点多, 过. 

## day 3
### 计划
1. 跑一下step2 lab0-0 的实验, 先试试水, 不知道能做到第几节.

### 成果
1. os1 目录能跑起来, 对于编译器toolchain稍微改了下配置, 否则似乎会因为版本太旧而报错. 但是 github CI 仍然出错 ???  
2. 看到链接器脚本, 结合文档介绍, 大致明白它规定了 kernel 的 entrypoint 和各个段 (SECTION) 的先后顺序及起止地址的内存对齐. 
3. 通过阅读 riscv 手册, 我知道 RustSBI 运行在 Machine 模式, 而 Kernel 仅运行在 Supervisor 模式. 想更深入的了解下 SBI 接口的细节. 文档中说 SBI 提供的服务很少, 例如关机, 显示字符串等.


## day 2
### 计划
1. 做完 rustling
2. step 1 自学risc-v系统结构. 由于risc-v我也有一点基础, 所以不清楚陌生知识会占多大比例. 按照练习要求, 今天的重点放在RV32/64特权架构这部分.

### 成果
1. rustling 做完了
2. 读完《RISC-V手册：一本开源指令集的指南》的第十章. 此章内容总体上已经比较了解了. 在阅读文中的最简中断处理函数的汇编代码后, 感觉对于scratch寄存器的用法清晰了很多.

## day 1
### 计划
1. 已经有一定rust的基础了, 所以快速刷一下rustling的基础题吧

### 成果
1. 竟然完成了 81/84, 看来我比自己想象中要厉害啊, 哈哈哈

