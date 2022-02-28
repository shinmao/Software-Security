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
Symbolic execution(Symbex) emulates a program with symbolic values rather than concrete values. Symbex engines maintain the collection of symbolic values and formulas which is called symbolic state. What are the components of symbolic states?
```
symbolic state = symbolic expressions + path constraints
```
Path constraints(branch constraints) encode the limitation imposed on symbolic expressions.
```
--> code
if(x < 5)
  if(y >= 4)
--> sym expr
x -> a1
y -> a2
--> path constraint
(a1 < 5) ^ (a2 >= 4)
```
When we feed path constraint to constraint solver, it will solve the possible solutions of symbolic variables for us.  
### Symbex variants and limitations
#### Static symbolic execution
Explore multiple program paths in parallel.  
Advantage:
* Don't need to run program
* Do need to analyze whole program  

Disadvantage:
* scalability
Even with heuristics to limit the number of explored paths, cannot sure capture all interesting paths.
* Needs some efforts to handle the interaction with the environment out of program itself (libraries, system call).  
  * Sol1 - effect modeling, make a summary of the effects that system call or libraries can have on symbolic state. However, the model of summary is hard to be accurate. Also, we need to rewrite for different environments.  
  * Sol2 - Direct external interaction, just leave the engine make the call directly. However, if explored multiple paths operate on the same file in parallel, it would cause to conflict issue. Also, it is hard to decide conrete value for library.

#### Dynamic symbolic execution (Concolic execution)
Use concrete input to drive the execution while maintaining symbolic state as metadata. Rather than exploring paths in parallel, concolic execution only explore one path at once decided by the input. To explore different paths, concolic execution would flip the path constraints, and constraint solver would compute the concrete input that can lead to new branch, then we can start a new state.  
Advantage:  
* more scalable  
We don't need to mantain multiple execution states in parallel.  
* simply running external interaction with conrete input
* maintain fewer and more interesting symbolic states such as variables, also make solver easier to compute  

Disadvantage:  
* Coverage depends on initial concrete input

#### Online vs. Offline
Online means working in parallel while offline means working one path at once. There are also some implementations of offline static symbex and online dynamic symbex. Offline would execute same chunk of code for multiple times compared to online. However, online would need to track all states in parallel and cause to memory issue.

#### Symbolic state
Which part of program should be represented symbolically and which to be conrete? How should the symbolic state updated with some memory operation(e.g., value is written to array with symbolic index)?  
* Sol1 - Fully symbolic memory, one is copy several states for all possible values. If there is a constraint `a1 < 5` on index, then we can fork states for `a1 = 0, 1, ...4`. However, if the symbolic part is unbounded address, it would be impossible to list all values.  
* Sol2 - Address concretization, replace unbounded address with concrete values. However, it depends on value so that we might loss some interesting possible states.  

Most of the cases, we combine two solutions.

#### Path coverage
To solve path explosion problem, we need to decide which paths to explore.  
Strategies include DFS(explore nested code first), BFS(explore all paths in parallel), and so on.

### How to make scalability better?
#### Simplfying constraints
* limit the number of symbolic variables, but how to promise you symbolize the "interesting" program paths?  
employ taint analysis and fuzzing to find vulnerability. Symbolize only those inputs that taint analysis show to be relevant to vulnerability (e.g., overwritten return address) 
* Only symbolically execute the related instructions, e.g., what we are concerned about is only rax register, we can make a backward slice to find out all related instructions which would contribute to rax's value. The rest of the path should be conrete.  
* Don't make fully symbolic memory, it is crazy to symbolize unbounded memory address. e.g., Triton only accesses word-aligned addresses.

#### Avoiding constraint solver
Avoid using constraint solver on the path you are not interested in, and avoid using constraint solver on same problem for twice (we can cache prev results).

### Reading list
* [My learning notes for Symbolic Execution](https://github.com/shinmao/Software-Security/blob/main/slides/symbolic%20execution.pdf)
* [Symbolic execution for software testing: three decades later](https://zhuanlan.zhihu.com/p/26927127?fbclid=IwAR2PQ-0wiOf9zZxMJSdCeuQ3NrdCVfxjRM4qSrjqyVuuIH0SLLCXVMrdpvg)
* [关于静态分析技术符号执行，从一个故事讲起······](https://bbs.huaweicloud.com/blogs/205975)

## Automatic Exploit Generation
The first paper about Automatic Exploit Generation was released in 2008 on APEG, which was patch-based. Year 2016 is so special that DARPRA gave an CGC challenge, and most of the research work before 2016 are based on stack, format string while most of the work are based on heap (heap overflow, UAF) after 2016. Therefore, in following section about research paper, I would also separated them to stack-based and heap-based.

### Research paper
#### Revery
* [CCS-18 Revery: From Proof-of-Concept to Exploitable](https://dl.acm.org/doi/10.1145/3243734.3243847)  
* [slide notes](https://github.com/shinmao/Software-Security/blob/main/slides/Revery-From-PoC-to-Exploitable.pdf)

#### Reference
* [软件漏洞自动利用研究进展](https://github.com/SCUBSRGroup/Automatic-Exploit-Generation)
* [SCUBSRGroup /
Automatic-Exploit-Generation](https://github.com/SCUBSRGroup/Automatic-Exploit-Generation)
* [Mr.Ma3k4H3d's blog](https://ma3k4h3d.top/)
