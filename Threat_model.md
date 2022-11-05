# Threat model

Our software and firmware threat model assumes that an attacker can exploit a vulnerability, either a stack or heap overflow, 
or use-after-free, present in the target software binary. This vulnerability can be used to overwrite key components of the 
running program like return addresses, function pointers, or VTable pointers. We also consider that the attacker has successfully 
bypassed ASLR, and has full knowledge of the memory layout, CFI is orthogonal to ASLR and does not interfere with it in any way. 
Nevertheless, the system enforces that (i) the .text segment is non-writable, preventing the application's code from being overwritten, 
and (ii) the data segments are non-executable blocking the attacker from executing injected data with proper CFI annotation. With 
regards to linking we assume that the binaries are configured with RELRO in order to protect GOT and PLT sections from being written. 
For privileged modes we exclude extending the kernel/firmware using unverified/untrusted kernel/firmware extensions such as drivers or
modules that may disable the protections. We also consider that the tool chain used for compiling the binary and libraries loaded by 
the binary are trusted and will not emit malware behavior intentionally. For each privilege level, we assume that the higher privilege 
levels are not exploited and that the code is trusted and does not include malware. 

We do not consider Data Oriented Programming (DOP) attacks in our threat model. In this attack scenario the attacker does not need 
to divert from the predefined control flow graph of the application. Rather, the attacker overwrites (non-control) data in order 
to change the CFI-compliant behavior of the application, e.g. change the arguments passed to a system call. This exploitation technique 
is possible even in the theoretically most accurate control flow integrity scheme. In some cases the attacker needs to also combine DOP 
and code reuse attacks in order to achieve exploitation. Thus, Control Flow Integrity can harden the application against exploitation, 
since an attacker has a limited set of valid target functions and cannot dynamically select to execute unintended code blocks of the 
application. We also exclude techniques based on debugging, emulation and code injection using hooking techniques. Finally, side-channel 
techniques allowing arbitrary data accesses are outside of the problem description CFI aims to solve. 

## Abbreviations
* CFI: Control Flow Integrity
* VTable: Virtual Method Table
* ASLR: Address Space Layout Randomization 
* RELRO: RELocation Read-Only
* GOT: Global Offset Table
* PLT: Procedure Linkage Table
* DOP: Data Oriented Programming
