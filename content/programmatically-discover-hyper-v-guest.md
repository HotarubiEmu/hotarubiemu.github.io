+++
title = "Programmatically Discover Hyper-V Guests"
date = 2022-10-17

[taxonomies]
tags = ["c++", "c", "x86", "virtual machines"]
+++

Hotarubi includes some fancy detection for virtual machine guests. There isn't any particular rationale for this other than I simply wanted to spend an evening working on it (I might have some ideas for it later down the road). I do imagine, however, there are other uses for this so I figured I'd write up a quick post about my not-so-straightforward journey in detecting Hyper-V guests.

> This post will focus specifically on Hyper-V but the ideas and methodology here can be applied to most any hypervisor.

Let's start with a simple explanation of a special cpu instruction called `cpuid`. It is used on x86 and x86-64 processors to request information about the cpu the code is currently executing on. The caller specifies a sub-function of the `cpuid` instruction by passing an integer value known as a "leaf" in the `eax` register. `cpuid` returns information pertaining to that leaf in the `eax`, `ebx`, `ecx` and `edx` registers.

> There are additionally sub-sub-functions known as "subleaves" passed via `ecx` but most of the time we want this value to be `0` for the default subleaf. None of the `cpuid` leafs I'm using in this post require subleaves but it's worth noting their existence.

Don't worry if this sounds a little low-level because most compilers support special functions called "intrinsics" that do the work of writing the assembly for you. Unfortunately, they're different depending on your compiler so I'll provide a quick generic version so we can be on our way:

```c
#include <stdbool.h>
#if defined(_MSC_VER)
#include <intrin.h> // __cpuid
#else
#include <cpuid.h> // __cpuid
#endif

// calls the cpuid instruction
// populates the eax, ebx, ecx, and edx variables with their respective values.
// this follows the GCC convention
void cpuid(int leaf, int* eax, int* ebx, int* ecx, int* edx)
{
#if defined(_MSC_VER) // MSVC
  int data[4];
  __cpuid(data, leaf);
  
  *eax = data[0];
  *ebx = data[1];
  *ecx = data[2];
  *edx = data[3];
#else // GCC and Clang
  __cpuid(leaf, eax, ebx, ecx, edx);
#endif
}
```

What is important to know is that each vendor (AMD, Intel, VIA, etc) maintains their own range of cpuid leaves with some overlap (for example, AMD and Intel often use each other's leaves to avoid the need for vendor-specific duplicates of certain features like detecting SSE4 support or obtaining the CPU model).

There is also a special range of leaves that both AMD and Intel essentially promise never to use because their functionality is resevered for a special purpose: **hypervisors**.

> "A hypervisor (also known as a virtual machine monitor, VMM, or virtualizer) is a type of computer software, firmware or hardware that creates and runs virtual machines. A computer on which a hypervisor runs one or more virtual machines is called a host machine, and each virtual machine is called a guest machine." -[source](https://en.wikipedia.org/wiki/Hypervisor)

The hypervisor range starts with leaf `0x40000000` which defines the highest accepted leaf in that range as well as a vendor name enocded as ascii in the 3 remaining 32-bit integers (which is to say, 12 characters).

Before we can use this range however, we need to check for the presence of a hypervisor. That is done by checking a specific bit of the `ecx` register of the `0x00000001` leaf:
```c
bool hypervisor_present(void)
{
  int eax, ebx, ecx, edx;
  // this function is almost gurenteed to be supported on any processor
  // made in the past 20 years so I'm assuming support for it here.
  cpuid(0x00000001, &eax, &ebx, &edx, &ecx);

  // check the last bit of ecx
  // the "hypervisor present" bit
  return (ecx >> 31) & 1;
}
```
This bit will tell us if we're in a virtual machine and whether we can use the hypervisor leaves to give us information about the environment the guest is running in. You might think to yourself "Well job well done, we've programmatically figured out that we're in a virtual machine. Post complete. Roll credits!".

Unfortunately, like many things in the land of Microsoft, it's not that simple. There is a curious property of some hypervisors: the host operating system runs as a virtual guest with extra privileges to manage other guests.

Such is the case for Microsoft's kernel-level hypervisor. This will be true so long as virtualization support is enabled in the bios. Yes, it will be true even if your Windows install is Home edition. Whenever you have cpu virtualization support enabled Windows will boot you into a special Hyper-V guest that will set the hypervisor bit to `1`.

So essentially there are two types of Hyper-V guests:
1. The "host" guest or "root partition" that manages all the other guests.
2. The "guest" guest or virtual machine.

> This is also apparently true for Zen though I haven't independently verified that for myself. Regardless, each hypervisor interfaces with the hypervisor bit and hypervisor leaves in their own way so consider reading the code or documentation for any you wish to detect.

So we need to query the hypervisor leaves for more information about if we're a root partition or not. Before we can do that, however, we need to make sure that we *can* query the hypervisor leaves. Accessing them otherwise is undefined as the values returned by unsupported leaves can vary by vendor and cpu model:

```c
bool hypervisor_leaf_supported(int leaf)
{
  // we aren't going to use these but they are the 12 ascii
  // characters of the vendor name of the hypervisor in order
  int vendor_0, vendor_1, vendor_2;
  
  // this number represents the highest leaf in the
  // 0x40000000 range we can use for future cpuid calls
  int highest_hypervisor_leaf;

  // no hypervisor and therefor no hypervisor leaves.
  // on Windows this is probably because you don't
  // have virtualization enabled in the bios.
  if (!hypervisor_present())
    return false;
    
  // ignore any leaves not in the 0x40000000 range
  if ((leaf & 0x40000000) != 0x40000000)
    return false;

  cpuid(0x40000000, &highest_hypervisor_leaf, &vendor_0, &vendor_1, &vendor_2);
  
  return leaf <= highest_hypervisor_leaf;
}
```
Our next step is to check leaf `0x40000001`. This leaf is designed to tell us about the hypervisor cpuid interface. It's important we query this information because unlike the Intel and AMD leaves, the leaves following `0x4000001` are subject to changes and differences depending on vendor for the same leaves.
```c
// integer representation of the ascii "Hv#1"
#define MS_HYPER_V 0x31237628

bool hypervisor_interface_hyperv(void)
{
  // 4-bytes of ascii representing the interface name
  int interface_signature;
  
  // reserved and unused
  int ebx, ecx, edx;

  // hyper-v implies this leaf is supported
  // so if it's not then it's not hyper-v
  if (!hypervisor_leaf_supported(0x40000001))
    return false;

  cpuid(0x40000001, &interface_signature, &ebx, &ecx, &edx);

  // you can opt for a string comparison here instead
  // just make sure you null terminate
  return interface_signature == MS_HYPER_V;
}
```
Now that we've got the boilerplate for detecting the Hyper-V cpuid interface and detecting hypervisor presence we need to query the Hyper-V cpuid interface for a certain value. There are two ways to do this but for now I'm going to stick to how the Linux kernel does it: [by checking for cpu management support](https://github.com/torvalds/linux/blob/f56dbdda4322d33d485f3d30f3aabba71de9098c/arch/x86/kernel/cpu/mshyperv.c#L288-L302).

In order to do that we need to check the [`HV_PARTITION_PRIVILEGE_MASK`](https://learn.microsoft.com/en-us/virtualization/hyper-v-on-windows/tlfs/datatypes/hv_partition_privilege_mask). Thankfully Microsoft has mapped it to `eax` and `ebx` of leaf `0x40000003`. `eax` contains the lower 32 bits and `ebx` contains the upper 32 bits. We need bit 43 which would be bit 11 of the upper 32 bits and therefor bit 11 of `ebx`:
```c
bool hyperv_host(void)
{
  // lower 32-bits of HV_PARTITION_PRIVILEGE_MASK
  int priv_mask_lower;
  
  // upper 32-bits of HV_PARTITION_PRIVILEGE_MASK
  int priv_mask_upper;
  
  // unused
  int ecx, edx;

  // no hyper-v hypervisor, so not a hyper-v guest
  if (!hypervisor_interface_hyperv())
    return true;
  
  // we're hyper-v and support for this leaf is implied
  cpuid(0x40000003, &priv_mask_lower, &priv_mask_upper, &ecx, &edx);
  
  // the cpu management bit of HV_PARTITION_PRIVILEGE_MASK
  return (priv_mask_upper >> 11) & 1;
}
```
This is the magic bit that is only set when you are running code as the Hyper-V host. So if this bit is not set, then we are a guest.

```c
int main()
{
  if (hyperv_host())
    printf("I'm the host!");
  else
    printf("I'm the guest!");
  
  return 0;
}
```
And with that our yak is sufficiently shaved.