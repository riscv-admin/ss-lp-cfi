# Hardware CFI techniques.


## Pointer Integrity (Cryptographically enforced CFI)


This mechanism utilizes cryptographic primitives:  a keyed message authentication code (mac) with a modifier. 
The modifier includes process context, and in some instructions the stack pointer, in order to verify that the 
control flow pointers are not corrupted before using them. The pointer authentication code (PAC) of each control 
flow pointer is stored in the unused bits of the pointer. The number of otherwise unused bits varies based on the 
size of the virtual address space. There are no free bits when a 32-bit VA is used, 24 free bits when 40-bit VA 
is used, 16 when 24-bit VA is used, and 9 when 59 bit VA is used. Since there are fewer free bits with the larger 
VAs (beyond 32-bit), the strength of the MAC weakens as the VA size increases. PACs are calculated and authenticated 
using custom instructions. These instructions have opcodes that are NOP on processors that do not support PAC.


- PAC <pointer>, &<pointer> (usually in stack) : calculates a MAC using a process key

- AUTH <pointer>, &<pointer> : authenticates the MAC using the process key


Each process has a unique modifier. Typically there is a set of static keys established at boot time for code and data. 
Then each context has a context modifier which is mixed into the hash. The modifier is used in order to calculate and 
  authenticate the control flow pointers. This technique can protect both data and control pointers, however it requires 
  significant chip area and has non negligible overhead. Also it is not suitable for 32-bit processors. Creates conflict 
  with use of unused address bits required for other usages - e.g. J-extension (pointer masking), future memory tagging 
  extensions, etc. Finally, there is an increase in power consumption due to the additional crypto operations.


PAC is also complemented with Branch Target Identification instructions in order to guard against execution of 
instructions that are not intended as branch targets. 


**Relevant material**

https://developer.arm.com/documentation/ddi0596/2021-09/Base-Instructions/BTI--Branch-Target-Identification-#

https://firmwaresecurity.com/2018/09/21/arm-v8-5-a-adds-branch-target-indicators-for-new-security/

https://www.usenix.org/system/files/sec19fall_liljestrand_prepub.pdf


## Dynamic Information Flow Tracking


A notable technique proposed in the literature, in order to counter buffer-overflow related exploits, is Dynamic 
  Information Flow Tracking (DIFT). The key concept of this mechanism is to taint memory regions where untrusted 
  data are residing, and track their propagation in the address space. Also, any new data resulting from computation 
  with tainted data, also become tainted. Exploits are detected with a predefined policy, depending on the implementation 
  of DIFT. In general, when tainted data are used in a suspicious manner a security exception is raised. A common 
  security exception trigger is when tainted data are used as an indirect jump operand. For example, in the case 
  where an attacker overwrites a return address, by exploiting a buffer overflow, input data will be tainted and 
  the DIFT policy will detect the violation since the return address will also be tainted. In the majority of DIFT 
  implementations, the protected application is oblivious of the mechanism, thus there is no need for source code 
  modifications. A detailed overview was sent by Greg Sulivan in the tech-tee mailing list. This mechanism requires 
  each word in memory to have a metadata word associated. For more complex processors with multiple levels of caches, 
  OOO execution multiple harts, and multiple instructions retirement per cycle this mechanism could lead to a non-trivial 
  overhead.


**Relevant Material**

https://lists.riscv.org/g/tech-tee/message/1113?p=%2C%2C%2C20%2C0%2C0%2C0%3A%3Arecentpostdate%2Fsticky%2C%2CIFC%252C+DIFT%2C20%2C2%2C0%2C87272764


## Memory Tagging


Recently ARM introduced memory tagging in processors The key idea of this mechanism is that each 16 byte block 
  of memory will be tagged using a 4-bit value. Each pointer will hold the tag of the valid memory blocks which 
  can be accessed on its 4 most-significant bits. Thus, loads and stores can only work if the address and the target 
  have the same tag. Memory blocks are retagged when freed. This mechanism can prevent a lot of exploitation techniques 
  which rely on arbitrary memory access. For example, a pointer pointing to a buffer will not be able to access contiguous 
  memory blocks beyond the buffer bounds, effectively protecting adjacent memory from being overwritten if the buffer 
  overflows. A potential problem with this mechanism is that 4 bit tags will offer reduced entropy and thus many memory 
  blocks will have the same tag. The performance overhead of this mechanism has not been measured. If the performance 
  overhead is substantial, memory tagging will not be a practical solution. In a similar manner with PAC this technique 
  requires that the MS bits of the VA are used for storing the tag. Thus, this technique cannot be deployed in 32-bit 
  architectures. In terms of memory overhead this technique imposes 3.25% memory for storing the tags. Moreover there 
  is noticeable memory fragmentation due to the need of 16 byte alignment. Memory tagging for stack locations - to 
  address pointers passed through the stack - could have additional overheads as the stack pointer cannot have a static 
  tag and not checking could lead to imprecisions.


Relevant material

https://llvm.org/devmtg/2018-10/slides/Serebryany-Stepanov-Tsyrklevich-Memory-Tagging-Slides-LLVM-2018.pdf

## Active Set Control Flow Integrity


Davi et al. proposed HAFIX [1], a system for backward edge CFI.  HAFIX does use dedicated, hidden memory elements for 
  storing critical information.Their implementation utilizes labels to mark functions as active call sites. Labels are 
  used as index in a bitmap, which dictates if a function is active or inactive. When a call instruction is executed 
  the next instruction must be a CFIBR in order to activate the function. Only active functions are valid as return 
  instruction targets. CFIDEL instructions are appended in the epilog of each function in order to deactivate them 
  during return. However, the aforementioned design has the disadvantage of allowing the attacker to jump to any active 
  function. This is important, since their method also allows for an attacker, using stack unwinding, to avoid the execution 
  of CFIDEL instructions in order to deactivate functions and eventually mark every function as an active one,  
  thus effectively permitting jumps anywhere in the program, and therefore being possibly vulnerable. HAFIX is based on 
  Active Set Control Flow Integrity, i.e. only functions that have been called are available for return targets. However, 
  it has been proven that this notion is very relaxed and can be circumvented [2]. 

[1]https://ieeexplore.ieee.org/abstract/document/7167258/

[2]https://www2.eecs.berkeley.edu/Pubs/TechRpts/2017/EECS-2017-78.pdf



## Shadow stack based + landing points 


In June 2016, Intel announced Control-flow Enforcement Technology (CET). In CET a shadow stack is defined in order 
  to protect backward-edge control flow transfers. When CET is enabled, call instructions are responsible for pushing 
  the return address in the shadow stack as well as in the original stack. Ret instructions pop the shadow stack and 
  ensure that it matches the return address acquired from the application’s stack. In case of mismatch, an exception
  is raised and the execution of the application stops. The shadow stack’s integrity is protected by the MMU in order
  to prevent an adversary from overwriting the return addresses residing in it. Any memory instruction trying to access 
  the contents of the shadow stack is blocked by the MMU and a page fault is raised. In order to protect forward-edge 
  control flow transfers ENDBRANCH instruction is used to mark the legitimate landing points for call and indirect 
  jump instructions within the applications code. When a jump is issued CET enters WAIT_FOR_ENDBRANCH state. If an 
  ENDBRANCH instruction is not the next instruction in the program stream, the processor raises a control protection
  fault. ARM also has a landing point based technique implemented on recent processors, namely Branch Target Identification 
  (BTI). If the binary is configured with BTI, branch instructions can only point to Branch Target instructions.

https://ieeexplore.ieee.org/document/6237009

https://software.intel.com/sites/default/files/managed/4d/2a/control-flow-enforcement-technology-preview.pdf

https://cseweb.ucsd.edu/~dstefan/cse227-spring20/papers/shanbhogue:cet.pdf

https://developer.arm.com/documentation/102433/0100/Jump-oriented-programming

## Labels + shadow stack

Other approaches use labels in order to protect forward-edges. An indirect call site is paired with valid target addresses, 
  using labels. The label must be set just before the indirect jump and will be checked at the target address. The target 
  address must always be a label, either by new instructions or by checking before the jump. These techniques require that
  the CFG of the application is extracted in order to pair indirect call sites with the appropriate functions. A more 
  accurate technique is to use per-call-site labels. In this variation, each call-site has a unique label which is prepended 
  in every valid target function, thus minimizing the use of same labels in functions and call-sites. 

https://www.usenix.org/conference/usenixsecurity14/technical-sessions/presentation/tice

https://pax.grsecurity.net/docs/PaXTeam-H2HC15-RAP-RIPROP.pdf

https://www.vvdveen.com/publications/TypeArmor.pdf

https://cse.usf.edu/~ligatti/papers/cfi-tissec.pdf

https://samos-conference.com/wp/wp-content/uploads/2021/07/SEC3_Hard-edges-Hardware-based-Control-Flow-Integrity-for-Embedded-Devices.pdf

https://www.phoronix.com/scan.php?page=news_item&amp;px=Intel-FineIBT-Security




| Technique |	Forward Edge |	Backward Edge	| Memory Overhead |	Runtime Overhead |	Architectural Modifications |
|-----------| -------------|---------------|------------------|------------------| -----------------------------|
|Code Pointer Integrity |	Probabilistic |(Entropy limited by available MS bits)	|Probabilistic|	Minor	Acceptable when protecting only code pointers |	Major (addition of cryptographic accelerator)|
DIFT |	Fine grained |	Fine grained |	Significant|	Significant (due to cache stressing)|	Major (Each memory word is extended)|
Memory Tagging|	Probabilistic (Restricted by tag width)|	Probabilistic|	Noticeable|	Minor|	Major (requires tag cache)|
Active Set CFI |	None	| Coarse (proven vulnerable) |	Minor	|Minor	|Minor|
Shadow stack + Labels|	Fine (requires accurate CFG)|	Total (return address copies are architecturally protected)|	Minor	|Minor	|Modest (requires new instructions and minor MMU modifications)|


More resources:  
https://arxiv.org/ftp/arxiv/papers/1706/1706.07257.pdf
