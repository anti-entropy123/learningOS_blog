# learningOS_blog
2022 OS 训练营每日进度记录. 
慢慢来, 比较快. 不贪多, 力求弄懂眼前遇到的问题. 

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

