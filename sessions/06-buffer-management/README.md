# Overview

## Buffers

A buffer is an area of contiguous data in memory, determined by a starting address, contents and length. Understanding how buffers are used (or misused) is vital for both offensive and defensive purposes.
In C, we can declare a buffer of bytes as a char array, as follows:

```c
char local_buffer[32];
```

Which results in the following assembly code:

```nasm
push   rbp
mov    rbp,rsp
sub    rsp,0x20
...
ret
```
Notice that buffer allocation is done by simply subtracting its intended size from the current stack pointer (`sub rsp, 0x20`). This simply reserves space on the stack (remember that on x86 the stack grows “upwards”, from higher addresses to lower ones).

> A compiler may allocate more space on the stack than explicitly required due to alignment constraints or other hidden elements (like canary values). To exploit a program, the C source code may not be a good enough reference point for stack offsets. Only disassembling the executable will provide relevant information.

Buffers can be also be stored in other places in memory, such as the heap, `.bss`, `.data` or `.rodata`.

Analyze and compile the following snippet (also present in the lab files, go to `00-tutorial` and run `make buffers`):
```c
#include <stdio.h>
#include <stdlib.h>

char g_buf_init_zero[32] = {0};
/* g_buf_init_vals[5..31] will be 0 */
char g_buf_init_vals[32] = {1, 2, 3, 4, 5};
const char g_buf_const[32] = "Hello, world\n";

int main(void)
{
    char l_buf[32];
    static char s_l_buf[32];
    char *heap_buf = malloc(32);

    free(heap_buf);

    return 0;
}
```

Check the common binary sections and symbols. Use the usual coomands (`readelf -S`, `nm`).
Observe in which section each variable is located and the section flags.
<pre>
$ readelf -S buffers
...
  [16] .rodata           PROGBITS         <b>0000000000402000</b>  00002000
       0000000000000040  0000000000000000   <b>A</b>       0     0     32
...
  [24] .data             PROGBITS         <b>0000000000404040</b>  00003040
       0000000000000040  0000000000000000  <b>WA</b>       0     0     32
  [25] .bss              NOBITS           <b>0000000000404080</b>  00003080
       0000000000000060  0000000000000000  <b>WA</b>       0     0     32
...
Key to Flags:
  W (write), A (alloc), X (execute)

$ nm buffers
...
<b>0000000000402020 R</b> g_buf_const
<b>0000000000404060 D</b> g_buf_init_vals
<b>00000000004040a0 B</b> g_buf_init_zero

Key to Flags:
  R (symbol is read-only)
  D (symbol in initialized data section)
  B (symbol in BSS data section)

  A lowercase flag means variable is not visible local (not visible outside the object)
</pre>

You can also inspect these programmatically using pwntools and the ELF class:
```python
from pwn import *

elf = ELF('buffers')

bss    = elf.get_section_by_name('.bss')
data   = elf.get_section_by_name('.data')
rodata = elf.get_section_by_name('.rodata')

bss_addr    = bss['sh_addr']
data_addr   = data['sh_addr']
rodata_addr = rodata['sh_addr']

bss_size = bss['sh_size']
data_size = data['sh_size']
rodata_size = rodata['sh_size']

# A (Alloc) = 1 << 1 = 2
# W (Write) = 1 << 0 = 1
bss_flags    = bss['sh_flags']
data_flags   = data['sh_flags']
rodata_flags = rodata['sh_flags']

print("Section info:")
print(".bss:    0x{:08x}-0x{:08x}, {}".format(bss_addr, bss_addr+bss_size, bss_flags))
print(".data:   0x{:08x}-0x{:08x}, {}".format(data_addr, data_addr+data_size, data_flags))
print(".rodata: 0x{:08x}-0x{:08x}, {}".format(rodata_addr, rodata_addr+rodata_size, rodata_flags))

print()

print("Variable info:")
print("g_buf_init_zero: 0x{:08x}".format(elf.symbols.g_buf_init_zero))
print("g_buf_init_vals: 0x{:08x}".format(elf.symbols.g_buf_init_vals))
print("g_buf_const:     0x{:08x}".format(elf.symbols.g_buf_const))
```

Another handy utility is the `vmmap` command in `pwndbg` which shows all memory maps of the process at runtime:
```gdb
pwndbg> b main
pwngdb> run
pwndbg> vmmap
LEGEND: STACK | HEAP | CODE | DATA | RWX | RODATA
          0x400000           0x401000 r--p     1000 0      /home/user/buffers
          0x401000           0x402000 r-xp     1000 1000   /home/user/buffers
          0x402000           0x403000 r--p     1000 2000   /home/user/buffers
          0x403000           0x404000 r--p     1000 2000   /home/user/buffers
          0x404000           0x405000 rw-p     1000 3000   /home/user/buffers
    0x7ffff7dc9000     0x7ffff7dcb000 rw-p     2000 0
...
    0x7ffffffdd000     0x7ffffffff000 rw-p    22000 0      [stack]
0xffffffffff600000 0xffffffffff601000 --xp     1000 0      [vsyscall]
```

Non-static local variables and dynamically allocated buffers cannot be seen in the executable (they have meaning only at runtime, because they are allocated on the stack or heap in a function scope). The symbol names aren't found anywhere in the binary, except if debug symbols are enabled (`-g` flag).

## Stack buffer overflow

TODO: Graphical Stack View

We should know by now that the stack serves multiple purposes:
* Passing function arguments from the caller to the callee
* Storing local variables for functions
* Temporarily saving register values before a call
* Saving the return address and old frame pointer

Even though, in an abstract sense, different buffers are separate from one another, ultimately they are just some regions of memory which do not have any intrinsic identification or associated size. This is the reason why in some higher level languages it is not possible to write beyond the bounds of containers - the size is integrated into the object itself.

But in our case, bounds are unchecked, therefore it is up to the programmer to code carefully. This includes checking for any overflows and using **safe functions**. Unfortunately, many functions in the standard C library, particularly those which work with strings and read user input, are unsafe - nowadays, the compiler will issue warnings when encountering them.

### Buffer size and offset identification

When trying to overflow a buffer on the stack we need to know the size and where the buffer is in memory relative to the saved return address (or some other control flow altering value/pointer).

#### Static Analysis

One way, for simple programs, you can do **static analysis** and check some key points in the diassembled code.

For example, this simple program (`00-tutorial/simple_read`, run `make simple_read` to compile):
```c
#include <stdio.h>

int main(void) {
    char buf[128];
    fread(buf, 1, 256, stdin);
    return 0;
}
```

generates the following assembly:
```nasm
push   rbp
mov    rbp,rsp
sub    rsp,0x90
mov    rax,QWORD PTR fs:0x28

mov    QWORD PTR [rbp-0x8],rax
xor    eax,eax
# important bit
mov    rdx,QWORD PTR [rip+0x2ed6]        # 4040 <stdin@@GLIBC_2.2.5>
lea    rax,[rbp-0x90]                    # <- stack buffer starts at rbp-0x90
mov    rcx,rdx                           # <- 4th argument fo fread, stdin
mov    edx,0x100                         # <- 3rd argument of fread, number of elements read
mov    esi,0x1                           # <- 2nd argument of fread, size of element
mov    rdi,rax                           # <- 1st argument of fread (buffer address saved in RAX)
call   1030 <fread@plt>

push   rbp
mov    rbp,rsp
add    rsp,0xffffffffffffff80
# --- important bit ---
mov    rdx,QWORD PTR [rip+0x2efb]        # 404030 <stdin@@GLIBC_2.2.5>
lea    rax,[rbp-0x80]  # <- stack buffer starts at rbp-0x80
mov    rcx,rdx         # <- 4th argument fo fread, stdin
mov    edx,0x100       # <- 3rd argument of fread, number of elements read
mov    esi,0x1         # <- 2nd argument of fread, size of element
mov    rdi,rax         # <- 1st argument of fread (buffer address saved in RAX)
call   401030 <fread@plt>
# ---------------------
mov    eax,0x0
leave
ret
```

Looking at the `fread` arguments we can see the buffer start relative to `RBP` and the number of bytes read. `RBP-0x80+0x100*0x1 = RBP+0x80`, so the fread function can read 128 bytes after `RBP` -> return address stored at 136 bytes after `RBP`.


#### Dynamic analysis

You can determine offsets at runtime in a more automated way with pwndbg using an [De Bruijin sequences](https://en.wikipedia.org/wiki/De_Bruijn_sequence) which produces strings where every substring of length N appears only once in the sequence; in our case it helps us identify the offset of an exploitable memory value relative to the buffer.

For a simple buffer overflow the worflow is:
1. generate an long enough sequence to guarantee a buffer overflow
2. feed the generated sequence to the input function in the program
3. the program will produce a segmentation fault when reaching the invalid return address on the stack
4. search the offset of the faulty address in the generated pattern to get an offset

In pwndbg this works as such:
```
pwndbg> cyclic -n 8 256
aaaaaaaabaaaaaaacaaaaaaadaaaaaaaeaaaaaaafaaaaaaagaaaaaaahaaaaaaaiaaaaaaajaaaaaaakaaaaaaalaaaaaaamaaaaaaanaaaaaaaoaaaaaaapaaaaaaaqaaaaaaaraaaaaaasaaaaaaataaaaaaauaaaaaaavaaaaaaawaaaaaaaxaaaaaaayaaaaaaazaaaaaabbaaaaaabcaaaaaabdaaaaaabeaaaaaabfaaaaaabgaaaaaab
pwndbg> run
...
pwndbg>
<reading input, paste the generated pattern>
...
pwndbg> continue
...
Program received signal SIGSEGV, Segmentation fault
...
   0x401141 <main+27>    mov    esi, 1
   0x401146 <main+32>    mov    rdi, rax
   0x401149 <main+35>    call   fread@plt <fread@plt

   0x40114e <main+40>    mov    eax, 0
   0x401153 <main+45>    leave
 ► 0x401154 <main+46>    ret    <0x6161616161616172>
...
pwndbg> cyclic -n 8 -c 64 -l 0x6161616161616172
136
```
_Note: we get the same 136 offset computed manually with the static analysis method._

### Input-Output with pwntools

TODO

# Challenges

## 01. Challenge: Parrot

Some programs feature a stack _smashing protection_ in the form of stack canaries, that is, values kept on the stack which are checked before returning from a function. If the value has changed, then the “canary” can conclude that stack data has been corrupted throughout the execution of the current function.

We have implemented our very own `parrot`. Can you avoid it somehow?

## 02. Challenge: Indexing

More complex programs require some form of protocol or user interaction. This is where _pwntools_ shines.
Here's an interactive script to get you started:

```python
    #!/usr/bin/env python
    from pwn import *

    p = process('./indexing')

    p.recvuntil('Index: ')
    p.sendline() # TODO (must be string)

    # Give value
    p.recvuntil('Value: ')
    p.sendline() # TODO (must be string)
    p.interactive()
```

> Go through GDB when aiming to solve this challenge. As all input values are strings, you can input them at the keyboard and follow their effect in GDB.

## 03. Challenge: Smashthestack Level7

Now you can tackle a real challenge. See if you can figure out how you can get a shell from this one.

> Hints:

> There's an integer overflow + buffer overflow in the program.

> How does integer multiplication work at a low level? Can you get get a positive number by multiplying a negative number by 4?

> To pass command line arguments in gdb use `run arg1 arg2 ...` or `set args arg1 arg2 ...` before a `run` command

> In _pwntools_ you can pass a list to `process` (`process(['./level07', arg1, arg2]`)

## 04. Challenge: Neighbourly

Let's overwrite a structure's function pointer using a buffer overflow in its vicinity. The principle is the same.

## 05. Challenge: Input Functions

On the same idea as the _Indexing_ challenge but much harder. Carefully check what input functions are used and parse the input accordingly.

## 06. Challenge: Bonus: Birds

Time for a more complex challenge. Be patient and don't speed through it.

# Further Reading

TODO