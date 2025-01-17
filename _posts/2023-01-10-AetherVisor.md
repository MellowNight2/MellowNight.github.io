---
layout: post
title: "How AetherVisor works under the hood"
date: 2023-01-19 01:01:01 -0000
author: MellowNight
---

<br>

- [Introduction](#introduction)
- [Virtual machine setup](#virtual-machine-setup)
  * [Checking for AMD-V support](#checking-for-amd-v-support)
  * [Setting up the VMCB](#setting-up-the-vmcb)
  * [MSR intercepts](#msr-intercepts)
  * [Setting up nested paging](#setting-up-nested-paging)
  * [vmmcall interface](#vmmcall-interface)
  * [VM launch and VM exit operation](#vm-launch-and-vm-exit-operation)
  * [Stopping the hypervisor](#stopping-the-hypervisor)
- [Loading the hypervisor](#loading-the-hypervisor)
- [Features](#features)
  * [Nested Page Table hooks](#nested-page-table-hooks)
    + [Really annoying problems](#really-annoying-problems)
  * [Sandboxing](#sandboxing)
    + [Intercepting out-of-sandbox code execution](#intercepting-out-of-sandbox-code-execution)
    + [Intercepting out-of-sandbox memory access](#intercepting-out-of-sandbox-memory-access)
    + [AetherVisor sandbox vs. other tools](#aethervisor-sandbox-vs-other-tools)
  * [Branch Tracing](#branch-tracing)
  * [Process-specific syscall hooks](#process-specific-syscall-hooks)
- [Future plans](#future-plans)

<small><i><a href='http://ecotrust-canada.github.io/markdown-toc/'>Table of contents generated with markdown-toc</a></i></small>

<br>
<br>

## Introduction

&emsp;&emsp;A while ago, I wrote AetherVisor: a stealthy dynamic analysis and memory hacking framework, based on a type-2 AMD hypervisor. I no longer want to treat protected software as a black box, so I paused this project to study other topics such as x86 deobfuscation. AetherVisor is a minimal hypervisor, so it may be unstable, and many special instruction intercepts aren't supported. For more robust and stable tool development, it's better to use more established options like KVM. Although KVM has its advantages, AetherVisor remains a valuable tool for building minimal, stealthy, debugger tools and writing hacks.
<br>
<br> 

This is a general overview of AetherVisor's implementation, with insight into some potential issues.

<br> 

## Virtual machine setup

This first section will go over the initialization and launch process for AetherVisor.

<br> 

### Checking for AMD-V support 

Before any VM initialization, three conditions must be met:
<br> 
1. AMD SVM must be supported.
2. Virtualization must be enabled in BIOS, so that VM_CR.SVMDIS can be set to 0 and VM_CR.LOCK can be locked.
3. The MSR_EFER.svme bit is set, after conditions #1 and #2 are met.

<br> 

*First, check if AMD SVM is supported:*

<br> 
```cpp
enum CPUID
{    
    vendor_and_max_standard_fn_number = 0x0,
    feature_identifier = 0x80000001,
};

bool IsSvmSupported()
{
	int32_t	cpu_info[4] = { 0 };

	__cpuid(cpu_info, CPUID::feature_identifier);

	// 1. check if SVM is supported with CPUID Fn8000_0001_ECX

	if ((cpu_info[2] & (1 << 2)) == 0)
	{
		return false;
	}

	int32_t vendor_name_result[4];

	char vendor_name[13];

	__cpuid(vendor_name_result, CPUID::vendor_and_max_standard_fn_number);
	memcpy(vendor_name, &vendor_name_result[1], sizeof(int));
	memcpy(vendor_name + 4, &vendor_name_result[3], sizeof(int));
	memcpy(vendor_name + 8, &vendor_name_result[2], sizeof(int));

	vendor_name[12] = '\0';

	DbgPrint("[SETUP] Vendor Name %s \n", vendor_name);

	// 2. check if we are running on an AMD processor or inside a VMWare guest by 
	// querying the  CPUID Fn0000_0000_E[D,C,B]X value

	if (strcmp(vendor_name, "AuthenticAMD") && strcmp(vendor_name, "VmwareVmware"))
	{
		return false;
	}

	return true;
}
```
<br> 
<br> 

*The VM_CR.LOCK bit will be locked to 1 if SVM is disabled in BIOS, preventing you from changing the value of VM_CR.SVMDIS. If VM_CR.LOCK is already locked and VM_CR.SVMDIS is 1, then abort initialization. Otherwise, clear VM_CR.SVMDIS and set VM_CR.LOCK.*

<br> 

```cpp
enum MSR : UINT64
{
    VM_CR = 0xC0010114,
};

bool IsSvmUnlocked()
{
	MsrVmcr	msr;

	msr.flags = __readmsr(MSR::VM_CR);

	/*  Check if SVM is locked	*/

	if (msr.svm_lock == 0)		// bit 3
	{
		msr.svme_disable = 0;   // bit 4
		msr.svm_lock = 1;       
		__writemsr(MSR::VM_CR, msr.flags);
	}
	else if (msr.svme_disable == 1)
	{
		return false;
	}

	return true;
}
```

<br>
<br>
*Finally, we can enable AMD SVM for this core:*

<br>

```
enum MSR : UINT64
{ 
	EFER = 0xC0000080,
};

void EnableSvme()
{
	MsrEfer	msr;
	msr.flags = __readmsr(MSR::EFER);
	msr.svme = 1;
	__writemsr(MSR::EFER, msr.flags);
}
```
<br>

### Setting up the VMCB


&emsp;&emsp;The Virtual Machine Control Block (VMCB) contains core-specific information about the AMD virtual machine's state. It is split into two parts: the save state area and the control area.

<br>

&emsp;&emsp;The save state area contains most of the guest state, including general purpose registers, control registers, and segment registers. The control area mostly consists of VM configuration options for the CPU core. Host register values are simply copied to the save state area in AetherVisor.

<br>

*The VMCB:*

<br>

![alt text](https://raw.githubusercontent.com/MellowNight/MellowNight.github.io/main/assets/img/VMCB.png "Logo Title Text 1")

<br> 

### MSR intercepts

&emsp;&emsp;AetherVisor only intercepts reads and writes to the EFER msr. The EFER.svme bit indicates that AMD SVM is enabled, so it's necessary to spoof it to zero to hide the hypervisor. 

<br>

*Look into the manual to see the MSR permission map format lol*

<br>

```cpp
size_t bits_per_msr = 16000 / 8000;
size_t bits_per_byte = sizeof(char) * 8;
size_t msrpm_size = PAGE_SIZE * 2;

// ...

auto section2_offset = ();

auto efer_offset = section2_offset + (bits_per_msr * (MSR::EFER - 0xC0000000));

/*	intercept EFER read and write	*/

RtlSetBits(&bitmap, efer_offset, 2);
```

<br>
<br>
*Spoofing EFER.SVME to 0*

<br>

```cpp
void HandleMsrExit(VcpuData* core_data, GuestRegisters* guest_regs)
{
    LARGE_INTEGER   msr_value{ msr_value.QuadPart = __readmsr(msr_id) };

    switch (msr_id)
    {
		case MSR::EFER:
		{
			auto efer = (MsrEfer*)&msr_value.QuadPart;
			efer->svme = 0;
			break;
		}
    }

    core_data->guest_vmcb.save_state_area.Rax = msr_value.LowPart;
    guest_regs->rdx = msr_value.HighPart;
}
```

<br> 

&emsp;&emsp;EasyAntiCheat and Battleye write to unimplemented MSRs to try and trigger undefined behavior while running under the hypervisor, so I inject #GP(0) when the guest writes to an MSR outside the manual's specified ranges.

<br>
 
*Preventing crashes from unimplemented MSR access*

<br>

```cpp
// ...

uint32_t msr_id = guest_regs->rcx & (uint32_t)0xFFFFFFFF;

if (!(
	((msr_id > 0) && (msr_id < 0x00001FFF)) || 
	((msr_id > 0xC0000000) && (msr_id < 0xC0001FFF)) || 
	(msr_id > 0xC0010000) && (msr_id < 0xC0011FFF)
	))
{
	/*  Battleye/EAC/PUBG unimplemented MSR checks    */

	InjectException(core_data, EXCEPTION_GP_FAULT, true, 0);
	return;
}

// ...
```

<br>

### Setting up nested paging

&emsp;&emsp;Nested paging/AMD RVI adds a second layer of page tables that translates gPA (guest physical address) to hPA (host physical address). gPA are identity mapped to hPA with AetherVisor's nested page table setup. A lot of magic can be done by manipulating NPT entries, such as hiding memory, hiding hooks, isolating memory spaces, etc. Think outside of the box :) 

<br>

#### Here's how to set up an nested page directory with identity mapping:

1. Obtain physical memory ranges using MmGetPhysicalMemoryRanges. 
2. Allocate a page for npml4/nCR3
3. Do a page walk into the nCR3 directory using each physical page address. For each nested page level, we check the indexed NPT entry's present bit. If present == 0, we use the existing table pointed to by NPT entry's PFN; otherwise, we allocate a new table for the PFN
4. At the last level, point nPTE->PFN to the physical page address itself.

<br>

Boom, we've created a 1:1 gPA->hPA mapping for a page.

<br>

*This is basically the same as normal virtual->physical paging lol*

<br>

![alt text](https://raw.githubusercontent.com/MellowNight/MellowNight.github.io/main/assets/img/NestedPagingSetup.png "Logo Title Text 1")

<br>

### vmmcall interface

The guest can interface with the hypervisor by executing the vmmcall instruction with specific parameters. Based on the code passed in RCX, one of the following operations are executed:

<br>

```cpp
enum VMMCALL_ID : uintptr_t
{
    disable_hv = 0x11111111,
    set_npt_hook = 0x11111112,
    remove_npt_hook = 0x11111113,
    is_hv_present = 0x11111114,
    sandbox_page = 0x11111116,
    register_instrumentation_hook = 0x11111117,
    deny_sandbox_reads = 0x11111118,
    start_branch_trace = 0x11111119,
};
```

<br>

Wrapper functions for the vmmcall interface are provided by aethervisor-api.lib. You can use it by including aether_api.h and the static library in your project.

<br>

### VM launch and VM exit operation

The final step of preparing for SVM operation is executing vmload to load hidden guest state information. The vmrun instruction launches the hypervisor, stops host state execution, and loads the guest context from VMCB.

<br>

```cpp
; omitted
EnterVm:
	mov rax, [rsp]	; put physical address of guest VMCB in rax

	vmload rax		; vmload hidden guest state

	; int 3

	vmrun rax		; virtualize this processor (execution will pause here)

	vmsave rax		; vmexit! save hidden state

	PUSHAQ			; save all guest general registers

	mov rcx, [rsp + 8 * 16 + 2 * 8]	; pass virtual processor data ptr in arg 1
	mov rdx, rsp					; pass guest registers in arg 2

	; omitted code...

	call HandleVmexit	; vmexit handler
```

<br>

Once a #VMEXIT occurs, execution resumes and line 11 is reached.

<br>

### Stopping the hypervisor

To completely stop the hypervisor, we vmexit out of guest state, disable SVM, load the guest state registers, and resume execution where the guest exited. 

There are multiple steps involved. In the C++ vmexit handler, we do the following:

<br>

In HandleVmexit():
<br>
```
if (end_hypervisor)
{	
	// 1. Load guest CR3 context
	__writecr3(vcpu_data->guest_vmcb.save_state_area.Cr3.Flags);

	// 2. Load guest hidden context
	__svm_vmload(vcpu_data->guest_vmcb_physicaladdr);
	
	// 3. Enable global interrupt flag
	__svm_stgi()
	
	// 4. Disable interrupt flag in EFLAGS (to safely disable SVM)
	_disable()

	MsrEfer msr;

	msr.flags = __readmsr(MSR::EFER);
	msr.svme = 0;

	// 5. disable SVM
	__writemsr(MSR::EFER, msr.flags);

	// 6. load the guest value of EFLAGS
	__writeeflags(vcpu_data->guest_vmcb.save_state_area.Rflags.Flags);	

	// 7. restore these values later
	guest_ctx->rcx = vcpu_data->guest_vmcb.save_state_area.Rsp;
	guest_ctx->rbx = vcpu_data->guest_vmcb.control_area.NRip;

	Logger::Get()->Log("ending hypervisor... \n");
}

return end_hypervisor;

// ...
```

<br>

After disabling virtualization, there is no more "guest" state; there is only the "host" processor state. We will resume execution from where the guest left off:

<br>

```
	call HandleVmexit	; the C++ vmexit handler

	;  omitted asm...

	test al, al	; if return 1, then end VM

	POPAQ	; 8. load the guest's general purpose register context

	; ...

EndVm:
	; in HandleVmexit, rcx is set to guest stack pointer, and rbx is set to guest RIP
	; but guest state is already ended so we continue execution as host

	mov rsp, rcx	; 9. load guest stack

	jmp rbx		; 10. resume execution from where the guest exited

LaunchVm endp
```

<br>

## Loading the hypervisor

&emsp;&emsp;Most of the time, I just used OSRLoader to test my hypervisor, which worked flawlessly. However, when I attempted to launch the hypervisor with KDMapper, I got the following VMWare error:

<br>

<p align="center">
  <img src="https://raw.githubusercontent.com/MellowNight/MellowNight.github.io/main/assets/img/kdmapperfault1.PNG">
</p>

<br>

&emsp;&emsp;Unfortunately, there was no crash dump, so I was unable to gather any useful information. I was confused as to why there were no similar issues with OSRLoader. There were two things I was certain of: First, the hypervisor launched successfully on all cores, and second, the crash occurred some time after I exited my driver. To learn more about this KDMapper issue, I wanted to see what happened when I triggered a vmexit before exiting DriverEntry, and what happened when vmexited outside of AetherVisor's entry point. I placed a breakpoint after vmrun, to catch vmexits:

<br>

*breakpoint after vmrun, to catch #VMEXIT:*

<br>

<p align="center">
  <img src="https://raw.githubusercontent.com/MellowNight/MellowNight.github.io/main/assets/img/kdmapperfault2.PNG">
</p>

<br>

*vmmcall #VMEXIT test in a seperate driver, ran after AetherVisor's entry point returns:*

<br>

<p align="center">
  <img src="https://raw.githubusercontent.com/MellowNight/MellowNight.github.io/main/assets/img/kdmapperfault3.PNG">
</p>

<br>

&emsp;&emsp;I vmmcall'ed the hypervisor before returning from DriverEntry, and then I executed vmmcall from a 2nd driver. The breakpoint I placed right after vmrun should've been hit twice, but only one breakpoint was hit before the crash.

<br>

<p align="center">
  <img src="https://raw.githubusercontent.com/MellowNight/MellowNight.github.io/main/assets/img/kdmapperfault4.jpg">
</p>

<br>

&emsp;&emsp;This must mean that the vmexit handler is somehow fked up after DriverEntry returns! If the breakpoint on vmexit is not being reached, and the exception handlers crash without double fault or bluescreen, I can assume that either the segments are messed up, or no code is mapped to the CR3 context. 

<br>

&emsp;&emsp;I came to the conclusion that I received the crash because AetherVisor was initialized from within kdmapper's process context, thus KDMapper's CR3 would have been saved in guest VMCB. After guest mode is launched, the KDmapper process exits inside guest mode, but the host page tables (used for vmexit handlers) are still using the KDMapper's CR3! I fixed this by launching my hypervisor from a system thread, in the context of system process, which never exits.  

<br>

## Features

In this last section, I will explain the implementation details of features provided by AetherVisor.

<br>

### Nested Page Table hooks


&emsp;&emsp;EPT/NPT hooking is a technique to hide inline hooks, by intercepting and redirecting memory reads to a different page. 

<br>

*What happened to the mov instruction at 0:29? 😳*

<br>

<details open="" class="details-reset border rounded-2">
  <summary class="px-3 py-2">
    <svg aria-hidden="true" height="16" viewBox="0 0 16 16" version="1.1" width="16" data-view-component="true" class="octicon octicon-device-camera-video">
    <path fill-rule="evenodd" d="M16 3.75a.75.75 0 00-1.136-.643L11 5.425V4.75A1.75 1.75 0 009.25 3h-7.5A1.75 1.75 0 000 4.75v6.5C0 12.216.784 13 1.75 13h7.5A1.75 1.75 0 0011 11.25v-.675l3.864 2.318A.75.75 0 0016 12.25v-8.5zm-5 5.075l3.5 2.1v-5.85l-3.5 2.1v1.65zM9.5 6.75v-2a.25.25 0 00-.25-.25h-7.5a.25.25 0 00-.25.25v6.5c0 .138.112.25.25.25h7.5a.25.25 0 00.25-.25v-4.5z"></path>
</svg>
    <span aria-label="Video description ShooterGameNptTest.mp4" class="m-1">ShooterGameNptTest.mp4</span>
    <span class="dropdown-caret"></span>
  </summary>

  <video src="https://user-images.githubusercontent.com/66788741/215348869-1f41482f-e0e6-4069-982c-92c8cf591960.mp4" data-canonical-src="https://user-images.githubusercontent.com/66788741/215348869-1f41482f-e0e6-4069-982c-92c8cf591960.mp4" controls="controls" muted="muted" class="d-block rounded-bottom-2 border-top width-fit" style="max-height:640px; min-height: 200px">

  </video>
</details>

<br>

&emsp;&emsp;Intel supports execute-only pages through extended page tables, so developers can simply create an execute-only page containing hooks, and a copy of the page, without the hooks. An Intel HV can handle EPT faults caused by attempted reads from the page, and redirect the read to the copy page. The hooked page is restored on EPT faults thrown by instruction fetches from the page.

<br>

<p align="center">
  <img src="https://raw.githubusercontent.com/MellowNight/MellowNight.github.io/main/assets/img/EPTHook.png">
</p>

<br>

&emsp;&emsp;AMD nested page tables do not support the execute-only permission, so AMD system programmers might need to trap every execute access to the hook page, which causes overhead. Two workarounds can be considered to achieve execute-only pages on AMD:

<br>

**SEV-SNP (secure nested paging):** pages in the guest can be restricted to execute-only with VMPL permission masks in the RMP (reverse map table). These RMP permission checks are only in effect when SEV-SNP is enabled. See AMD system programming manual sections 15.36.3 to 15.36.5.  

<br>

**Memory Protection Keys:** execute-only memory can be achieved with with MPK by disabling read access
through the PKRU register and allowing execution through the page table. Memory protection keys control read and write access to pages, but ignore instruction fetches. See AMD system programming manual section 5.6.7.
    
<br>

&emsp;&emsp;Unfortunately, none of these features were supported on my AMD ryzen 2400G CPU, so I needed to somehow hide hooks by trapping on execute.

<br>

&emsp;&emsp;To start off, I set up two ncr3 direcories: a **"shadow"** nCR3 with every page set to read/write only, and a **"primary"** ncr3 with every page allowing read/write/execute permissions. By default, the **primary** nCR3 is used. Upon executing the hooked page, #NPF is thrown and AetherVisor switches the guest to the **shadow** nCR3 context. AetherVisor switches back to **primary** nCR3 context whenever RIP goes outside of the hooked page.

<br>

The following steps describe how the NPT hook is set:

<br>

1. __writecr3() to attach to the process context saved in VMCB
2. Create a non-paged pool shadow copy of the target 4KB page
3. Copy the hook shellcode to **shadow** copy + hook page offset.    
4. Update the target page nPTE's PFN in the **shadow** nCR3 to the **shadow** copy 
5. Set the permissions of the **shadow** nCR3 nPTE to RWX
6. Set the permissions of the **primary** nCR3 nPTE (which points to the hookless copy of the target page) to rw-only
7. Create an MDL to lock the hooked page's virtual address to the guest and host physical addresses. *If our target page is paged out, then the NPT hook will redirect execution for some unknown memory page!!!*

<br>

*This diagram for SetNptHook()[an NPT hook(link to setnpthook)] is a lot easier to understand:*

<br>

![alt text](https://raw.githubusercontent.com/MellowNight/MellowNight.github.io/main/assets/img/NPTHookDiagram.png "Logo Title Text 1")


<br>
#### Really annoying problems

It took an absurd amount of time to get the NPT hooking feature to work properly. I spent months figuring out how to fix some nasty bugs: 

<br>

**Windows KVA shadowing crash:**  The KVA Shadow is a Windows mitigation for the "Meltdown" vulnerability. Two CR3 contexts are created for each process: Usermode dirbase and kernel dirbase. Invoking SetNptHook() from usermode caused the vmexit handler to crash, because the VMCB saved the usermode dirbase on vmexit, where AetherVisor's code wasn't even mapped. *Any process interfacing with AetherVisor must run as administrator to prevent this crash!*

<br>

**Split instruction hang/infinite loop:**  When two adjacent pages have conflicting execute permissions, an #NPF might occur from an instruction split across the page boundary. This will cause an infinite #NPF loop, because the instruction is never fully executable, so I had to figure out how to execute the entire instruction safely. (My current solution isn't optimal) I spent 24+ days debugging this!!*

<br>

**Hooking copy-on-write pages:**  I wanted to NPT hook functions in ntdll.dll and kernel32.dll, but setting an NPT hook does not trigger COW. why the fuck did I spend 2 weeks trying to point guest PTE to a copy of the hook page? I even went as far as to patch Windows memory management functions to prevent PFN inconsistency bugchecks 🤦‍♂️. It was foolish of me to try and recreate COW instead of just normally triggering COW (see TriggerCOWAndPageIn() (link)). 

<br>

### Sandboxing 

&emsp;&emsp;We just saw how we can mess with EPT/NPT entries to manipulate data exposed to the guest; you can also isolate memory regions and dynamically instrument read, write, and execute memory accesses. This serves as the basis for some EDR, software containerization, or RE/dynamic analysis solutions. For example, [Docker Desktop](https://docs.docker.com/desktop/faqs/linuxfaqs/#why-does-docker-desktop-for-linux-run-a-vm) uses KVM's EPT/NPT functionality to isolate containers. The concept behind AetherVisor's NPT sandbox is similar to that of [Alex Ionescu's Simpleator](https://github.com/ionescu007/Simpleator).  

<br>


#### Intercepting out-of-sandbox code execution

&emsp;&emsp;AetherVisor's sandbox feature isolates a memory region by disabling execute for its pages in the **"Primary"** nCR3 context. The sandboxed pages behave the same way as NPT hooked pages, but a third nCR3, named **"sandbox"**, is used for sandboxed pages instead of the **"shadow"** nCR3, used for NPT hooks. Whenever RIP leaves a sandbox region, the following events occur:


<br>
1. #NPF is thrown
2. Switch from **sandbox** context -> **primary** context
2. VMM sets RIP to a user-registered callback
3. Execute destination is pushed onto the stack; the instrumentation callback will return to this address
4. All registers are saved
5. guest execution resumes at the callback, in **primary"** context

[INSERT SANDBOX EXECUTE LOG PICTURE HERE]

<br>

This mechanism can be used to log the exceptions thrown or APIs called by a DLL.

<br>

#### Intercepting out-of-sandbox memory access

&emsp;&emsp;I wasn't able intercept every single memory read and write, because guest page table walks caused #NPF. I could only log reads and writes by denying read/write permissions on specific pages. I had to set up a fourth nCR3: **"all access"**, with every page mapped as RWX, to handle read/write instructions. Whenever a read/write instruction in the sandbox is blocked, the following events occur:

<br>

1. #NPF is thrown
2. Switch to special **"all access"** context
3. the read/write instruction is single-stepped 
2. Switch from **"all access"** context -> **"Primary"** context
4. VMM sets RIP to a user-registered callback
5. Execute destination is pushed onto the stack; the instrumentation callback will return to this address
6. All registers are saved
7. guest execution resumes at the callback, in **"Primary"** context

[INSERT SANDBOX READ/WRITE LOG PICTURE HERE]

<br>

This could be used to figure out detection vectors of an anti-cheat, such as the mapped driver traces they look for.

<br>

#### AetherVisor sandbox vs. other tools

Other projects use different methods to emulate pieces of code in a sandboxed environment, such as:

<br>

[**Qiling**](https://github.com/qilingframework/qiling) **and** [**Speakeasy**](https://github.com/mandiant/speakeasy): Uses a CPU emulator to intercept API calls, memory access, and more

[**KACE:**](https://github.com/waryas/KACE) Intercepts access to DLLs and system modules by blocking access to the DLLs, and using an exception handler to redirect access

[**Simpleator:**](https://github.com/ionescu007/Simpleator) Uses the Hyper-V API to create an isolated guest address space and logs Winapi calls

<br>

&emsp;&emsp;AetherVisor provides an advantage over these projects as it allows code to be sandboxed on-the-fly on real systems, without the need for emulating startup code, or setting up a fabricated system environment. Thus, AetherVisor is the only solution feasible for analyzing stateful software with multiple parts.

<br>

### Branch Tracing

&emsp;&emsp;The branch tracing feature in AetherVisor uses a combination of Last Branch Record (LBR) and Branch Trap Flag (BTF), to notify the VMM whenever a branch is executed.

<br>

&emsp;&emsp;The problem with my implementation is that #DB is thrown on every branch, causing a lot of overhead. I thought of collecting branch information in the LBR stack instead of single-stepping every branch, but on AMD, there's no way to signal when the LBR stack is full 😔😔. I considered using Lightweight Profiling (LWP), which has a lot more fine-grained controls for tracing instructions, but it only works in usermode. Still, LWP = a useful feature that can be added later.

<br>

&emsp;&emsp;When I wanted to test branch tracing, I struggled for hours due to the way VMware and Windows messed with the debugctl MSR.

&emsp;&emsp;First of all, VMware was forcing all debugctl bits to 0, which meant that I had to do some testing outside of VMware.

<br>

&emsp;&emsp;Secondly, Windows only enables LBR and BTF when the context is switched to a thread with DR7 bits 7 and 8 set, respectively (See KiRestoreDebugRegisterState or whatever). In this manner, Windows manages extended debug features, and my changes this debugctl are essentially ignored. 

<br>


### Process-specific syscall hooks


in progress...

<br>

## Future plans

I want to use AetherVisor's functionality to create projects like, DLL injectors, x64dbg extensions, or comprehensive HWID spoofers. If I ever decide to extend my hypervisor, I would add LWP, and I would harden it against anti-viruses/anti-cheats.

