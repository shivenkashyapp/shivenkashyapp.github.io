---
title: Anatomy of a function call
tags: [C, Assembly, Low-level]
desc: Let's examine the anatomy of a function.
layout: post
---
<!-- more --> 


**PS:** to keep the main content concise, the technical details relating to some concepts are mentioned as [Footnotes](#footnotes).
<hr>
<br>

# The Program Counter (`PC`)
A Program Counter or instruction pointer in context of [x86 systems](https://en.wikipedia.org/wiki/X86) is a register that holds the address of an instruction to be executed.
In a simple environment, the instructions (which are stored in RAM) are "fetched" sequentially.
[Control Transfer](https://docs.oracle.com/cd/E19120-01/open.solaris/817-5477/eoizl/index.html) instructions however, can change this sequence by placing a new value in the PC.
Even the return value<a href="#function-call-cycle"><sup>[ref]</sup></a> from a function is a *control transfer* instruction. Other examples include the `jump` instruction.
`goto` in C behaves analogous to `jump`, and the compiler might generate `jump` calls in the assembler as well. These control transfer instructions can branch off execution. In multi-core and multi-threader processors, each core or thread has its own PC. This means that each core/thread can execute a different instruction sequence, independently of others. In SIMD applications, each parallel computation unit performs the exact same instruction, just with different data.
<br/><br/>
## Synchronization
The program counter is very important for synchronization. For instance, locks and mutexes prevent concurrent accesses to shared resources, and thus the program counter for parellel units 
must be coordinated carefully across the different threads to ensure mutual exclusion. Say, the PC gets stuck in a waiting state, unable to progress, because each thread is waiting for
a resource held by another thread, causing a circular wait. This is a deadlock. A livelock is caused when the PC's of threads involved are continuously changing state without making any progress. It is a risk with algorithms that recover from a deadlock. When changing states, if more than one process takes action, a deadlock detection algorithm gets triggered repeatedly. Read more about synchronization problems from [this article](https://web.cs.wpi.edu/~cs3013/c07/lectures/Section06-Sync.pdf) by Jerry Breecher.
<br/><br/>

# Instruction cycle
It is a fundamental sequence of steps that a CPU performs to execute a single operation. Simply put, it is a fetch $\rightarrow$ decode $\rightarrow$ execute cycle, 
and it looks something like this:
<br/><br/>

1. It starts with a fetch instruction, which retrieves the next instruction to be executed from the Program Counter<sup>[1]</sup>.
2. The [Control Unit](https://en.wikipedia.org/wiki/Control_unit) interprets the [instructions](https://en.wikipedia.org/wiki/Opcode) via a [decoder](https://www.sciencedirect.com/topics/engineering/instruction-decoder).
3. The ALU executes the arithmetic and logic operations, by reading the operands from registers or from memory.
<br/>
Some intermediate read/write phases might also be involved. 
<br/>

# Function call cycle
When a function is called, a stack frame is created, which includes the return address, function arguments and the variables local to the function.
The return address specifies where control should return after the function call completes.
<br/>
A simplified breakdown of a function call looks something like this:
<br/>
1. Arguments of the function are placed on the stack.
2. Some memory is allocated for the return value from the function.
3. A platform specific `call` instruction is executed. This places the **return address** of the function is placed onto the stack.
The *program counter* jumps to this location - because of which the control is transferred to this function.
4. The function reads the arguments from the stack and the code in the function body is run.
5. The `ret` instruction is used to grab the return value(s) from the function. This instruction pops the return address from the stack and thus control returns back to the caller. 
<br/>
<br/>

# Disassembling
Consider a simple C program:
```c
int add(int a, int b) { 
    return a + b; 
}

int main() {
    int result = add(123, 456);
    return 0;
}
```
<br/>

To get the assembler output that the GCC compiler generates:
```bash
$ gcc -S -fverbose-asm -o asm.s add.c
```
<br/>

Let us examine the assembler output:
```nasm
    movl    $456, %esi,
    movl    $123, %edi,
    call    add
```
<br/>

First, `456` is moved into `esi`, which stores the first integer argument, and `edi` stores the second. This is the first couple of steps where the function arguments are placed on the stack.
As I explained earlier, `call add` will now pass control to the function body, which looks like this:
<br/><br/>
```nasm
add:
.LFB0:
    .cfi_startproc
    pushq   %rbp    #
    .cfi_def_cfa_offset 16
    .cfi_offset 6, -16
    movq    %rsp, %rbp  #,
    .cfi_def_cfa_register 6
    movl    %edi, -4(%rbp)  # a, a
    movl    %esi, -8(%rbp)  # b, b
    movl    -4(%rbp), %edx  # a, tmp100
    movl    -8(%rbp), %eax  # b, tmp101
    addl    %edx, %eax  # tmp100, _3
    popq    %rbp
    .cfi_def_cfa 7, 8
    ret
    .cfi_endproc
.LFE0:
    .size   add, .-add
    .globl  main
    .type   main, @function
```
<br/><br/>
Which looks very complicated, albeit performing a simple addition. I'm going to focus on the important bits of this piece of assembler. 
<br/><br/>

`.cfi_startproc` and `.cfi_endproc` are directives used for debugging and exception handling. They mark the start and end of the stack frame.
The base pointer `rbp` is pushed onto the stack, and the stack pointer `rsp` is moved into `rbp`; this makes it easier to access function parameters and local variables.
<br/>
The local variables are stored at `-4(%rbp)` and `-8(%rbp)`. Notice the difference `4`, which is the size of the `int` on my machine.
This nomenclature roughly translates to `4` bytes before the address in `rbp`. Here, `a` and `b` will be moved into `edx` and `eax` respectively, and their addition is computed using 
the `addl` opcode. 
<br/>
Subsequently, the stack frame is cleaned up by popping `rbp` from the stack, and the value is returned via `ret`
<br/><br/>

# C/C++ specifics
From the GNU GCC manual:
<br/>
You can actually grab the return or frame address of the function. The prototype of this utility looks like this:
```c
void* __builtin_return_address(unsigned int level)
```
<br/>
This function returns the return address of the function, or one of its callers. `level` is the number of frames to scan up the call stack.

<br/>
`level = 0` yields the return address of the current function.
`level = 1` yields the address of the caller.

and so on.
<br/><br/>

On some platforms, additional post-processing is needed to get the *actual* address from `__builtin-return_adresss`. This is done as follows:
```c
void* addr = __builtin_extract_return_addr( __builtin_return_address (...) );
```

<br/><br/>

To give an example, x86/x86-64 systems, stack protection is used, with something called a Stack Canary.
It is some random value that is inserted typically between local variables and the return address. Before the function returns, this Canary value is checked.
If that value is altered, the program is usually terminated to prevent malicious code from running. The GCC compiler is awesome, so it provides some stack smashing protections<sup>[2]</sup> as well.

<br/><br/>

<hr>

# Footnotes
<sup>[1]</sup> The [Address Bus](https://en.wikipedia.org/wiki/Bus_(computing)#Address_bus) is used to carry the instruction from the PC to memory, and the fetched instruction 
is carried from memory to the *Instruction Register* by the *Data Bus*. The instruction is only temporarily placed in the IR.
Modern [memory buses](https://en.wikipedia.org/wiki/Bus_(computing)) connect directly to the DRAM chips, and have very low latency.

<br/><br/>


<sup>[2a]</sup> You can pass the `-fstack-protector` flag to detect buffer overflows on the stack. This is done by placing the Canary next to critical stack data, as explained earlier. However, this works only for functions that the compiler *thinks* is vulnerable. You can override this behaviour with the flag `-fstack-protector-all` for all functions.
<br/><br/>

```c
tree TARGET_STACK_PROTECT_GUARD (void)
```
Via the `TARGET_STACK_PROTECT_GUARD` hook in GCC, the compiler can obtain the value of the Canary. I think this value is set during compilation. 
A "guard" variable is created to detect stack smashing attacks, and it exists in the form of a global variable `__stack_chk_guard`. 

This hook allows each architecture to provide its own implementation. I'm not familiar with GCC internals though, you can read more [here](https://gcc.gnu.org/onlinedocs/gccint/target-macros/stack-layout-and-calling-conventions/stack-smashing-protection.html).

<br/><br/>

<sup>[2b]</sup>  Position Independent Executables (PIEs) allow the program code to be loaded at different addresses, making it more difficult for an attacker to predict the locations of specific code segments. You can enable PIE during compilation with the `fPIE` flag. 


PIE exists to support Address Space Layout Randomization (ASLR) in executables. You can explore this in action by compiling a simple C program with te `-g` flag, and set br/eakpoints at different positions in the source file, and you'll observe the randomization of addresses at the br/eakpoints. 

I think you'll need to call `set disable-randomization off` in GDB to turn on ASLR for the process. Otherwise, GDB will default to give fixed addresses.

<br/>
On Linux, you can check if ASLR is on by running:
```bash
$ sudo cat /proc/sys/kernel/randomize_via_space
```

<br/>

Usually with modern Linux kernels, it is enabled by default and set to a specific value of `2`. You can read in-depth about PIE by taking a look at [OpenBSD's PIE implementation](http://www.openbsd.org/papers/nycbsdcon08-pie/).
