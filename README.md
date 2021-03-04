# Software-Security
Start from understanding how program works.  
An executable consists of several object files.

## What does an object file contain?
* Symbol
* Compiled code
* Constant data
* Imports
* Exports

The linker would turn bunch of files into an executable.

## Linking process
In the linking process, there are several important informations.  
### Relocations
Object files would include many reference to others' code or data. In the linking process, linker would use relocations to fill in virtual address.

### Symbol table
In symbol table, there a list of names and their corresponding offset in txt or data segments.

### Static link v.s. Dynamic link
![]()  
![]()  
For static link, libraries are attached by linker at linking process.  
For dynamic link (e.g. win `.dll`, linux `.so`), libraries are attached by linker at running time.  

For static link, executables would exist as single file instead of multiple files and only contain the necessary part of library.  
For dynamic link, linker made the reference to the shared obj and put into executables. Dynamic linked executables must load entire library because they are not able to know about the invoked library in advance.

### ELF (Executable and Linkable Format)
![]()  
ELF is the way we specify the layout of obj file on Linux systems. There are two kinds of view ways for ELF at linking time (left side) and execution time (right side).  
Linking view deals with sections, which provide information necessary at linking time. Section has name and type, and can be located by section header table. Each section has one corresponding section header, and won't overlap with each others.  
Execution view deals with segments, which provide information necessary at running time. Segments consist of a group of sections, e.g. `.txt` contains code, `.data` contains data, `.dynamic` is related to dynamic loading.

### Focus on file using dynamic link
```
  link editor add PT_INTERP to header of ELF
                |
                |
                v
             PT_INTERP
                |
                |  invoke
                v
             dynamic linker   >  create process image
                |
                |   executable, read-only -> text
                |   .data, .bss -> data
                |
                v
             merge sections into segments
                |
                v
             add exe's file mem segment to process image
             add shared obj mem segment to process image
                |
                |
                v
            relocations: update absoloute address
                |
                |
                |
                v
            close file descriptor for reading ELF
                |
                |
                |
                v
           give control to program
```
sections:  
* `.text`: store executable code
* `.data`: store global variables with initialized values
* `.bss`: store global variables with no init values
* `.rodata`: store read-only data

### GOT (Global Offset Table)
First, what is **Lazy binding**?  
ELF loada the entire library with several unecessary functions. Therefore, lazy binding let ELF find real address only after call function for the first time (This is also why it is called lazy).  
How to check whether your program use Lazy binding or not?  
```
objdump -d elf
```
And you can find something like `call <puts@plt>`.  
Where is GOT table? It resides in data section.  
How do we use GOT to get the address of function? The most important part in GOT is `.got.plt`.  
![]()  
When we see `call <puts@plt>` in the program, it means that program will call `.got.plt` for help. `.got.plt` would locate to `puts@plt+6`. In order to run `puts@plt+6`, program would search `puts@plt`'s location first, then we can find that the first instruction in `puts@plt` is `jmp puts@got`, which means to run next line directly. Come to `plt0`, program would push `link_map` to the stack, with `index` previously pushed, program has all the parameters needed for `dl_runtime_resolve()` now! The resolve functon will call `call_fix_up()` to replace `puts@plt+6` with the real address.
