What is Computer Architecture?
====================

.. note::

   This part is based on Lecture 2 of CMU447.

Introduction to Computer Architecture
--------

Computer Architecture in Levels of Transformation
~~~~~~~~

- Problem
- Algorithm
- Program/Language
- Runtime System (VM, OS, MM)
- **ISA**
- Microarchitecture
- Logic
- Circuits
- Devices and Electrons


What is an Algorithm?
~~~~~~~~

An Algorithm is a step-by-step procedure for solving a problem in a finite amount of time.
Each step has three properties:
	- Definite (precisely defined)
	- Effectively computable (by a computer)
	- Terminates

What is a Computer?
~~~~~~~~

A Computer has three key components:
	- computation: control and datapath
	- communication: I/O
	- storage

**The Von Neumann Model/Architecture** of a computer has two key properties:
	- Stored Program
		- instructions stored in a linear memory array
		- memory is **unified** between instructions and data
	- Sequential instruction processing
		- one instruction processed at a time
		- **program counter** identifies the current instruction
		- program counter is advanced sequencially except for control transfer instructions

**The Dataflow Model** of a computer
	* An instruction is fetched and executed in **data flow order**
	* In a data flow machine, a program consists of data flow nodes. A data flow node fires when all its inputs are ready.

What is Computer Architecture?
~~~~~~~~

**ISA + implementation definition**:
The science and art of designing, selecting, and interconnecting hardware components and designing the hardware/software interface to create a computing system that meets functional, performance, energy consumption, cost and other specific goals.

**Traditional (ISA-only) definition**:
The term _architecture_ is used to describe the attributes of a system as seen by the programmer, i.e., the conceptual structure and functional behavior as distinct from the organization of the dataflow and controls, the logic design, and the physical implementation.

ISA vs Microarchitecture
  * Implementation (uarch) can be various as long as it satisfies the specification (ISA)
  * Microarchitecture usually changes faster than ISA


Fundamentals of Instruction Set Architecture (ISA)
--------

Elements of an ISA
~~~~~~~~

* Instruction sequencing model: control flow vs. data flow
* instruction processing style
	* specifies the number of "operands" an instruction "operates" on and how it does so
* Instructions
	* Opcode
	* Operand secifiers (Addressing modes)
* Data types
* Memory organization
	* address space: how much uniquely identifiable locations in memory
	* addressablility: how much data does each uniquely identifiable location store
* Registers
	* **Why registers?** Because programs exhibit a characterstic called **data locality**
	* General purpose registers (GPR)
		* all registers good for all purposes
	* affects:
		* number of bits used for encoding register address
		* number of values kept in fast storage (register file)
		* (uarch) size, access time, power consumption of register file
	* Large number of registers:
		* enables better register allocation by compiler
		* larger instruction size
		* larger resiter file size
* Addressing modes


Different types of computer architecture
~~~~~~~~

* 栈体系结构
  * 操作数位于栈的顶部（TOS）
  * 0-address: op, push A, pop A
  * Advantages
    * small instruction size: simple logic & compact code
    * efficient procedure calls: all parameters on stack
  * Disadvantage
    * computations that are not easily expressible with "postfix notation" are difficult to map to stack machines
    * not flexible
* 累加器体系结构
  * 操作数为隐式的累加器
  * 1-address: op ACC, ld A, st A
* 通用寄存器体系结构
  * 只有显式操作数：寄存器或存储地址
  * 两类不同的寄存器计算机：
    * **寄存器-存储器**体系结构：可以用任意指令来访问存储器
    * **载入-存储**体系结构：只能用载入和存储指令来访问存储器


Different types of addressing modes
~~~~~~~~

Addressing modes specifies how to obtain the operands from the registers/memory.
* 寄存器寻址
	* 值存储在寄存器中
* 立即数寻址 (absolute)
	* use immediate value as address
* 寄存器间接寻址 (register indirect)
	* use $GPR[r_{base}]$ as address
* 位移量寻址 (displaced or based)
	* use $offset +GPR[r_{base}]$ as address
* 索引寻址 (indexed)
	* use $GPR[r_{base}]+GPR[r_{index}]$ as address
* 存储器间接寻址 (memory indirect)
	* use value at $M[GPR[r_{base}]]$ as address
* 自动递增寻址 (auto increment)
	* use $GPR[r_{base}]$ as address, but inc. $GPR[r_{base}]$ each time
* 自动递减寻址 (auto decrement)
* 比例寻址
	* 用于索引数组

Advantages of more addressing modes
  * enables better mapping of high-level constructs to the machine
Disadvantages of more addressing modes
  * higher complexity of the architecture
  * higher CPI
  * more work for the complier
  * more work for the microarchitect

Different types of operations in an ISA
~~~~~~~~

* 算数与逻辑：加减乘除，与或，etc.
* 数据传送：载入-存储
* 控制
  * 跳转：控制中的改变是**无条件**的
  * 分支：控制中的改变是**有条件**的
  * 过程调用与返回
* 系统：操作系统调用、虚拟内存管理指令
* 浮点
* 十进制：十进制加法与乘法、二进制到字符的转换
* 字符串：移动、比较、搜索
* 图形：像素与顶点操作、压缩/解压缩操作

Encoding of an ISA
~~~~~~~~

ISA所支持的各种操作通常在一个称为操作码(Opcode)的字段中指定。指令集编码将寻址方式与操作结合在一起
  * **地址标识符**：说明使用哪种寻址方式来访问操作数
* **定长编码** (Fixed length)：所有指令的大小都相同
  * 当寻址方式和操作数都较少时，效果最好
  * Pros
    * easier to decode single instruction in hardware -> simpler hardware design
    * easier to decode multiple instructions concurrently
  * Cons
    * wasted bits in instructions -> higher energy consumption retrieving operands
    * harder-to-extend ISA
* **变长编码** (Variable length)
  * 几乎允许对所有操作使用所有的取址方式
  * 在存在许多寻址方式和操作时，是最绝佳选择
  * Enables **Uniform Decode** (usually goes with fixed length)
    * same bits in each instruction correspond to the same meaning
    * easier decode, simpler hardware
    * enables parallelism: generate target address before knowing the instruction is a branch
    * restrcits instruction format -> less flexibility
  * Pros
    * compact encoding, e.g. Intel 432: Huffman encoding
  * Cons
    * more logic to decode a single instruction
    * harder to decode multiple instructions concurrently
* Tradeoffs
  * code size vs. hardware complexity
  * ISA extensibility and expressiveness vs. hardware complexity
  * smaller code vs. ease of decode
* Uniform Decode
  * same bits in each instruction correspond to the same meaning