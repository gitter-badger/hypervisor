# Virtual Machine Monitor (VMM) {#bfvmm_readme}

## Description

The VMM is the part of the hypervisor that monitors each virtual machine. The term "hypervisor" can mean a lot of things. Generally speaking, it should refer to the piece of code that maintains control of the supervisor, but instead generally refers to everything within a hypervisor's project including the drivers and userspace code (e.g. the Xen hypervisor is made up of everything from the actual hypervisor itself, but also it's hypercall libraries, libXL, etc...). Thus, the Bareflank hypervisor describes the entire project, while the VMM is the piece of code that oversees the management of each virtual machines, and the "exit handler" is the actual piece code that maintains control over the supervisor. The exit handler is contained within the VMM, and the VMM is contained within the hypervisor.

## How It is Used

The VMM itself is made up of a set of shared libraries, is loaded into memory by the driver entry point, and is executed from different points of view. When starting / stopping, the VMM is executed in ring 0 along side the host OS kernel. When a VM exit occurs, the VMM is executed with so called "ring -1" privileges in the form of the exit handler, and is responsible for emulating the offending instruction.

Since the VMM is relatively isolated it must provide it's own execution environment. This includes support for memory management (i.e. malloc / free), debugging, and STL support via libc++.so.

The bootstrapping process starts with the driver entry loading all of the VMM modules into memory and relocating each symbol. From there, the driver entry calls local_init to execute all of the global constructors (e.g. std::cout is initialized here). Once all of the global constructors are executed, the driver entry calls into the VMM's memory manager to add memory descriptors to the memory manager. These descriptors tell the VMM what the virtual -> physical memory mappings are, that the VMM will later use to setup the VMM and it's associated VMs. Serial is initialized the first time it is executed as it is defined as a static "singleton-ish" class. Finally, the driver entry will execute init_vmm and start_vmm found in the "entry" module.

The entry module guards the host OS kernel from the VMM by doing two things: catching all exceptions that bubble up to the entry functions, and provide a custom stack. Most of the errors in the VMM are designed to be caught by the entry code. If an error should occur, commit_or_rollback classes are added to the code to automatically rollback changes to the VMM, and the exception will eventually be picked up by the top level catch alls provided by the entry module. This code will tell the developer what the error was, and then hand control back to the host OS indicating that an error occurred. In addition, a custom stack is provided as most host OSes provide very small stacks. The VMM leverages libc++ to speed up the process of creating new hypervisor technologies at the expense of running a library designed for userspace in the kernel. For prototyping this is great as it significantly lowers the barrier to entry for doing hypervisor research, but will likely need to be replaced with a kernel friendly STL implementation in the distant future should Bareflank ever be used in a production environment. One problem that was encountered was libc++ assumes access to a large stack (for example, std::setw is not kind to the stack), resulting in the VMM corrupting the host OSes stack. To address this issue, the entry module provides a custom stack prior to executing the VMM logic for all entry functions. This way, the VMM has a much larger stack to play with. Once the VMM is ready to hand control back to the host OS, the stack is put back, and execution continues as normal.

The entry code then calls into the vCPU manager which in turn sets up each vCPU. A lot of the VMM code is designed to work with both Intel and ARM. The vCPU however is designed specifically to be architecture specific, and this is where ARM support will be added later on in the future. Currently the only vCPU that is provided is the Intel x64 vCPU, which has it's intrinsics class (raw assembly functions), it's VMXON class, VMCS class and exit handler class. The rest of the process is right out of the Intel Software Developers Manual. VMXON is started and initialized, then the VMCS. Once the VMM is started, and a VM exit occurs, execution is handed to the exit handler to be emulated.

## Limitations

Currently the VMM is Intel specific. It's designed specifically to be able to be extended with ARM support in the future. Once this process begins, it's likely some things will have to change to support this added functionality, so keep an eye on the project as changes to the interfaces might occur if needed to support other architectures.

The VMM has it's own memory pool which is static. Future versions will likely work to make the management of this memory pool easier but it's likely to remain a statically compiled resource for the foreseeable future. If you end up with an out-of-memory error, you will need to modify the amount of memory that is provided to the VMM in the constants.h file.

Bareflank has no support for libc functions. Some are there are they are needed by libc++ but don't rely on them as they could be added / removed over time as they are only there to support libc++ and nothing else. We also do not support anything in libc++ that relies on floating point fucntions. For example, currently in libc++, std::unqiue_map uses ceilf, which we do not support. The same goes for some iostream functions. Furthermore, some STL functionality makes no sense in the kernel, like fstream, which we obviously do not support. Our goal is to use the STL to ease development of things that the kernel would normally use, not to provide generic support for the STL. The use of the STL in the kernel is already pretty controversial, lets not push it.

The VMM does not support multi-core and also does not have support for pthreads, mutexes, etc... Version 1.1 of Bareflank will provide multi-core support, which will add basic mutex support, but generic thread support in the VMM will not be added. Currently we have no plans to provide "worker thread" support in the VMM, but such functionality can certainly be added by a user of Bareflank.

## Notes

Since the VMM is compiled using System V compilers (i.e. GCC or Clang/LLVM), a System V specific issue must be addressed. The System V spec states that leaf functions do not need to move the stack pointer, providing an optimization (called the red zone) that actually has a pretty dramatic improvement on performance (since each leaf function can avoid at least 2 instructions per call). The problem is, in kernel code this optimization does not work as interrupts use the existing stack pointer as their execution stack, which results in a corrupt stack if an interrupt fires while a leaf function is executing (a bug that was not fun to track down). Great care must be taken to ensure that all code that will execute in the VMM is compiled using the -mno-red-zone. For more information, please see the following:

[red zone](http://eli.thegreenplace.net/2011/09/06/stack-frame-layout-on-x86-64/)

Currently the VMM does not link against libgcc. Most articles on osdev.org would tell you that not linking against libgcc is a big "no-no". In our case, it's fine. Libgcc mainly provides 64bit instruction support on 32bit systems, as well as the C++ unwinder. Bareflank does not support 32bit systems, and it provides it's own C++ unwinder, thus has no reason to add support for libgcc. If libgcc is needed in the future, a patch would have to be made to GCC to tell it to compile libgcc without red zone support (a patch this project does not want to maintain if it can be avoided). If support is needed in the future, it will manifest itself as a missing symbol while trying to load the VMM.
