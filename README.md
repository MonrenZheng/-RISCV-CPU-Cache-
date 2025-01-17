# 五级RISCV架构CPU

## 1.整体架构

整了CPU架构是由4个流水级构建的，如果把**regfile** **寄存器堆** 当做是一个流水级的话，那么就是5个级了，也更加能够说通5级流水了。

![image](https://github.com/zmr66z6xx6/-RISCV-CPU-Cache-/assets/124332399/6eb884dc-27ae-4e60-b318-e014c732f39c)



### 五级流水对应5个阶段

CPU设计分成 **IF, ID, EX, MEM, WB**五个模块，整个流水线的执行过程，就发生在这些流水线寄存器中，信号在一级一级之间进行传递。而在流水级之间，大多利用组合逻辑得到相应的数据或者控制信号对流水级的更新进行控制，以确保正确的更新。

**IF阶段**，实现的是PC, instr的更新，或者说是实现**IF/ID流水寄存器**的更新这里使用了双模动态分支预测。相较于其他的实现过程，这里将PC更新模块(时序时钟控制)与IF/ID流水线寄存器更新整合在一起，减少了流水级别(原来是PC更新 + IF/ID流水级寄存器，由于都是时钟触发，相邻之间存在一个时钟的延迟)，并且个人认为这才是实现了真正的5级流水。

**ID阶段**，实现的是将IF/ID流水级寄存器的数据传送到ID/EX流水寄存器，也即是实现**ID/EX寄存器**的更新。要注意的是这里的rs1和rs2寄存器对应的数据的获取，借用了WB模块进行数据的传送。

**EX阶段**，实现ID/EX流水级寄存器的数据传送到EX/MEM流水寄存器，也即是实现**EX/MEM寄存器**的更新。在其更新之前的ALU计算的组合逻辑实现将在第2部分讲述。

**MEM阶段**，实现EX/MEM流水级寄存器的数据传送到MEM/WB流水寄存器，也即是实现**MEM/WB寄存器**的更新。当然我们这一阶段中会将Cache的内容更新到MEM/WB寄存器中，而一旦发生访存错误，整个流水线将会发生停顿。这样也就需要实现MEM/WB流水线寄存器与内存存在数据传递。由于都是时钟触发，因此存在数据的延迟。从另一个角度来看，Cache的miss将会破坏正常的五级流水的进行，因此会发生相应的stall。

**WB阶段**，实现MEM/WB流水寄存器的数据传送到regfile寄存器堆中，也即是**更新寄存器堆**。



## 2.指令实现部分

### 指令类型

指令实现大致分为以下三种类型     

##### 计算指令(R型指令 ,I类型中的立即数运算）      

##### 访存指令(load指令、store指令)

##### 分支跳转(B类型指令、J跳转指令)

以上指令的实现最重要的正确地处理好对控制信号的传递，以及对数据进行正确的筛选与传递以确保计算模块得到正确的结果。此外可能存在数据依赖而发生数据冒险或者产生数据前推的需求(在第3部分讲述)，同时由于访存可能存在Cache失效的情况(Cache部分将在第5部讲述)，会存在访存stall。

### controller模块

由**controller**模块在IF/ID寄存器得到指令后进行更新，而其含有两个子模块分别为**alu_control**以及**main_control**。其中main_control主要负责memread, memwrite, regsrc, regwrite, branch，Immsel, aluop等控制信号的生成，alu_control主要负责alucontrol信号的生成。

### mux2_1模块

数据选择器主要是由于数据通路中后续发生的指令与之前的指令存在数据依赖而**需要数据前推**；不同的指令类型需要不同来源的操作数，比如**rs2**寄存器数值或者**imm**立即数的选择；在ID/EX流水级寄存器更新后根据指令类型或者相应的控制信号进行目标地址计算；数据写回寄存器堆时根据来源进行选择。

### ALU模块

根据输入的数据进行结果的计算，这里会利用 **controller** 模块产生的信号进行正确的结果运算。

### Imm_extend模块

立即数扩展模块，主要根据**controller** 模块产生的 immsel 进行立即数的获取。

### sleft 模块

将得到的立即数进行左移一位为计算目标地址做准备。

### adder 模块

进行目标地址的计算，可以不使用吃模块直接assign实现。

### compare 模块

进行是否跳转的结果计算，最后传出一位的信号**cmp**,需要结合**branch**信号确定跳转情况。



## 3.数据冒险以及数据前推 

数据前推本质也是数据冒险，这是解决停顿带来损失的办法，而 数据冒险在这里特指load-use数据冒险是无法避免的。



### Hazard_detect模块

将IF/ID流水线寄存器的**rs1**以及**rs2**字段与ID/EX流水线寄存器上的**rd**进行比较，并且分析ID/EX中的**memread**信号是否为1，另外这里**容易忽略**的是：针对部分指令没有利用到rs1或者rs2寄存器，需要使用**opcode**字段来解决伪数据冒险发生。

### Forward_detect模块

在ID/EX，EX/MEM，MEM/WB流水级寄存器更新后，利用后两者的**rd**数据以及相应的**regwrite**信号，以及ID/EX的**rs1**以及**rs2**, **opcode**来综合判断出是否发生数据前推，生成相应的数据前推信号**forwardA, forwardB**。

**这里要注意的是**，因为EX/MEM和MEM/WB两者均可能需要数据前推，这是优先考虑EX/MEM流水级寄存器，因为这一级的历史是较近的，或者说数据是更新的，是正确的结果。参考下列指令

##### add X18 X18 X9

##### addi X18  X19 10

##### add X20 X18 X19



## 4.动态分支预测

动态分支预测的实现是在**IF**阶段实现，并且会根据ID/EX寄存器更新后，组合逻辑得到的**预测正确与否**(**error**)信号，计算得到的真实需要执行的目标地址**dest**，以及是否发生跳转(**jump**)信号进行相应的状态机和预测表的更新。

##### 状态机

状态机是**reg**类型的数组维护的每一条指令地址PC对应的跳转状态，每个周期都跟根据历史预测的信号进行更新。初始化是2'b00，当然，也可以不进行初始化，将2'bxx当成2’b00。

##### 预测表

预测表记录的是每一条指令下一条预测执行的指令，默认为顺序执行，也即是PC+1。在流水过程中会根据数据通路中输入的结果对预测表进行更新，其相应的结果需要与状态机对应的值相符合。（比如状态值为2'b00或者2‘0，那么预测表存储的值一定是对应的PC的下一条指令地址PC+1的）。

**注意：**这里的指令地址都是逐一递增的，但是在数据通路的相邻的指令地址是相差4的或者说是**mod4**为0的,因此**IF**阶段和数据通路之间进行数据传输时的**4倍关系**需要处理好。

##### 指令存储寄存器

指令存储器相当于是**Cache**了，考虑到程序的指令数量一般不会太长，需要长时间运行的程序大概率是由于循环结构。基于上述原因，我们使用这个ICache就能够很好的完成各种类型程序要求了。

**注意**：Cache存储的是对应PC地址对应的32位指令。



## 5.Cache实现

Cache实现了直接映射，写返回的方式进行。这里由于**block**中的数据是**单字的**，因此在Cache中写数据发生写缺失时，可以将脏页写回内存(脏页存在的情况下)，而不需要访问内存将相应的数据载入Cache中，直接将需要写的数据写入Cache。



Cache的实现利用**reg** 类型数组进行，在程序中，我们实现的Cache有128 lines，我们默认地址是32位的，那么tag字段是23位的(尽管这里设计的block IP的深度是1024，10位地址)。

### 读缺失

一旦发生读数据缺失，需要从内存中获取数据，需要将整个流水线寄存器进行停顿。

停顿两个周期，第一个停顿的周期进行**数据的读取**，第二个周期进行**脏数据的写回**内存(如果存在)。

**注意**：读取数据地址 以及 脏数写回地址是不同的。

### 写缺失 

写缺失一般是写数据是发现已经存在脏数据，那么可以将脏数据写回(如果存在)，并且将需要写入的数据写进Cache。



**写在后面**，为什么IP核读数据一般没有问题，写数据要wea写使能信号为1,还需要ena同时为1是什么个恶心人的东西555555555
