# CPU Pipelining

!!! info "本节内容"
    本节内容主要来自 CMU 447 第 7、8 讲，以及《计算机体系结构：量化研究方法》附录 C：流水线基础与中级概念。

## Pipelining: Basic Idea

* divide the instruction processing style into distinct stages of processing
* ensure that there are enough hardware resources to process one instruction in each stage
* process a different instruction in each stage
* thus **increasing instruction throughput**

### An Ideal Pipeline

* increase throughput with little increase in cost (hardware cost)
* repetition of identical operations
* repetition of independent operations
* uniformly partitionable suboperations

### Instruction Pipeline: Not an Ideal Pipeline

* diffrent instructions -> not all instructions need the same stages
  * forcing different instructions to go through the same pipeline stages
  * external fragmentation: some pipeline stages idle for some instructions
* different pipeline stages -> not the same latency
  * need to force each stage to be controlled by the same clock
  * internal fragmentation (some pipeline stages are too fast but all take the same clock cycle)
* instructions are not independent of each other
  * need to detect and resolve inter-instruction dependencies to ensure the pipeline provides correct values -> pipeline stalls

## 引言

### RISC-V 指令集基础知识

* 载入-存储体系结构 ISA
* 所有数据操作都是对寄存器中数据的操作，通常会改变整个寄存器
* 只有载入和存储操作会影响存储器
* 格式指令相对固定，所有指令位宽通常相同

### RISC 指令集的简单实现

* 关注 RISC 体系结构中的一个整数子集的流水线，包括：载入-存储字、分支和整数 ALU 操作
* RISC 子集的每条指令都可以在最多5个时钟周期内实现：
  1. 取指周期 (Instruction Fetch, IF)
	 * 将程序计数器发送到存储器，从存储器中提取当前指令
	 * 向程序计数器 +4（每条指令的长度为四个字节，将程序计数器更新到下一条顺序指令
  2. 指令译码/读寄存器周期 (Instruction decode and register operand fetch, ID/RF)
	 * 对指令译码，并且从寄存器堆中读取与寄存器源说明符相对应的寄存器
	 * 在读取寄存器时对其进行相等测试，以确定可能的分支
	 * 在后续需要时，对指令的偏移量字段进行符号扩展，并增加到程序计数器上，计算出可能的分支目标地址
	 * 指令译码与寄存器的读取的并行执行的，因为 RISC 体系结构中，**寄存器说明符位于固定位置**，这一技术称为**固定字段译码**
	 * 对于载入和 ALU 指令的立即数操作，立即数字段总是在相同的位置，从而可以进行符号扩展
  3. 执行/有效地址周期 (Execute/Evaluate memory address, EX/AG)
	 * ALU 对上一周其中准备的操作数进行操作，根据指令类型执行四种操作之一
	   1. 存储器访问——ALU 将基址寄存器和偏移量加到一起，形成有效地址
	   2. 寄存器-寄存器 ALU 指令——ALU 对读自寄存器堆的值执行由 ALU 操作码指定的操作
	   3. 寄存器-立即数 ALU 指令——ALU 对读自寄存器堆的第一个值和符号扩展立即数执行由 ALU 操作码指定的操作
	   4. 条件分支——判断条件是否为真
	 * 在载入-存储体系结构中，有效地址与执行周期可以合并到一个时钟周期当中，这是因为没有指令需要同时计算数据地址并对数据执行操作
  4. 存储器访问 (Memory operand fetch, MEM)
  5. 写回周期 (Store/Writeback result, WB)

### RISC 处理器的经典五级流水线实现

* 使用分离的指令存储器与数据存储器，通常用分离的指令缓存和数据缓存实现
* 在两个阶段都使用了寄存器堆：在 ID 中进行读取、在 WB 中进行写入
* 流水化的基本性能问题
  * 流水线产生延迟
  * 流水线各级之间失衡
  * 流水线引入额外的开销
  * 流水线中的指令并非相互独立的，可能存在依赖关系


## 流水线的主要障碍——流水线冒险

* 有一些称为冒险的情形，阻止指令流中的下一条指令在自己指定的时钟周期内执行，从而导致流水线停顿
* 冒险降低了流水化所获得的理想加速比的性能
* 三种类型的冒险：**结构冒险**、**数据冒险**、**控制冒险**

### 带有停顿的流水线性能

* $流水化加速比=\frac{非流水化指令平均执行时间}{流水化指令平均执行时间}=\frac{非流水化CPI}{流水化CPI}\times \frac{非流水化时钟周期}{流水化时钟周期}$
* $流水化CPI=1+每条指令的流水线停顿时钟周期$
* 如果忽略流水话的周期时间开销，并假定流水级之间达到完美平衡，则两个处理器的周期时间相等，由此得到：
$$加速比=\frac{非流水化CPI}{1+每条指令的流水线停顿周期}$$
* 简单考虑：所有指令的周期数都相同，并且必然等于流水线级数（即**流水线深度**），非流水化 CPI 等于流水线深度，则有：
$$加速比=\frac{流水线深度}{1+每条指令的流水线停顿周期}$$
* 如果没有流水线停顿，则可以得到：**流水化可以使性能提高的倍数为流水线深度**

### 结构冒险

* 在重叠执行模式下，如果硬件无法同时支持指令的所有组合方式，可能会出现资源冒险，从而导致结构冒险
* 可能发生在不太常用的特殊用途功能单元，通常并非性能瓶颈

### 数据冒险 (Data dependence)

* 假设指令i按照程序顺序出现在指令 j 之前，两个指令都使用寄存器 x，可能出现三种冒险：
  * Flow dependence - read after write (RAW)：当指令 j 对寄存器 x 的读取发生在指令 i 对寄存器 x 的写之前，就会发生 RAW 冒险
  * Anti dependence - write after read (WAR)：当指令 i 对寄存器 x 的读取发生在指令 j 对寄存器 x 的写入之后，这时指令 i 会使用错误的 x 值
  * Output dependence - write after write (WAW)：当指令 i 对寄存器 x 的写入发生在指令 j 对寄存器 x 的写入之后时，寄存器 x 会传送错误的值  
* For all of them, we need to ensure semantics of the program is correct.
* Flow dependences always need to be obeyed because they constitute true dependence on a value.
* Anti and output dependences exist due to limited number of architectural registers. They depend on a name, not a value.

#### Five fundamental ways of handling flow dependences

* detect and wait until value is available in register file
* detect and **forward/bypass** data to dependent instruction
  * 来自 EX/MEM 和 MEM/WB 流水线寄存器的 ALU 结果总是被反馈回 ALU 的输入端
  * 如果**前递硬件**检测到前一个 ALU 操作已经对当前 ALU 操作的源寄存器进行了写操作，则控制逻辑选择前递结果作为 ALU 输入，而不是选择从寄存器堆中读取的值
  * ALU 输入既可以使用来自相同流水线寄存器的前递输入，也可以使用来自不同流水线寄存器的前递输入
  * 可以将前递技术加以推广，将结果直接传送给所需的功能单元
* detect and eliminate the dependence at the software level
* predict the needed values, execute "speculatively", and verify
* do something else (fine-grained multithreading), therefore no need to detect

#### Interlocking

* 流水线互锁会检测冒险，并**使流水线停顿**，直到该冲突被清除
* 在这种情况下，互锁使得流水线停顿，让希望使用某一数据的指令等待，直到源指令生成该数据为止

#### Approaches to Dependence Detection

* **Scoreboarding**
  * each register in register file has a _Valid bit_ associated with it
  * an instruction that is writing to the register resets the valid bit
  * an instruction in Decode stage checks if all its source and destination registers are valid, and stall the instruction if not valid
  * Pro: simple 1 bit register
  * Con: need to stall for all types of dependences
* **Combinational dependence check logic**
  * special logic that checks if any instruction in later stages is supposed to write to any source register of the instruction that is being decoded
  * Pro: no need to stall on anti and output dependences
  * Con: logic is more complex

### 控制冒险 (Control dependence)

* Data dependence on the Instruction Pointer/Program Counter
* All instructions are dependent on previous ones
* If the fetched instruction is a **non-control-flow instruction**
  * next fetch PC is the address of the next sequential instruction
  * easy to determine if we know the size of the fetched instruction
* If the instruction that is fetched is a **control-flow instruction**
  * need to determine the next Fetch PC
* 在执行一个分支后，更新的程序计数器的值可能等于（也可能不等于）当前值加 4.
  * 如果分支将程序计数器地址改为其目标地址，则它**选中分支**
  * 如果程序计数器的值依然为当前值加4，则它**未选中分支**

#### Fundamental Ways of Handling Control Dependences

* **Stall** the pipeline until we know the next fetch address
  * wait for the true-dependence on PC to resolve
* **Branch Prediction**: Guess the next fetch address
  * One method: always guess NextPC = PC + 4 -> a form of _next fetch address prediction_
	* Idea: Maximize the chances that the next sequential instruction is the next instruction to be executed
	  * Software level: **Profile guided code positioning**: lay out the control graph such that the "likely next instruction is on the not-taken path of a branch
	  * Hardware level: Trace cache
	* Idea: Get rid of control flow instructions/minimize their occurrence
	  * Get rid of unnecessary control flow instructions -> **predicate combining**
		* combine predicate operations to feed a single branch instruction instead of having one branche for each
		* use **condition registers** to store and operate on predicates
		* Pro: fewer branches in code
		* Con: possibly unnecessary work
	  * Convert control dependences into data dependences -> **predicated execution**
		* Pro: Always-not-taken prediction works better; compiler has more freedom to optimize the code
		* Con: Useless work: some instructions fetched/executed but discarded
		* requires additional ISA support
  * Idea: Predict the next fetch address
  Requires three things to be predicted at fetch stage:
	* Whether the fetched instruction is a branch
	  * can be accomplished using a BTB
	* Conditional branch direction
	* Branch target address (if taken)
	  * Observation: target address remains the same for a conditional direct branch across dynamic instances
	  * Idea: store the target address from previous instance and access it with PC
	  * called **Branch Target Buffer (BTB)**, or Branch Target Address Cache
* **Branch Delay Slot**: Employ delayed branching
  * Change the semantics of a branch instruction: Branch after N instructions, branch after N cycles
  * Idea: Delay the execution of a branch. N instructions that come after the branch are always executed regardless of branch direction
  * Problem: Find instructions to fill the delay slots: branch must be independent of delay slot instructions, otherwise fix-up code is needed
  * Fancy delayed branching: _Delayed branch with squashing_, if the branch is not taken, the delay slot instruction is not executed
  * Advantages: Keeps the pipeline full with useful instructions in a simple way assuming that:
	* number of delay slots == number of instructions to keep the pipeline full before the branch resolves
	* all delay slots can be filled with useful instructions
  * Disadvantages:
	* Not easy to fill the delay slots
	* Ties ISA semantics to hardware implementation: could hinder future microarchitecture development
* **Fine-grained Multithreading**
  * Idea: Hardware has multiple thread contexts. Each cycle, fetch engine fetches from a different thread
	* By the time the fetched branch resolves, no instruction is fetched from the same thread
	* Branch/Instruction resolution latency overlapped with execution of other threads' instructions
  * Pro:
	* No logic needed for handling control and data dependences within a thread
	* Improved system throughput, latency tolerance, utilization
  * Con:
	* Single thread performance suffers
	* Extra logic for keeping contexts
	* Does not overlap latency if not enough threads to cover the whole pipeline
	* Resource contention between threads in caches and memory
* **Predicated Execution**: Eliminate control-flow instructions
  * Idea: Compiler converts control dependence into data dependence -> branch is eliminated
  * Each instruction has a predicate bit set based on the predicate computation
  * Only instructions with TRUE predicates are committed (others turned into NOPs)
  * Advantages:
	* Eliminates mispredications for hard-to-predict branches
	* Enables code optimizations hindered by the control dependency
  * Disadvantages:
	* Causes useless work for branches that are easy to predict
	* Additional hardware and ISA support
	* Cannot eliminate all hard to predict branches
* **Multipath Execution**: Fetch from both possible paths
  * Idea: execute both paths after a conditional branch
  * Advantage: improves performance if misprediction cost > useless work; no ISA changes needed
  * Disadvantage: potentially growing paths; more complex hardware

#### Static Branch Prediction

* Always not-taken
  * Simple to implement: no need for BTB
  * Low accuracy ~30%-40%
* Always taken
  * No direction prediction
  * Better accuracy: 60%-70%
* Backward taken, forward not taken (BTFN)
* Profile-based
  * Idea: Compiler determines likely direction for each branch using a profile run -> encodes that direction as a hint bit in the branch instruction format
  * Pro: per branch prediction
  * Con: requires hint bits in the branch instruction format
  * accuracy depends on dynamic branch behaviour
* Program-based
  * Idea: use heuristics based on program analysis to determine statcially-predicted direction
* Programmer-based
  * Idea: Programmer provides the statically-predicted direction
  * Via _pragmas_ in the programming language that qualify a branch as likely-taken versus likely-not-taken
* All previous techniques can be combined
* Disadvantages of these techniques:
  * Cannot adapt to dynamic changes in branch behaviour

#### Dynamic Branch Prediction (hardware-based)

* Last time predictor
  * Single bit per branch (stored in BTB)
  * Indicats which direction branch went last time it executed
  * Downside: always mispredicts the last iteration and the first iteration of a loop branch
	* Accuracy for a loop with N iterations = (N-2)/N
  * Solution: add hysteresis to the predictor so taht prediction does not change on a single different outcome
* Two-Bit Counter Based Prediction
  * Each branch associated with a two-bit counter
  * One more bit provides _hysteresis_
  * Accuracy for a loop with N iterations = (N-1)/N
  * Downside: More hardware cost
* Global Branch Correlation
  * Recently executed branch outcome in the execution path is correlated with the outcome of the next branch
  * Idea: Associate branch outcomes with "global T/NT history" of an branches
  * Implementation:
	* Keep track of the "global T/NT history" of all branches in a register -> **Global History Register (GHR)**
	* Use GHR to index into a table that recorded the outcome that was seen for each GHR value in the recent past -> **Pattern History Table**
  * **Two Level Global Branch Prediction**
	* First level: Global branch history register (N bits) -> the direction of last N branches
	* Second level: Table of saturating counters for each history entry -> the direction the branch took the last time the same history was seen
  * Improving Global Predictor Accuracy
	* Add more context information to the global predictor to take into account which branch is being predicted
	* _Gshare predictor_: GHR hashed with the Branch PC
* Local Branch Correlation
* Idea: Have a per-branch history register
  * Associated the predicted outcome of a branch with "T/NT history" of the same branch
  * Make a prediction based on the outcome of the branch the last time the same local branch history was encountered
  * Uses two level of history (Per-branch history register + history at that history register value )
	* Fisrt level: a set of local history registers
	* Second level: table of saturating counters for each history entry
* Hybrid Branch Predictors
  * Idea: use more than one type of predictor (i.e., multiple algorithms) and select the best prediction
  * Advantages: better accuracy, reduced warmup time
* Biased Branches
  * Observation: many branches are biaesd in one direction
  * Problem: these branches _pollute_ the branch prediction structures
  * Solution: detect such biased branches, and predict them with a simpler predictor


## Pipelining and Precise Exceptions: Preserving Sequential Semantics

### Multi-cycle Execution

* Not all instrutions take the same amount of time for execution
* Idea: Have multiple different functional units that take different number of cycles
  * Can be pipelined or not pipelined
  * Can let independent instructions start execution on a different functional unit before a previous long-latency instruction finishes execution

#### Exceptions vs. Interrupts

* Cause
  * Exceptions: internal to the running thread
  * Interrupts: external to the running thread
* When to handle
  * Exceptions: when detected and known to be non-speculative
  * Interrupts: when convenient

#### Precise Exceptions and Interrupts

The architectural state should be consistent when the exception/interrupt is ready to be handled.

1. All previous instructions should be completely retired.
2. No later instruction should be retired.

* Why Precise Exceptions?
  * Semantics of the von Neumann model ISA specifies it
  * Aids software debugging
  * Enables easy recovery from exceptions, e.g. page faults
  * Enables easily restartable processes
  * Enables traps into software (e.g. software implemented opcodes)

### Ensuring Precise Exceptions in Pipelining

#### Reorder Buffer (ROB)

* Idea: Complete instructions **out-of-order**, but reorder them before making results visible to architectural state（允许指令乱序执行，强制它们**顺序**提交）
  * When instruction is decoded it reserves an entry in the ROB
  * When instruction completes, it writes result into ROB entry
  * When instruction oldest in ROB and it has completed without exceptions, its result moved to register file or memory
* 重排序缓冲区 (ROB) 保存已经完成执行但还没有提交的指令结果
* An ROB entry consists of:
  * DestRegID, DestRegVal
  * StoreAddr, StoreData
  * PC
  * Valid bits for reg/data and control bits
  * Exc?
* Problem: results first written to ROB, then to register file at commit time, what if a later operation needs a value in the ROB?
  * Context addressable memory (CAM): Search the reorder buffer in a particular order to find the wanted register value (very complex when ROB becomes bigger, could become the critical path)
  * Use **indirection**
	* access register file first
	  * if register not valid, register file stores the ID of the ROB entry that contains (or will contain) the value of the register
	  * practically the **mapping of the register to a ROB entry**: register file maps the register to a ROB entry if there is an in-flight instruction writing to the register
	* access reorder buffer next
* Important: **Register Renaming** with a Reorder Buffer
  * Output and anti dependences are not true dependences. They exist due to lack of register IDs in the ISA
  * The register ID is renamed to the reorder buffer entry that will hold the register's value
	* Register ID -> ROB ID
	* Architectural register ID -> Physical register ID
* Reorder Buffer Storage Cost
  * Idea: Reduce ROB entry storage by specializing for different instruction types
* In-Order Pipeline with Reorder Buffer
  * Decode (D): Access regfile/ROB, allocate entry in ROB, check if instruction can execute, if so **dispatch** execution
  * Execute (E): Instructions can complete out-of-order
  * Completion (R):Write result to reorder buffer
  * Retirement/Commit (W): Check for exceptions, if none, write result to regfile/memory; else, flush pipeline and start from exception handler
  * **In-order dispatch/execution, out-of-order completion, in-order retirement**
* Reorder Buffer Tradeoffs
  * Advantages:
	* Comceptually simple for supporting precise exceptions
	* Can eliminate false dependences
  * Disadvantages:
	* Reorder buffer needs to be accessed to get the results that are yet written to the regfile
	  * CAM or indirection
  * Other solutions to eliminate the disadvantages: history buffer, future file, checkpointing

#### History Buffer (HB)

* Idea: Update the register file when instruction completes, but **undo updates** when an exception occurs
* When instruction is decoded, it reserves an HB entry
* When the instruction completes, it stores the old value of its destination in the HB
* When instruction is oldest and no exceptions/interrupts, the HB entry is discarded
* When instruction is oldest and an exception needs to be handled, old values in the HB are written back into the architectural state from tail to head
* Advantage:
  * Register file contains up-to-date values for incoming instructions
* Disadvantage:
  * Need to read the old value of the destination register
  * Need to unwind the history buffer upon an exception -> increased exception/interrupt handling latency
  
#### Future File (FF) + ROB

* Idea: Keep two register files (speculative and architecutural)
  * Arch reg file: Updated in program order for precise exceptions
  * Future reg file: Updated as soon as an instruction completes(if the instruction is the youngest one to write to a register)
* Future file is used for fast access to latest register values (speculative state) -> Frontend register file
* Architectural file is used for state recovery on exceptions (architectural state) -> Backend register file
* Advantage: no need to read new values from ROB or the old value of the destination register
* Disadvantage:
  * multiple register files -> more complex design
  * need to copy arch. reg. file to future file on an exception
* In-Order Pipeline with Future File and ROB
  * Decode (D): Access future file, allocate entry in ROB, check if instruction can execute, if so **dispatch** instruction
  * Execute (E): Instructions can complete out-of-order
  * Completion (R): Write result to reorder buffer and future file
  * Retirement/Commit (W): Check for exceptions; if none, write result to architectural register file or memory; else, flush pipeline, copy architectural file to future file, and start from exception handler
  * **In-order dispatch/execution, out-of-order completion, in-order retirement**
* Reduce the Overhead of Two Register Files
  * Use indirection, i.e. pointers to data in frontend and retirement
  * Have a single storage that stores register values
  * Keep two register maps(speculative and architectural); also called register alias tables (RATs)

### Pipelining Issues: Branch Mispredictions

* A branch misprediction resembles an "exception"
* How to do branch misprediction recovery? -> similar to exception handling except that it can be initialted before the branch becomes the oldest instruction in the ROB
* Branch mispredictions are more common than exceptions
* Improving Branch State Recovery Latency
  * Goal: Restore the frontend state(future file) such that the correct next instruction after the branch can execute right away after the branch misprediction is resolved
  * Idea: Checkpoint the frontend register state/map at the time a branch is decoded and keep the checkpointed state updated with results of instructions older than the branch

#### Checkpointing

* When a branch is decoded
Make a copy of the future file and associate it with the branch
* When an instruction produces a register value
All future file/map checkpoints that are younger than the instruction are updated with the value
* When a branch misprediction is detected
  * Restore the checkpointed future file/map for the mispredicted branch when the branch misprediction is resolved
  * Flush instructions in pipeline younger than the branch
  * Deallocate checkpoints younger than the branch
* Advantages: correct frontend register state available right after checkpoint restoration -> reduced state recovery latency
* Disadvantages: Storage overhead; complexity in managing checkpoints
