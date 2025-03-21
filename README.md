# Abstract

My objective is to run [LM Studio](https://lmstudio.ai/) and [Comfy UI](https://github.com/comfyanonymous/ComfyUI) under windows with full GPU acceleration.

With an Nvidia GPUs this is trivial to achieve. The applications work out of the box with an installer/script.

With AMD ROCm this is hardcore, it took me a month to get the software stack and performance figured out. It took dozens of rebuilds only one led me to a working toolchain. I'm not sure how repeatable this is, adrenaline and windows updates are liable to break the toolchain.

The scope of this repository is to document is to remind the future me how I made ROCm acceleration work, and hopefully give useful pointers to other ROCm user on which debug steps to take to get ROCm to run.

DISCLAIMER: FOLLOW THE STEPS IN THIS DOCUMENT AT YOUR OWN RISK

## Why?

My RTX3080 10GB just can't run bigger models, and my ambitions are increasing, i want to run bleeding edge and larger LLMs and Diffusion models.

2025 has seen another GPU shortage. The Nvidia 4000 and 5000 series are rare and overpriced.

The 7900XTX 24GB is a 930 €. A research led me to believe AMD had ROCm figured out, so I bought that.

### My System specs

- Windows 11
- [RX 7900XTX](https://www.techpowerup.com/gpu-specs/radeon-rx-7900-xtx.c3941)
- Intel 13700F
- 64GB DDR5 6400 CL40

With adrenaline 25.1 and 25.3 HIP 6.2.4 under windows, HIP 6.3 under WSL2 Ubuntu 22.

IMPORTANT: DO NOT SKIMP ON RAM this is true for bith Nvidia and AMD. With 24GB VRAM, the model still passes through primary memory. 32GB would leave 8GB for everything else which is deficient, and you do not want to swap. RAM is so much cheaper than the GPU that you have no excuse. 64GB is the minimum, I rutinely use over 55GB with Flux. I'm considering going to 96GB or 128GB.

### Positives

Adrenaline works fine. If you just game, AMD cards work well enough.

Price/performance of the 7900XTX when ROCm is properly accelerating is AMAZING. Easily 2X to 3X better than Nvidia.

I get the 17GB Flux FP8 model to render in 60s 20 step, and I can get up to 100 tokens per second on 14B llm models which is AMAZING performance.

It's possible to make ROCm acceleration work.

### Negatives

You'll see online performance benchmarks that are an insignificant fraction of what I reported.

It's because AMD acceleration is as fragmented as Linux distributions are. You have OpenCL, DirectML, Vulkan, ROCm, etc... NONE, and I mean N O N E of it works well. People get something to work, like [Amuse](https://www.amuse-ai.com/) on DirectML and reports 1/20th of the performance of an Nvidia card.

I spent a month refusing of giving up, and got ComfyUI to work under WSL2 Ubuntu22 with ROCm accelerating Pytorch, and I get 17GB Flux FP8  model to render in 60s 20 step which is AMAZING performance.

The biggest hurdle, is the same linux has. You'll find dozens of guides, none of which will work because something changed. This guide you are reading is included in the list. I can't guarantee you'll follow my guide and it'll work for you. It's all so brittle...

It's a journey, and discovering this took an enormous amount of effort on my side with support from many strangers and AMD. It had me question my sanity and why wasn't I just buying an Nvidia GPU multiple times.

I'm not sure how long it will last, I'm afraid every adrenaline update will brick my toolchain, like going from 25.1 to 25.3 briefly did.

Many told me to dual boot linux, or use linux, which is ludicrous, the idea of rebooting to run an aplication. There is a reason most users use windows, and it's because often applications works by default on there without much fiddling. I want to be a USER of ROCm accelerated applications, not a developer of ROCm. The only acceptable outcome for AMD should be to have a one click installer like A1111/Forge/SD Next for CUDA under windows, where you double click, and it works out of the box.

# Driver

First thing is to get Adrenaline and ROCm set up and working.

NOTE: I tried literal dozens of guides over a month, some will ask to install older HIP/Adrenaline version. That's bad. You cannot settle for anything but the latest release AND you cannot settle for anything but the mainline of your application. If you do, you might get some of it to work, but some pieces of the acceleration will just not work, and it will get worse and worse the further behind the mainline you are. You are wasting time if you use a fork behind the mainline that uses some hack.

## STEP 0A - Compatibility matricies

AMD windows mainline is behind the linux mainline, and it can take a very long time to provide ROCm support to new architectures. Check the compatibility matricies to see if your GPU has a driver and ROCm runtime available.

You'd think that since Nvidia CUDA is called CUDA and you download CUDA toolkit, AMD is the same. It is not. ROCm isn't called ROCm, but it's called HIP SDK and that's what you download to run ROCm acceleration.

Same with the GPU, they have different names. It's not the 7900XTX, but is GX1100 under HIP.

Check that your GPU has compatibility with HIP, and discover what the GX name for your GPU is from this table under the AMD Radeon tab [Windows compatibility matrix](https://rocm.docs.amd.com/projects/install-on-windows/en/latest/reference/system-requirements.html)

Knowing the GX number for your GPU check that it has support under linux and the versions. [Adrenaline ROCM Compatibility matrix](https://rocm.docs.amd.com/en/latest/compatibility/compatibility-matrix.html)

If there is windows compatibility for HIP for your card under windows, you have hope and you can proceed to the next step.

## STEP 1A - DDU

ROCm is really brittle, you need to take great care that driver remants aren't bricking your ROCm runtime.

- Download [DDU](https://www.guru3d.com/download/display-driver-uninstaller-download/)
- Disconnect your computer from the internet
- Stop windows updates for now
- Uninstall everything GPU related like Adrenaline, old HIP versions, or if you switched cards like me all Geforce/Nvidia applications
- Run DDU and cleanly wipe your graphics drivers. NOTE: you need to start in safe mode, [look at a guide if you have doubt](https://www.youtube.com/watch?v=xn8z39tiEL0)
- Reboot in normal mode
- Reconnect the internet.
- I would resume windows updates AFTER the ROCm acceleration is working.

NOTE: this is really important and made me lose two weeks because some pieces of Nvidia drivers were left behind, likely a windows update after DDU

## STEP 1B - Adrenaline

[Download and install Adrenaline](https://www.amd.com/en/support/download/drivers.html)

If you use your computer for things that are not ROCm related, like gaming, you want the latest adrenaline AND staying up to date.

In my case, I had to download a preview of the next adrenaline 25.1 when the latest was 24 to get ROCm running. Also, update to Adrenaline 25.3 bricked for a few reboots ROCm.

It is extremely brittle. I do not know if the latest adrenaline you download will let you accelerate ROCm.

Reboot.

## STEP 1C - HIP

Next step is to install the ROCm runtime.

[HIP Download page](https://www.amd.com/en/developer/resources/rocm-hub/hip-sdk.html)

Download the latest HIP compatible with your GX card and install it.

Now you need to set some environment variables that points to where you installed ROCm:

HIP_PATH C:\Program Files\AMD\ROCm\6.2\
HIP_PATH_62 C:\Program Files\AMD\ROCm\6.2\

![](/images/HIP_environment_var.png)

If there is other HIP but that, delete them and make sure those are uninstalled as per step 1A. It's really important, the acceleration stack will get confused about the binaries, it needs those system vars.

Reboot.

## STEP 1D - CHECK HIP

If this worked, HIP should be visible system wide, and  you can run a cmd line that will tell you about your card.

NOTE: This is really brittle. There might be shenanigans if you have an iGPU. I have a F processor without iGPU and I didn't have trouble due to that, in case you might consider disabling the iGPU from the bios. 

```
C:\Users\FatherOfMachines>hipinfo

--------------------------------------------------------------------------------
device#                           0
Name:                             AMD Radeon RX 7900 XTX
pciBusID:                         3
pciDeviceID:                      0
pciDomainID:                      0
multiProcessorCount:              48
maxThreadsPerMultiProcessor:      2048
isMultiGpuBoard:                  0
clockRate:                        2482 Mhz
memoryClockRate:                  1250 Mhz
memoryBusWidth:                   0
totalGlobalMem:                   23.98 GB
totalConstMem:                    2147483647
sharedMemPerBlock:                64.00 KB
canMapHostMemory:                 1
regsPerBlock:                     0
warpSize:                         32
l2CacheSize:                      4194304
computeMode:                      0
maxThreadsPerBlock:               1024
maxThreadsDim.x:                  1024
maxThreadsDim.y:                  1024
maxThreadsDim.z:                  1024
maxGridSize.x:                    2147483647
maxGridSize.y:                    65536
maxGridSize.z:                    65536
major:                            11
minor:                            0
concurrentKernels:                1
cooperativeLaunch:                0
cooperativeMultiDeviceLaunch:     0
isIntegrated:                     0
maxTexture1D:                     16384
maxTexture2D.width:               16384
maxTexture2D.height:              16384
maxTexture3D.width:               2048
maxTexture3D.height:              2048
maxTexture3D.depth:               2048
hostNativeAtomicSupported:        1
isLargeBar:                       0
asicRevision:                     0
maxSharedMemoryPerMultiProcessor: 64.00 KB
clockInstructionRate:             1000.00 Mhz
arch.hasGlobalInt32Atomics:       1
arch.hasGlobalFloatAtomicExch:    1
arch.hasSharedInt32Atomics:       1
arch.hasSharedFloatAtomicExch:    1
arch.hasFloatAtomicAdd:           1
arch.hasGlobalInt64Atomics:       1
arch.hasSharedInt64Atomics:       1
arch.hasDoubles:                  1
arch.hasWarpVote:                 1
arch.hasWarpBallot:               1
arch.hasWarpShuffle:              1
arch.hasFunnelShift:              0
arch.hasThreadFenceSystem:        1
arch.hasSyncThreadsExt:           0
arch.hasSurfaceFuncs:             0
arch.has3dGrid:                   1
arch.hasDynamicParallelism:       0
gcnArchName:                      gfx1100
peers:
non-peers:                        device#0

memInfo.total:                    23.98 GB
memInfo.free:                     23.84 GB (99%)
```

If you got here, congratulation. This is a big step.

# LM Studio

I prefer LM Studio to ollama because it has a much better UI, has the localhost model provider API and has a curated model download section. It also has a runtime section to select the acceleration.

[Download latest LM studio](https://lmstudio.ai/)

Go to the runtime tab, and select the ROCm Runtime.

![](/images/LM-Studio-Runtime.png)

Good news, is that LM Studio works easily under windows ROCm natively. The Vulkan runtime doesn't even need HIP, but for me it leaves about 1/2 performance on the table.

### Troubleshooting

[If you have issues, you can ask help on Github. It did not help me, but it might help you.](https://github.com/lmstudio-ai/lmstudio-bug-tracker/issues)

E.g. for me LM Studio ROCm runtime refused to work with this error:

```
Failed to load LLM engine from path: C:\Users\FatherOfMachines\.cache\lm-studio\extensions\backends\llama.cpp-win-x86_64-amd-rocm-avx2-1.11.0\llm_engine_rocm.node. \\?\C:\Users\FatherOfMachines\.cache\lm-studio\extensions\backends\llama.cpp-win-x86_64-amd-rocm-avx2-1.11.0\llm_engine_rocm.node is not a valid Win32 application.
\\?\C:\Users\FatherOfMachines\.cache\lm-studio\extensions\backends\llama.cpp-win-x86_64-amd-rocm-avx2-1.11.0\llm_engine_rocm.node
```

Solution for me was to do STEP1A, uninstall LM Studio, and it turns out LM studio leaves some leftover remants that aren't removed by ununstall under the user, so wipe the following folders manually. There might be others.

```
C:\Users\>YOUR USER<\.cache\lm-studio
C:\Users\>YOUR USER<\AppData\Local\lm-studio-updater
C:\Users\>YOUR USER<\AppData\Roaming\LM Studio
```

# Comfy UI

Support for latest diffusion models comes first to Comfy UI, and later to other UIs like A1111 Forge, SD Next. I started there but moved to Comfy UI because I want the latest models. E.g. I'm having fun with 3D model generation for D&D, and Video generation.

This is a lot harder than LM Studio because you need your python application pytorch to be accelerated by ROCm, which is hardcore. It is not for the faint of heart.

[AMD has a guide to get a WSL2 Ubuntu 22 ROCm Python 3.10 Pytorch going](https://rocm.docs.amd.com/projects/radeon/en/latest/docs/install/wsl/install-radeon.html) There are some quirks to get fixed that aren't obvious from the guide.


### Comfy UI repository

You'll find there are literally dozens of Comfy UI forks with twice as many guides instructing you to git clone them.

There aren't. As far as you are concerned, there is only ONE Comfy UI repository. [The objective is to run the main fork of Comfy UI.](https://github.com/comfyanonymous/ComfyUI) Do not settle for anything less or you are wasting time. I mean it. Nothing is more frustrating than getting a ComfyUI fork to diffuse well with Flux only to discover you cannot run Wan 2.1 or Hunyuan 3D because the support requires something that is not in your Zluda fork, or Zluda doesn't map one of the countless calls, and you have to redo everything.

### Toolchain

It is IMPOSSIBLE to get proper Comfy UI ROCm acceleration running natively under windows reliably. It cannot be done. You can get chunks of it running with Stability Matrix, Zluda and whatnot, but eventually some nodes you need for your workflow won't accelerate and there is nothing you can do about it. You are wasting time.

As far as you are concerned, there is only ONLY way to run Comfy UI, which is also the ONLY toolchain that AMD supports. WSL2. Meaning you are running a VM.

The crux of the issue is that there aren't reliable pytorch binaries for AMD ROCm like there are pytorch binaries for Nvidia CUDA. There is no workaround for this. The binaries aren't there.

Running ROCM under WSL2 is actually faster than running ROCm under windows with Zluda translation at least 30% faster for me. And WSL2 has much wider coverage for pytorch calls, meaning way more nodes work.

This being a VM, means you have a RAM penality, and a penality moving data in and out of the VM.

### STEP 1 - WSL

[Microsoft Guide to get WSL2 in your system](https://learn.microsoft.com/en-us/windows/wsl/install)

The first thing not obvious from Microsoft guide is the VM configuration. I gave far more RAM to the VM, reduced the swap size and increased the fule system. The rest worked fine, but if you are having trouble with detecting GPU you might want to check the optional features to make sure your system is passing your GPU through to the VM. Needless to say the GPU needs to work under windows to have a chance to work under the VM.

IDEA: the swap is a swap file, I'll experiment with other settings.

![](/images/WSL-Settings.png)

To install Ubuntu 22 there are a number of ways. I advise to use this command line, as the windows store is incredibly slow for me, taking up to 8h to install Ubuntu 22.

```
wsl --list

wsl --install -d Ubuntu-22.04 --web-download
```

Now you can run wsl and set your username inside the VM

```
PS C:\Users\FatherOfMachines> wsl --install --distribution Ubuntu-22.04 --web-download
Ubuntu 22.04 LTS is already installed.
Launching Ubuntu 22.04 LTS...
The requested operation is successful. Changes will not be effective until the system is rebooted.
PS C:\Users\FatherOfMachines>

Installing, this may take a few minutes...
Please create a default UNIX user account. The username does not need to match your Windows username.
For more information visit: https://aka.ms/wslusers
Enter new UNIX username: soraka
New password:
Retype new password:
passwd: password updated successfully
The operation completed successfully.
Installation successful!
To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

Welcome to Ubuntu 22.04.2 LTS (GNU/Linux 5.15.167.4-microsoft-standard-WSL2 x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage


This message is shown once a day. To disable it please create the
/home/soraka/.hushlogin file.
soraka@TowerOfBabel:~$ sudo shutdown now
```

Now, this will install the VHD under C drive, which for me bricked it hard later because I didn't have the space. So, I move the VHD to another drive by using a junky series of command to export and import the VM.

```
wsl --shutdown

wsl --export Ubuntu-22.04 "F:\WSL-Ubuntu22\WSL-Ubuntu22.tar"

wsl --unregister Ubuntu-22.04

wsl --import Ubuntu-22.04 "F:\WSL-Ubuntu22" "F:\WSL-Ubuntu22\WSL-Ubuntu22.tar"

wsl
```

Now you should have a WSL2 Ubuntu22 with a ext4.vhdx on a big drive.

NOTE: WSL2 is brittle as well. Sometimes launching it give error and it's not logged in as user. Run the command to be in users, I bricked one attempt because it was root.

```
su username

cd
```

NOTE: It's not obvious, but under WSL2 the host machine is mounted under \mnt\DRIVE LETTE  which makes it easier to move file with mv and cp but risks bricking installations because apt permissions on that drive


### STEP 2 - Install ROCm under WSL2

You installed ROCm under windows, now you install ROCm under WSL2 Ubuntu 22 following AMD guide. READ CAREFULLY. There are IF conditions and make sure you do for Ubuntu 22 and not 24. Make sure you are logged in as user and on the home folder.

[AMD WSL2 Guide](https://rocm.docs.amd.com/projects/radeon/en/latest/docs/install/wsl/install-radeon.html)

READ CAREFULLY. PyTorch via PIP

[AMD WSL2 Pytorch guide](https://rocm.docs.amd.com/projects/radeon/en/latest/docs/install/wsl/install-pytorch.html)

This guide works pretty well, it reliably detects the card for me.

Below, the full commands for you to check at any point if acceleration is working under WSL2, very useful to see if it's a problem with your workflow or with the underlying accelerations and things are starting to break.

```
Microsoft Windows [Version 10.0.22631.4602]
(c) Microsoft Corporation. All rights reserved.

C:\Users\FatherOfMachines>wsl
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 5.15.167.4-microsoft-standard-WSL2 x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 System information as of Fri Mar 21 08:43:21 CET 2025

  System load:  0.0                  Processes:             70
  Usage of /:   12.4% of 1006.85GB   Users logged in:       0
  Memory usage: 1%                   IPv4 address for eth0: 172.28.206.241
  Swap usage:   0%

 * Strictly confined Kubernetes makes edge and IoT secure. Learn how MicroK8s
   just raised the bar for easy, resilient and secure K8s cluster deployment.

   https://ubuntu.com/engage/secure-kubernetes-at-the-edge

3 updates could not be installed automatically. For more details,
see /var/log/unattended-upgrades/unattended-upgrades.log

This message is shown once a day. To disable it please create the
/root/.hushlogin file.
root@TowerOfBabel:/mnt/c/Users/FatherOfMachines# su soraka
soraka@TowerOfBabel:~$ cd
soraka@TowerOfBabel:~$ rocminfo
WSL environment detected.
=====================
HSA System Attributes
=====================
Runtime Version:         1.1
Runtime Ext Version:     1.6
System Timestamp Freq.:  1000.000000MHz
Sig. Max Wait Duration:  18446744073709551615 (0xFFFFFFFFFFFFFFFF) (timestamp count)
Machine Model:           LARGE
System Endianness:       LITTLE
Mwaitx:                  DISABLED
DMAbuf Support:          YES

==========
HSA Agents
==========
*******
Agent 1
*******
  Name:                    13th Gen Intel(R) Core(TM) i7-13700F
  Uuid:                    CPU-XX
  Marketing Name:          13th Gen Intel(R) Core(TM) i7-13700F
  Vendor Name:             CPU
  Feature:                 None specified
  Profile:                 FULL_PROFILE
  Float Round Mode:        NEAR
  Max Queue Number:        0(0x0)
  Queue Min Size:          0(0x0)
  Queue Max Size:          0(0x0)
  Queue Type:              MULTI
  Node:                    0
  Device Type:             CPU
  Cache Info:
    L1:                      49152(0xc000) KB
  Chip ID:                 0(0x0)
  Cacheline Size:          64(0x40)
  Internal Node ID:        0
  Compute Unit:            24
  SIMDs per CU:            0
  Shader Engines:          0
  Shader Arrs. per Eng.:   0
  Memory Properties:
  Features:                None
  Pool Info:
    Pool 1
      Segment:                 GLOBAL; FLAGS: FINE GRAINED
      Size:                    56235500(0x35a15ec) KB
      Allocatable:             TRUE
      Alloc Granule:           4KB
      Alloc Recommended Granule:4KB
      Alloc Alignment:         4KB
      Accessible by all:       TRUE
    Pool 2
      Segment:                 GLOBAL; FLAGS: EXTENDED FINE GRAINED
      Size:                    56235500(0x35a15ec) KB
      Allocatable:             TRUE
      Alloc Granule:           4KB
      Alloc Recommended Granule:4KB
      Alloc Alignment:         4KB
      Accessible by all:       TRUE
    Pool 3
      Segment:                 GLOBAL; FLAGS: KERNARG, FINE GRAINED
      Size:                    56235500(0x35a15ec) KB
      Allocatable:             TRUE
      Alloc Granule:           4KB
      Alloc Recommended Granule:4KB
      Alloc Alignment:         4KB
      Accessible by all:       TRUE
    Pool 4
      Segment:                 GLOBAL; FLAGS: COARSE GRAINED
      Size:                    56235500(0x35a15ec) KB
      Allocatable:             TRUE
      Alloc Granule:           4KB
      Alloc Recommended Granule:4KB
      Alloc Alignment:         4KB
      Accessible by all:       TRUE
  ISA Info:
*******
Agent 2
*******
  Name:                    gfx1100
  Marketing Name:          AMD Radeon RX 7900 XTX
  Vendor Name:             AMD
  Feature:                 KERNEL_DISPATCH
  Profile:                 BASE_PROFILE
  Float Round Mode:        NEAR
  Max Queue Number:        128(0x80)
  Queue Min Size:          64(0x40)
  Queue Max Size:          131072(0x20000)
  Queue Type:              MULTI
  Node:                    1
  Device Type:             GPU
  Cache Info:
    L1:                      32(0x20) KB
    L2:                      6144(0x1800) KB
    L3:                      98304(0x18000) KB
  Chip ID:                 29772(0x744c)
  Cacheline Size:          64(0x40)
  Max Clock Freq. (MHz):   2482
  Internal Node ID:        1
  Compute Unit:            96
  SIMDs per CU:            2
  Shader Engines:          6
  Shader Arrs. per Eng.:   2
  Coherent Host Access:    FALSE
  Memory Properties:
  Features:                KERNEL_DISPATCH
  Fast F16 Operation:      TRUE
  Wavefront Size:          32(0x20)
  Workgroup Max Size:      1024(0x400)
  Workgroup Max Size per Dimension:
    x                        1024(0x400)
    y                        1024(0x400)
    z                        1024(0x400)
  Max Waves Per CU:        32(0x20)
  Max Work-item Per CU:    1024(0x400)
  Grid Max Size:           4294967295(0xffffffff)
  Grid Max Size per Dimension:
    x                        4294967295(0xffffffff)
    y                        4294967295(0xffffffff)
    z                        4294967295(0xffffffff)
  Max fbarriers/Workgrp:   32
  Packet Processor uCode:: 372
  SDMA engine uCode::      24
  IOMMU Support::          None
  Pool Info:
    Pool 1
      Segment:                 GLOBAL; FLAGS: COARSE GRAINED
      Size:                    25079456(0x17eaea0) KB
      Allocatable:             TRUE
      Alloc Granule:           4KB
      Alloc Recommended Granule:2048KB
      Alloc Alignment:         4KB
      Accessible by all:       FALSE
    Pool 2
      Segment:                 GLOBAL; FLAGS: EXTENDED FINE GRAINED
      Size:                    25079456(0x17eaea0) KB
      Allocatable:             TRUE
      Alloc Granule:           4KB
      Alloc Recommended Granule:2048KB
      Alloc Alignment:         4KB
      Accessible by all:       FALSE
    Pool 3
      Segment:                 GROUP
      Size:                    64(0x40) KB
      Allocatable:             FALSE
      Alloc Granule:           0KB
      Alloc Recommended Granule:0KB
      Alloc Alignment:         0KB
      Accessible by all:       FALSE
  ISA Info:
    ISA 1
      Name:                    amdgcn-amd-amdhsa--gfx1100
      Machine Models:          HSA_MACHINE_MODEL_LARGE
      Profiles:                HSA_PROFILE_BASE
      Default Rounding Mode:   NEAR
      Default Rounding Mode:   NEAR
      Fast f16:                TRUE
      Workgroup Max Size:      1024(0x400)
      Workgroup Max Size per Dimension:
        x                        1024(0x400)
        y                        1024(0x400)
        z                        1024(0x400)
      Grid Max Size:           4294967295(0xffffffff)
      Grid Max Size per Dimension:
        x                        4294967295(0xffffffff)
        y                        4294967295(0xffffffff)
        z                        4294967295(0xffffffff)
      FBarrier Max Size:       32
*** Done ***
soraka@TowerOfBabel:~$ python3 -c 'import torch' 2> /dev/null && echo 'Success' || echo 'Failure'
Success
soraka@TowerOfBabel:~$ python3 -c 'import torch; print(torch.cuda.is_available())'
True
soraka@TowerOfBabel:~$ python3 -c "import torch; print(f'device name [0]:', torch.cuda.get_device_name(0))"
/home/soraka/.local/lib/python3.10/site-packages/torch/cuda/__init__.py:645: UserWarning: Can't initialize amdsmi - Error code: 34
  warnings.warn(f"Can't initialize amdsmi - Error code: {e.err_code}")
device name [0]: AMD Radeon RX 7900 XTX
soraka@TowerOfBabel:~$ python3 -m torch.utils.collect_env
/usr/lib/python3.10/runpy.py:126: RuntimeWarning: 'torch.utils.collect_env' found in sys.modules after import of package 'torch.utils', but prior to execution of 'torch.utils.collect_env'; this may result in unpredictable behaviour
  warn(RuntimeWarning(msg))
Collecting environment information...
/home/soraka/.local/lib/python3.10/site-packages/torch/cuda/__init__.py:645: UserWarning: Can't initialize amdsmi - Error code: 34
  warnings.warn(f"Can't initialize amdsmi - Error code: {e.err_code}")
PyTorch version: 2.4.0+rocm6.3.4.git7cecbf6d
Is debug build: False
CUDA used to build PyTorch: N/A
ROCM used to build PyTorch: 6.3.42134-a9a80e791

OS: Ubuntu 22.04.5 LTS (x86_64)
GCC version: (Ubuntu 11.4.0-1ubuntu1~22.04) 11.4.0
Clang version: Could not collect
CMake version: Could not collect
Libc version: glibc-2.35

Python version: 3.10.12 (main, Feb  4 2025, 14:57:36) [GCC 11.4.0] (64-bit runtime)
Python platform: Linux-5.15.167.4-microsoft-standard-WSL2-x86_64-with-glibc2.35
Is CUDA available: True
CUDA runtime version: Could not collect
CUDA_MODULE_LOADING set to: LAZY
GPU models and configuration: AMD Radeon RX 7900 XTX (gfx1100)
Nvidia driver version: Could not collect
cuDNN version: Could not collect
HIP runtime version: 6.3.42134
MIOpen runtime version: 3.3.0
Is XNNPACK available: True

CPU:
Architecture:                         x86_64
CPU op-mode(s):                       32-bit, 64-bit
Address sizes:                        39 bits physical, 48 bits virtual
Byte Order:                           Little Endian
CPU(s):                               24
On-line CPU(s) list:                  0-23
Vendor ID:                            GenuineIntel
Model name:                           13th Gen Intel(R) Core(TM) i7-13700F
CPU family:                           6
Model:                                183
Thread(s) per core:                   2
Core(s) per socket:                   12
Socket(s):                            1
Stepping:                             1
BogoMIPS:                             4224.01
Flags:                                fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ss ht syscall nx pdpe1gb rdtscp lm constant_tsc rep_good nopl xtopology tsc_reliable nonstop_tsc cpuid pni pclmulqdq vmx ssse3 fma cx16 pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand hypervisor lahf_lm abm 3dnowprefetch invpcid_single ssbd ibrs ibpb stibp ibrs_enhanced tpr_shadow vnmi ept vpid ept_ad fsgsbase tsc_adjust bmi1 avx2 smep bmi2 erms invpcid rdseed adx smap clflushopt clwb sha_ni xsaveopt xsavec xgetbv1 xsaves avx_vnni umip waitpkg gfni vaes vpclmulqdq rdpid movdiri movdir64b fsrm md_clear serialize flush_l1d arch_capabilities
Virtualization:                       VT-x
Hypervisor vendor:                    Microsoft
Virtualization type:                  full
L1d cache:                            576 KiB (12 instances)
L1i cache:                            384 KiB (12 instances)
L2 cache:                             24 MiB (12 instances)
L3 cache:                             30 MiB (1 instance)
Vulnerability Gather data sampling:   Not affected
Vulnerability Itlb multihit:          Not affected
Vulnerability L1tf:                   Not affected
Vulnerability Mds:                    Not affected
Vulnerability Meltdown:               Not affected
Vulnerability Mmio stale data:        Not affected
Vulnerability Reg file data sampling: Mitigation; Clear Register File
Vulnerability Retbleed:               Mitigation; Enhanced IBRS
Vulnerability Spec rstack overflow:   Not affected
Vulnerability Spec store bypass:      Mitigation; Speculative Store Bypass disabled via prctl and seccomp
Vulnerability Spectre v1:             Mitigation; usercopy/swapgs barriers and __user pointer sanitization
Vulnerability Spectre v2:             Mitigation; Enhanced / Automatic IBRS; IBPB conditional; RSB filling; PBRSB-eIBRS SW sequence; BHI BHI_DIS_S
Vulnerability Srbds:                  Not affected
Vulnerability Tsx async abort:        Not affected

Versions of relevant libraries:
[pip3] mypy-extensions==1.0.0
[pip3] numpy==1.26.4
[pip3] pytorch-triton-rocm==3.0.0+rocm6.3.4.git75cc27c2
[pip3] torch==2.4.0+rocm6.3.4.git7cecbf6d
[pip3] torchaudio==2.4.0+rocm6.3.4.git69d40773
[pip3] torchsde==0.2.6
[pip3] torchvision==0.19.0+rocm6.3.4.gitfab84886
[conda] Could not collect
soraka@TowerOfBabel:~$
```

### STEP 2 TROUBLESHOOTING:

I got this error, and got past it by running the commands one at a time instead of bundling them like in the instructions

```
ERROR: Could not install packages due to an OSError: [Errno 2] No such file or directory: '/home/soraka/torch-2.4.0+rocm6.3.4.git7cecbf6d-cp310-cp310-linux_x86_64.whl'
```


### STEP 3 - Comfy UI

[Comfy UI repository](https://github.com/comfyanonymous/ComfyUI)

Now git clone the Comfy UI

```
git clone https://github.com/comfyanonymous/ComfyUI.git
cd ComfyUI/
cat requirements.txt
pip install -r requirements.txt
cd
python3 ComfyUI/main.py
```

This finally gets Comfy UI to launch. Go to the browser in your host machine, and you should see Comfy UI.

NOTE: Look at the WSL2 command line and wait until it's done loading all modules before clicking "Queue" to run a generation.

```
To see the GUI go to: http://127.0.0.1:8188
```

CTRL+C whenever you want to close.

I usually launch cmd, then run WSL as command, so that when I exit, i still have the terminal open if I want to run commands.

![](/images/ComfyUI.png)

exit comfy UI, install Comfy UI manager, you aren't going anywhere without that core extension

```
cd
cd ComfyUI/
cd custom_nodes/
git clone https://github.com/ltdrdata/ComfyUI-Manager comfyui-manager
python3 ComfyUI/main.py
```

### STEP 4 -Run Comfy UI inside WSL2 move models, and generate the first image

The great news is that you are running on a browser of the host machine, meaning you don't really care about WSL2, an can import and export workflow and images by drag and drop, and right click save. It's pretty painless to use.

The good news is that WSL2 mounts the host drives under /mnt

The bad news is that moving models from the host machine to WSL2 requires command line, and can't be done with the aplication running, unless you do fancy linux command line wizary to launch comfy ui as background application the kill the PID when you are done.


```
wsl
su soraka
cp /mnt/f/SD-Zluda/ComfyUI/models/checkpoints/flux1-dev-fp8-full.safetensors /home/soraka/ComfyUI/models/checkpoints
cp /mnt/f/SD-Zluda/ComfyUI/models/checkpoints/RMSD-XL-Aries-Fantasy.safetensors /home/soraka/ComfyUI/models/checkpoints
python3 ComfyUI/main.py

```

NOTE: ComfyUI expect models to be in specific directories. Checkpoint usually work for SD models. There are things like diffusion models, text embeddings, clip, VAE, and so much more.

You can now run a basic workflow, press QUEUE button, and you should be diffusing images in the most basic of fashion.

NOTE: the image below is a PNG with a workflow. You can drag it inside ComfyUI and it will open the workflow. Neat!

![](/images/basic-workflow.png)

```
soraka@TowerOfBabel:~$ python3 ComfyUI/main.py
[START] Security scan
[DONE] Security scan
## ComfyUI-Manager: installing dependencies done.
** ComfyUI startup time: 2025-03-21 11:57:25.617
** Platform: Linux
** Python version: 3.10.12 (main, Feb  4 2025, 14:57:36) [GCC 11.4.0]
** Python executable: /usr/bin/python3
** ComfyUI Path: /home/soraka/ComfyUI
** ComfyUI Base Folder Path: /home/soraka/ComfyUI
** User directory: /home/soraka/ComfyUI/user
** ComfyUI-Manager config path: /home/soraka/ComfyUI/user/default/ComfyUI-Manager/config.ini
** Log path: /home/soraka/ComfyUI/user/comfyui.log
[ComfyUI-Manager] PyTorch is not installed
ERROR: Could not install packages due to an OSError: [Errno 13] Permission denied: '/usr/local/lib/python3.10/dist-packages/comfyui_frontend_package'
Consider using the `--user` option or check the permissions.

[ComfyUI-Manager] Failed to restore comfyui-frontend-package
Command '['/usr/bin/python3', '-s', '-m', 'pip', 'install', 'comfyui-frontend-package==1.11.8']' returned non-zero exit status 1.

Prestartup times for custom nodes:
   1.6 seconds: /home/soraka/ComfyUI/custom_nodes/comfyui-manager

Checkpoint files will always be loaded safely.
Total VRAM 24492 MB, total RAM 54917 MB
pytorch version: 2.4.0+rocm6.3.4.git7cecbf6d
/home/soraka/.local/lib/python3.10/site-packages/torch/cuda/__init__.py:645: UserWarning: Can't initialize amdsmi - Error code: 34
  warnings.warn(f"Can't initialize amdsmi - Error code: {e.err_code}")
AMD arch: gfx1100
Set vram state to: NORMAL_VRAM
Device: cuda:0 AMD Radeon RX 7900 XTX : native
Using sub quadratic optimization for attention, if you have memory or speed issues try using: --use-split-cross-attention
ComfyUI version: 0.3.25
ComfyUI frontend version: 1.11.8
[Prompt Server] web root: /home/soraka/.local/lib/python3.10/site-packages/comfyui_frontend_package/static
[Crystools INFO] Crystools version: 1.22.1
[Crystools INFO] CPU: 13th Gen Intel(R) Core(TM) i7-13700F - Arch: x86_64 - OS: Linux 5.15.167.4-microsoft-standard-WSL2
[Crystools ERROR] Could not init pynvml (Nvidia).NVML Shared Library Not Found
[Crystools WARNING] No GPU with CUDA detected.
xFormers not available
xFormers not available
Web extensions folder found at /home/soraka/ComfyUI/web/extensions/ComfyLiterals
WAS Node Suite: OpenCV Python FFMPEG support is enabled
WAS Node Suite Warning: `ffmpeg_bin_path` is not set in `/home/soraka/ComfyUI/custom_nodes/was-node-suite-comfyui/was_suite_config.json` config file. Will attempt to use system ffmpeg binaries if available.
WAS Node Suite: Finished. Loaded 220 nodes successfully.

        "Your time is limited, don't waste it living someone else's life." - Steve Jobs

### Loading: ComfyUI-Manager (V3.31.5)
[ComfyUI-Manager] network_mode: public
### ComfyUI Revision: 3236 [2bc4b596] *DETACHED | Released on '2025-03-09'

Import times for custom nodes:
   0.0 seconds: /home/soraka/ComfyUI/custom_nodes/websocket_image_save.py
   0.0 seconds: /home/soraka/ComfyUI/custom_nodes/comfyui-inpaint-cropandstitch
   0.0 seconds: /home/soraka/ComfyUI/custom_nodes/comfyliterals
   0.0 seconds: /home/soraka/ComfyUI/custom_nodes/ComfyUI-TiledDiffusion
   0.0 seconds: /home/soraka/ComfyUI/custom_nodes/comfyui-depthanythingv2
   0.0 seconds: /home/soraka/ComfyUI/custom_nodes/comfyui-custom-scripts
   0.0 seconds: /home/soraka/ComfyUI/custom_nodes/comfyui-gguf
   0.0 seconds: /home/soraka/ComfyUI/custom_nodes/comfyui_essentials
   0.1 seconds: /home/soraka/ComfyUI/custom_nodes/comfyui-manager
   0.1 seconds: /home/soraka/ComfyUI/custom_nodes/comfyui_ttp_toolset
   0.2 seconds: /home/soraka/ComfyUI/custom_nodes/ComfyUI-Crystools
   0.4 seconds: /home/soraka/ComfyUI/custom_nodes/comfyui-florence2
   0.4 seconds: /home/soraka/ComfyUI/custom_nodes/comfyui-hunyan3dwrapper
   0.7 seconds: /home/soraka/ComfyUI/custom_nodes/was-node-suite-comfyui

Starting server

To see the GUI go to: http://127.0.0.1:8188
[ComfyUI-Manager] default cache updated: https://raw.githubusercontent.com/ltdrdata/ComfyUI-Manager/main/alter-list.json
[ComfyUI-Manager] default cache updated: https://raw.githubusercontent.com/ltdrdata/ComfyUI-Manager/main/model-list.json
[ComfyUI-Manager] default cache updated: https://raw.githubusercontent.com/ltdrdata/ComfyUI-Manager/main/github-stats.json
[ComfyUI-Manager] default cache updated: https://raw.githubusercontent.com/ltdrdata/ComfyUI-Manager/main/extension-node-map.json
[ComfyUI-Manager] default cache updated: https://raw.githubusercontent.com/ltdrdata/ComfyUI-Manager/main/custom-node-list.json
FETCH ComfyRegistry Data: 5/79
FETCH ComfyRegistry Data: 10/79
FETCH ComfyRegistry Data: 15/79
FETCH ComfyRegistry Data: 20/79
FETCH ComfyRegistry Data: 25/79
FETCH ComfyRegistry Data: 30/79
FETCH ComfyRegistry Data: 35/79
FETCH ComfyRegistry Data: 40/79
FETCH ComfyRegistry Data: 45/79
FETCH ComfyRegistry Data: 50/79
FETCH ComfyRegistry Data: 55/79
FETCH ComfyRegistry Data: 60/79
FETCH ComfyRegistry Data: 65/79
FETCH ComfyRegistry Data: 70/79
FETCH ComfyRegistry Data: 75/79
FETCH ComfyRegistry Data [DONE]
[ComfyUI-Manager] default cache updated: https://api.comfy.org/nodes
FETCH DATA from: https://raw.githubusercontent.com/ltdrdata/ComfyUI-Manager/main/custom-node-list.json [DONE]
[ComfyUI-Manager] All startup tasks have been completed.
got prompt
model weight dtype torch.float16, manual cast: None
model_type EPS
Using split attention in VAE
Using split attention in VAE
VAE load device: cuda:0, offload device: cpu, dtype: torch.float32
CLIP/text encoder model load device: cuda:0, offload device: cpu, current: cpu, dtype: torch.float16
Requested to load SDXLClipModel
loaded completely 8351.06171875 1560.802734375 True
Requested to load SDXL
loaded completely 6536.62861328125 4897.0483474731445 True
100%|███████████████████████████████████████████████████████████████████████████████████| 20/20 [00:01<00:00, 12.32it/s]
Requested to load AutoencoderKL
0 models unloaded.
loaded completely 1031.5322265625 319.11416244506836 True
Prompt executed in 7.92 seconds
```


IMPORTANT: It can take up to 10 minutes the first time you queue, and you won't see anything on the command line. It's because it's compiling in the background, let it run. Have faith.

### STEP 5 - Custom Node

Now that you have the manager, installing new custom node can be done from there without the command line. Just reboot, and reload when it's done.

I am not sure how resilient it is. Once installing Sage Attantion bricked all Comfy UI. 

# Conclusions

It is possible to run Comfy UI with great acceleration on amazing price to performance on 7900XTX Windows WSL2 Ubuntu ROCm python 3.10 pytorch

I tested a large number of custom nodes and they work fine.
