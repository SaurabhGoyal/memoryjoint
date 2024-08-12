+++
title = "Virtual Machine"
date = 2024-08-12
description = "Writing a 16 bit virtual machine in Rust - Taught me more than I expected"

[taxonomies]
tags = ["abstraction", "machine", "processor", "assembly", "virtualization"]

[extra]
toc = true
quick_navigation_buttons = true
+++

# Abstraction
If one thinks about it, abstraction is nothing but the most efficient form of trade contract. In this contract a seller of a service can announce the services it is offering with every single detail captured in it and interested customers can avail it **without having to have a side discussion outside the contract**. Similarly a customer can announce the service it needs and any seller that wants to provide it can contact the seller. *If this sounds too capitalistic, that is because it is capitalistic, every single thing interaction in the world is about capitalism!*.

**Abstraction's beauty is in its completeness (neccessary and sufficient)**. If the contract has been published, no further discussion has to happen between seller and customer (atleast regarding trade terms). What that means is that if the everyone creates their own contracts and people start following some conventions to standardise terms of different kinds of trades, then for that kind of trade any customer can connect with any seller at minimum possible effort. This results in more number of experiments around how to provide service for a certain trade type, which further fuels higher quality services at a much faster rate of availability. **This is why we need abstractions.**

Abstraction is most highlighted when working with computer systems. Everything is an abstraction over an abstraction over an abstraction and so on. In such systems, one needs to limit the layers of abstraction one dabbles into, as one can't possibly work on or understand every abstraction layer. Carl Sagan famously said that ***"If you want to make a sandwitch from scratch, you need to first make a universe"***, so focus on the nitty gritties of what abstraction you have been given and what abstraction you are trying to provide by building on top of it.

# Virtual Machine
Do you think computer is a complex machine! It surely is but its core design is so simple that one can intuitively comprehend it and maybe even come up with it with some brainstorming. This is the revelation I had when I looked at the interface and the state of a virtual machine.

As a programmer, I deal with some or all of the following things on a regular basis. Writing a simple VM helped me understand the foundation and reasoning behind each one them -
- Memory allocation and deallocation for storing intermediate results.
- Input Output.
- Debugger

*Related post - [Digital Machine Layer by Layer](../digital-machine-layer-by-layer)*

# Implementation
**(Checkout code at [https://github.com/SaurabhGoyal/vmrs/](https://github.com/SaurabhGoyal/vmrs/))**

## What value and API does a VM provide?
- A virtual machine is a mock for a real machine, in our case a CPU.
- The main value that CPU provides is to run instructions.
- This implementation is inspired by LC-3 architecture of a CPU to build the virtual machine BUT would not implement or follow it completely. 
- To run the machine, a program code has to be provided which is nothing but an array of instructions.
- The instructions themselves need to be in machine code, each instruction being a 16-bit value.
- The machine will simply execute and step through each instruction of the program one at a time.
- The instruction-set would be of Assembly language as it is smaller and easier to implement.

## How does it work internally?
- High level -
    - Machine needs to do take data, take instructions and do their calculations and produce results.
    - Machine needs a way for taking and storing the data, instructions and results. This is where **memory** comes.
    - Machine needs to perform calculations for instructions, that calculation happens in the logic gates implemented in the hardware of the machine and is encapsulated in the form of **registers**.
    - Data need to be sent to registers from memory and register is instructed to perform the operation.
    - For dealing with dynamic usecases such as IO devices, machine either uses dedicated registers (special purpose registers) or dedicated memory (memory mapped registers).
- Machine has two components -
    - Registers - For control purpose. Each register can hold a 16 bit value. There are 10 registers in this implementation -
        - R0-R7 - value storage during instruction execution
        - RPC - program counter for machine to track which instruction to be executed
        - RCOND - value storage of previous instructions for conditional instructions
    - Memory - For data storage purpose only. This provides a larger storage area than registers and is used purely for storage of data that can not be fit into storage registers (R0-R7). Each memory slot can also hold a 16-bit value.
- Machine exposes two APIs -
    - `Load` - any arbitrary data in the memory. Programmer should use it to load data and program code into memory at desired addresses. (This is programmer's responsibility that the addresses of the data in the program code point correctly to the loaded data in memory.)
    - `Run` - run the program code stored at given address. Machine sets `RPC` to that address and starts execution.
- Machine keeps loading the instruction at the memory address referred by `RPC` and executing the business logic of that instruction. If the instruction is `OP_TRAP` with `TRAP_HALT`, machine exits the program.
- Machine also provides a virtual component called **"Memory Mapped Registers"** which is nothing but memory area which has is used for control purpose and not just data storage purpose. The main usecase is for handling IO from devices, since they can be dynamic, machine doesn't map registers to their signals and uses memory instead as memory is more dispensable than registers.

## Doubts
- **Why to model registers as unsigned ints and then handle the negative numbers manually in VM logic instead of modelling them as signed ints only?**
    - Because registers in hardware are simple bit storage devices and do NOT care about the data they hold, i.e. they do not have a direct understanding of numbers, let alone positive or negative. This also keeps the hardware API simpler for different users to build whatever logic they want to build on top of a bit array register.
    - Ref - https://stackoverflow.com/a/27207704/2555504
- **Why do we need Program-Counter register (`RPC`)?**
    - To support non-linear execution of code which is powered by `go-to / jump` statement enabling connstructs such as `if-else` and `loop`.
- **Why do we need a dedicated register (`RSTAT`) for maintaining the sign of the result of previous instruction when the same can be checked from the result itself?**
    - `RSTAT` is a dedicated register for a quick lookup of multiple things such as sign of last result (+ve / -ve), status of last operation (underflow / overflow), augmented information of last result (carry) and various interrupts. While the sign can be directly checked from result, the check is mostly conducted in some kind of branching decision context which is where status register provides information in generic sense.
    - This register has been named as `RCOND` in the referring blog post and is also called as `Condition Code Register` or simply `Condition Register` sometimes.
- ~~**We are using a dedicated op-code for not treating an instruction as operation, for storing raw data in memory. This wastes 4 bits, is there any workaround?**~~
    - Using the two step (load, run) process now instead of the single step (run-with-load), resolving this issue.
- **Why is an address needed to load the program code? While I haven't used any custom address, i.e. loaded the program code simply at 0th address, the blog post suggest to use 0x3000, why?**
    - The reason is simply that in real world, machine may have more things that it needs to manage in the memory other than just the program code to be executed. One such thing is trap routine code which is nothing but some special instructions that machine itself has hardcoded to provide functionalities such as talking to IO devices and halt the program. 

# Instruction-Set
## General syntax
`OP_CODE - (4 bits) | OPERANDS (12 bits)` 

## Commands
### Data (OpCode - 1110)
- Stores given value in memory.
- Syntax - `1110 (4 bits) | Data (12 bits)`

### ADD (OpCode - 0001)
- Adds two values and puts the result into a destination register. The values van be either fetched from registers or given in-place.
- Syntax - `0001 (4 bits) | Dest Register (3 bits) | Source-Register-1 (3 bits) | Mode (1 bit) | Mode-Specific-Operands (5 bits)`
- Two modes
    - Register mode (Mode bit 0) - When second operand data is in another source register.
    - Syntax - `0001 (4 bits) | Dest Register (3 bits) | Source-Register-1 (3 bits) | 0 (1 bit) | 00 (2 bits) | Source-Register-2 (3 bits)`
    - Immediate mode (Mode bit 1) - When second operand data is in the instruction itself.
    - Syntax - `0001 (4 bits) | Dest Register (3 bits) | Source-Register-1 (3 bits) | 1 (1 bit) | Sign-of-Value (1 bit) | Number-of-Value (4 bits)`

### Load (OpCode - 0010)
- Loads the value given in the instruction to the destination register for later use.
- Syntax - `0010 (4 bits) | Dest Register (3 bits) | 0 (1 bit) | Sign-of-Value (1 bit) | Number-of-Value (7 bits)`

### LoadIndirect (OpCode - 0011)
- Loads the value at the memory address given in the instruction to the destination register for later use.
- Important thing to note is that the address will be relative to the program code instructions and not absolute. This means that the relative address can be negative and should be sign-extended. 
- Syntax - `0011 (4 bits) | Dest Register (3 bits) | Relative-Memory-Address (9 bits)`

### LoadRegister (OpCode - 0110)
- Loads the value stored in the source register given in the instruction to the destination register for later use.
- Syntax - `0001 (4 bits) | Dest Register (3 bits) | Source-Register (3 bits) | 000000 (6 bits)`

### Jump (OpCode - 0100)
- Sets program counter to the memory address given in the instruction.
- Enables non-liner execution of program.
- Syntax - `0100 (4 bits) | 000 (3 bits) | Relative-Memory-Address (9 bits)`

### JumpIfSign (OpCode - 0101)
- Sets program counter to the memory address given in the instruction if the register given in instruction has negative value.
- Enables non-liner execution of program.
- Syntax - `0101 (4 bits) | Dest Register (3 bits) | Relative-Memory-Address (9 bits)`

### Trap (OpCode - 1111)
- Sets a trap to the instruction execution for machine to do things outside the instruction in the program code.
- These things can be things such as talking to IO decices or halting the program. Simple way to imagine is that this is a set of machine defined functionalities to interact with outside the program-code and machine scope.
- Since these things are something that machine implements on its own and are not part of user-defined instructions, machine implements them itself and stores the implementation logic in memory. This is the reason, machine may need some part of its memory for its own things and program code would be stored at a non-zero address, typically 0x3000.
- **NOTE** - This memory address is needed mostly for the real hardware where the instructions for handling a trap code are written in machine code (translated from assembly) and stored in the memory. For our case (VM), we would implement them in the Rust runtime only so no dedicated memory address is needed.
- Because of multiple types of traps and their distinct nature of being a foreign-function-interface instruction, they have been categorised into one opcode where the trap type can be passed as an operand, instead of creating a dedicated opcode for each trap. https://www.jmeiners.com/lc3-vm/#trap-routines
    > Trap for additional functionalities You may be wondering why the trap codes are not included in the instructions. This is because they do not actually introduce any new functionality to the LC-3, they just provide a convenient way to perform a task (similar to OS system calls). In the official LC-3 simulator, trap routines are written in assembly. When a trap code is called, the PC is moved to that code’s address. The CPU executes the procedure’s instructions, and when it is complete, the PC is reset to the location following the initial call.

    > Note: This is why programs start at address 0x3000 instead of 0x0. The lower addresses are left empty to leave space for the trap routine code.
- Syntax - `1111 (4 bits) | 0000 (4 bits) | TrapCode (8 bits)`
- Trap always works with R0 only, i.e. if it reads character, it would store in R0, if it prints character, it would load value from R0.
- Trap codes -
    - Halt (0) - Stops the program execution.
    - GetC (1) - Blocks the program to get one character from keyboard and sets it it R0.


# References
- [https://en.wikipedia.org/wiki/Little_Computer_3](https://en.wikipedia.org/wiki/Little_Computer_3)
- [https://en.wikipedia.org/wiki/Little_man_computer](https://en.wikipedia.org/wiki/Little_man_computer)
- [https://www.jmeiners.com/lc3-vm/#:lc3.c_2](https://www.jmeiners.com/lc3-vm/#:lc3.c_2)
- [https://www.youtube.com/watch?v=oArXOAhzOdY&list=PLUkZG7_4JtUL22HycWYR_J-1xJo7rQGhr](https://www.youtube.com/watch?v=oArXOAhzOdY&list=PLUkZG7_4JtUL22HycWYR_J-1xJo7rQGhr)
- [https://www.andreinc.net/2021/12/01/writing-a-simple-vm-in-less-than-125-lines-of-c](https://www.andreinc.net/2021/12/01/writing-a-simple-vm-in-less-than-125-lines-of-c)
