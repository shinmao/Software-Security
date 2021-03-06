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
### Address and Storage Allocation
### Symbol Resolution
It can also be called as Symbol Binding, Name Binding, Name Resolution, etc. In details, resolution works for static-linking and binding works for dynamic-linking.
### Relocations
Object files would include many reference to others' code or data. In the linking process, linker would use relocations to fill in virtual address.

### Symbol table
In symbol table, there a list of names and their corresponding offset in txt or data segments.

### Static link v.s. Dynamic link
<img src="/img/static-link.png" width="50%">    
<img src="/img/dynamic-link.png" width="50%">  
For static link, libraries are attached by linker at linking process.  
For dynamic link (e.g. win `.dll`, linux `.so`), libraries are attached by linker at running time.  

For static link, executables would exist as single file instead of multiple files and only contain the necessary part of library. For static libraries, it would be format of `.a` in linux or `.lib` in windows.  
For dynamic link, linker made the reference to the shared obj and put into executables. Dynamic linked executables must load entire library because they are not able to know about the invoked library in advance. For dynamic libraries, it would be format of `.so` in linux or `.dll` in windows.

### ELF (Executable and Linkable Format)
<img src="/img/linking_execution.png" width="50%">  
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

Reading:  
* [The 101 of ELF files on Linux: Understanding and Analysis](https://linux-audit.com/elf-binaries-on-linux-understanding-and-analysis/)

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
<img src="/img/lazy-binding.png" width="50%">  
When we see `call <puts@plt>` in the program, it means that program will call `.got.plt` for help. `.got.plt` would locate to `puts@plt+6`. In order to run `puts@plt+6`, program would search `puts@plt`'s location first, then we can find that the first instruction in `puts@plt` is `jmp puts@got`, which means to run next line directly. Come to `plt0`, program would push `link_map` to the stack, with `index` previously pushed, program has all the parameters needed for `dl_runtime_resolve()` now! The resolve functon will call `call_fix_up()` to replace `puts@plt+6` with the real address.

## Software analysis

### CFG
With CFG, we can understand the program structure, and use it to achieve vulnerability mining and bug analysis.

* Static CFG analysis
* Dynamic CFG analysis

### Program Slicing
A technique to decompose programs by analyzing their data and control flow.  
slicing criterion: slice consists of program statements related to the values computed at point or variable.  
e.g. Given program p, `<s, v>` which specifies a statement s and a set of variables v we are interested in p. We always assign `s` with line number.  
A slice itself is an program susbset.  

To extract a slice, the dependences between statements must be computed first.  
Control Flow Graph (CFG): control dependencies for each operation.  
Program Dependence Graph (PDG): help to build slices in linear time.  
: nodes represent statements in source code, edges represent control and data flow dependences.

* Static slicing
Not assume any input for program.  
The slice is called static because it doesn't not consider any particular execution i.e., it works for any possible input data.  
* Dynamic slicing  
Useful for debugging.  
In general, dynamic slice is smaller than static ones.  
During program execution, the same statement can be executed several times with different values of variables (e.g. `for-loop`); therefore, slicing criterion needs to specify which particular execution of interest, `<si, v, {a1, a2, ...an}>`. `si` represents statement s executed for ith time, and `{a1, a2, ...an}` represents initial values of the program inputs.  
* Backward Slicing  
Traverse backward from slicing criterion.  
We are interested in all statements that would affect slicing criterion.  
Main application for debugging, program differencing and testing.  
* Forward Slicing  
How modification in a part of program can affect other parts of the program?  
Main application for software maintenance.

* [A vocabulary of program slicing-based techniques](https://dl.acm.org/doi/10.1145/2187671.2187674)

## Fuzzing
* [Recent Papers Related To Fuzzing](https://github.com/wcventure/FuzzingPaper)
* [Fuzzing技术总结（Brief Surveys on Fuzz Testing）](https://zhuanlan.zhihu.com/p/43432370)

## Symbolic Execution
* [My learning notes for Symbolic Execution](https://github.com/shinmao/Software-Security/blob/main/slides/symbolic%20execution.pdf)
* [Symbolic execution for software testing: three decades later](https://zhuanlan.zhihu.com/p/26927127?fbclid=IwAR2PQ-0wiOf9zZxMJSdCeuQ3NrdCVfxjRM4qSrjqyVuuIH0SLLCXVMrdpvg)
* [关于静态分析技术符号执行，从一个故事讲起······](https://bbs.huaweicloud.com/blogs/205975)

## Automatic Exploit Generation
* [软件漏洞自动利用研究进展](https://github.com/SCUBSRGroup/Automatic-Exploit-Generation)
* [Mr.Ma3k4H3d's blog](https://ma3k4h3d.top/)
