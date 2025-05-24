
#### UPDATE 2025-05-17

I rebuilt WSL using python 3.12 under Ubuntu 22, using a portable python environment UV and compartimentalizing all ROCm dependencies there and rebuilt ComfyUI. This way I can backup the full environment when it bricks when trying bleeding edge models.

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

<details>
<summary>WSL creation log</summary>

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

</details>

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

## Setup Comfy UI under Windows WSL

The stack I'm aiming for is as follow:
1) Windows 11
2) Adrenaline driver under windows (25.5.1)
3) HIP SDK under windows ()
4) WSL virtual machine to run Ubuntu 22
5) AMD driver under linux
6) HIP under linux
7) Python portable environment
8) Backup of portable environment
9) Pytorch ROCm binaries 
10) ComfyUI repository
11) ComfyUI pip dependencies
12) ComfyUI custom nodes
13) custom nodes pip dependencies

It looks absurd, and it is. It is also the only way I found to run ComfyUI under windows with wide coverage of pytorch calls.

WSL is needed because under windows there aren't pytorch binaries with wide coverage. You need linux, but I can't be dual booting when I want to run ComfyUI, so WSL to run linux under windows.

Portable python environment is needed because pytorch is brittle, ROCm is brittle, and Comfy UI is brittle. Especially if you play with bleeding edge custom nodes, your environment will brick.

With a portable environment you can backup ALL of it, and restore the backup when it bricks.

### WSL Compatibility Matrix

To see if your GPU is supported, you need to look at a very specific page. As of 2025-06-16 ONLY the 7000 series is supported.

[ROCm WSL compatibility matrix](https://rocm.docs.amd.com/projects/radeon/en/latest/docs/compatibility/wsl/wsl_compatibility.html)

If your GPU isn't in this page, drop following this guide immediately.

## STEP 1 - WSL

[Microsoft WLS Guide](https://learn.microsoft.com/en-us/windows/wsl/install)

Microsoft has a set of instructions to get WSL running, the instructions lack some quality of life improvements that will make the WSL experience useable.

1) install is done by default from microsoft store, that takes up to 8h on my system. Using the ```--web-download option```, lets you download from a repo that has a faster than dial up speed.
2) the partition is an ext4 that is stored on C drive, that is too small on my system. A set of instruction lets you unregister, export and reimport it to a bigger drive.
3) using the default name for WSL will prevent you from doing multiple WSL machines if you need to. The import instruction let you specify a custom name

Open a CMD command prompt

First install Ubuntu22, you will be prompted to select username and password, then shut down the VM

```
wsl --list

wsl --install -d Ubuntu-22.04 --web-download

USERNAME

PASSWORD

sudo shutdown now
```

<details>
<summary>WSL creation log</summary>

```
C:\Users\FatherOfMachines>wsl --list
Windows Subsystem for Linux has no installed distributions.
You can resolve this by installing a distribution with the instructions below:

Use 'wsl.exe --list --online' to list available distributions
and 'wsl.exe --install <Distro>' to install.

C:\Users\FatherOfMachines>wsl --install -d Ubuntu-22.04 --web-download
Ubuntu 22.04 LTS is already installed.
Launching Ubuntu 22.04 LTS...
Installing, this may take a few minutes...
Please create a default UNIX user account. The username does not need to match your Windows username.
For more information visit: https://aka.ms/wslusers
Enter new UNIX username: meridia
New password:
Retype new password:
passwd: password updated successfully
Installation successful!
To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 5.15.167.4-microsoft-standard-WSL2 x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 System information as of Fri May 16 08:59:18 CEST 2025

  System load:  0.31                Processes:             66
  Usage of /:   0.1% of 1006.85GB   Users logged in:       0
  Memory usage: 1%                  IPv4 address for eth0: 172.28.206.241
  Swap usage:   0%


This message is shown once a day. To disable it please create the
/home/meridia/.hushlogin file.
meridia@TowerOfBabel:~$ sudo shutdown now
[sudo] password for meridia:

```
</details><br>

Now move the EXT4 partition to another drive, and rename it.


```
wsl --shutdown

wsl --list

wsl --export Ubuntu-22.04 "F:\WSL-Ubuntu22\WSL-Ubuntu22-ComfyUI.tar"

wsl --unregister Ubuntu-22.04

wsl --import U22-ComfyUI "F:\WSL-Ubuntu22" "F:\WSL-Ubuntu22\WSL-Ubuntu22-ComfyUI.tar"

wsl --list

wsl
```


<details>
<summary>WSL move and rename log</summary>

```

C:\Users\FatherOfMachines>wsl --export Ubuntu-22.04 "F:\WSL-Ubuntu22\WSL-Ubuntu22-ComfyUI.tar"

```
</details><br>

When you log in into the VM, sometimes it will default to the host machine, which is wrong, it will brick if you try to install comfy UI this way.

Always log in with your WSL user, and set yourself in the home directory


Now you have a working WSL virtual machine that has the EXT4 partition in a drive of your chosing, with a custom name. You spin it up by opening a cmd and writing wsl.

```
wsl
su USER
PASSWORD
cd
```

<br>
<details>
<summary>WSL login</summary>

```
C:\Users\FatherOfMachines>wsl
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 5.15.167.4-microsoft-standard-WSL2 x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 System information as of Fri May 16 09:09:24 CEST 2025

  System load:  0.0                 Processes:             69
  Usage of /:   0.1% of 1006.85GB   Users logged in:       0
  Memory usage: 1%                  IPv4 address for eth0: 172.28.206.241
  Swap usage:   0%


This message is shown once a day. To disable it please create the
/root/.hushlogin file.
root@TowerOfBabel:/mnt/c/Users/FatherOfMachines# su meridia
meridia@TowerOfBabel:/mnt/c/Users/FatherOfMachines$ cd
meridia@TowerOfBabel:~$ ls
meridia@TowerOfBabel:~$
```

</details><br>

If everything worked as intended, you now have a WSL Ubuntu 22 virtual machine running under windows.

### WSL File Explorer

There is a bidirectional mounting.
- WSL will see the host machine under /mnt
- Windows will mount WSL in a special Linux tab in file explorer, from which you can move your files, models outputs wheels and more

![Access WSL partition and files](/images/WSL-File-Explorer.png)

### WSL memory configuration

There is an enormous RAM penality to using WSL-ROCm, with Flux I easily use 55GB of RAM. Make sure you have RAM to spare, I would aim to 3X RAM compared to VRAM.

![](/images/WSL-settings.png)

### WSL fast reboot error

If you boot WSL up too quickly after shutdown or reboot, there might be errors

```
<3>WSL (4619 - Relay) ERROR: UtilTranslatePathList:2878: Failed to translate F:\Programs\Git\cmd
<3>WSL (4619 - Relay) ERROR: UtilTranslatePathList:2878: Failed to translate F:\Programs\VSC\bin
<3>WSL (4619 - Relay) ERROR: UtilTranslatePathList:2878: Failed to translate F:\x86_64-13.2.0-release-win32-seh-msvcrt-rt_v11-rev1\bin
```

### WSL HOST Machine folder penality

I tried putting the models on the Host machine via /mnt/f/comfyui-models, but there is a catastrophic performance penality in doing so.

Flux degrades from 60s to 400s, and HiDream degrades even more.

Models need to be inside WSL, I redirected models to $HOME/comfy-ui models so that the models are outside the ComfyUI folder, but inside WSL

```extra_model_paths.yaml```
```
comfyui:
     #Models outside WSL suffer an extreme performance penality in loading
     #base_path: /mnt/f/comfyui-models
     
     #Models inside WSL
     base_path: $HOME/comfyui-models
     # You can use is_default to mark that these folders should be listed first, and used as the default dirs for eg downloads
     #is_default: true
     checkpoints: checkpoints/
     clip: clip/
     clip_vision: clip_vision/
     text_encoders: text_encoders/
     configs: configs/
     controlnet: controlnet/
     diffusion_models: |
                  diffusion_models
                  unet
     embeddings: embeddings/
     loras: loras/
     upscale_models: upscale_models/
     vae: vae/
```

<details>
<summary>Flux - models /mnt - Python 3.10 - 406s</summary>

```
got prompt
Using split attention in VAE
Using split attention in VAE
VAE load device: cuda:0, offload device: cpu, dtype: torch.float32
Requested to load FluxClipModel_
loaded completely 9.5367431640625e+25 9319.23095703125 True
CLIP/text encoder model load device: cuda:0, offload device: cpu, current: cuda:0, dtype: torch.float16
clip missing: ['text_projection.weight']
model weight dtype torch.float8_e4m3fn, manual cast: torch.bfloat16
model_type FLUX
Using split attention in VAE
Using split attention in VAE
VAE load device: cuda:0, offload device: cpu, dtype: torch.float32
CLIP/text encoder model load device: cuda:0, offload device: cpu, current: cpu, dtype: torch.float16
Requested to load Flux
loaded partially 11205.649199218751 11205.642578125 0
100%|███████████████████████████████████████████████████████████████████████████████████| 20/20 [00:40<00:00,  2.05s/it]
Requested to load AutoencodingEngine
0 models unloaded.
loaded completely 3660.523046875 319.7467155456543 True
Prompt executed in 406.13 seconds
```

</details><br>

<details>
<summary>Flux - models $HOME - Python 3.10 - 60s/40s</summary>

```
got prompt
Using split attention in VAE
Using split attention in VAE
VAE load device: cuda:0, offload device: cpu, dtype: torch.float32
Requested to load FluxClipModel_
loaded completely 9.5367431640625e+25 9319.23095703125 True
CLIP/text encoder model load device: cuda:0, offload device: cpu, current: cuda:0, dtype: torch.float16
clip missing: ['text_projection.weight']
model weight dtype torch.float8_e4m3fn, manual cast: torch.bfloat16
model_type FLUX
Using split attention in VAE
Using split attention in VAE
VAE load device: cuda:0, offload device: cpu, dtype: torch.float32
CLIP/text encoder model load device: cuda:0, offload device: cpu, current: cpu, dtype: torch.float16
Requested to load Flux
loaded partially 10567.039824218751 10563.812561035156 0
100%|███████████████████████████████████████████████████████████████████████████████████| 20/20 [00:39<00:00,  2.00s/it]
Requested to load AutoencodingEngine
0 models unloaded.
loaded completely 3658.5234375 319.7467155456543 True
Prompt executed in 60.92 seconds
got prompt
loaded partially 10389.160917968751 10388.998107910156 0
100%|███████████████████████████████████████████████████████████████████████████████████| 20/20 [00:41<00:00,  2.09s/it]
Requested to load AutoencodingEngine
0 models unloaded.
loaded completely 3667.255859375 319.7467155456543 True
Prompt executed in 41.74 seconds
```

</details><br>

<details>
<summary>Flux - models $HOME - Python 3.12 - 60s/40s</summary>

```
got prompt
Using split attention in VAE
Using split attention in VAE
VAE load device: cuda:0, offload device: cpu, dtype: torch.float32
Requested to load FluxClipModel_
loaded completely 9.5367431640625e+25 9319.23095703125 True
CLIP/text encoder model load device: cuda:0, offload device: cpu, current: cuda:0, dtype: torch.float16
clip missing: ['text_projection.weight']
model weight dtype torch.float8_e4m3fn, manual cast: torch.bfloat16
model_type FLUX
Using split attention in VAE
Using split attention in VAE
VAE load device: cuda:0, offload device: cpu, dtype: torch.float32
CLIP/text encoder model load device: cuda:0, offload device: cpu, current: cpu, dtype: torch.float16
Requested to load Flux
loaded partially 10354.188261718751 10354.1142578125 0
100%|███████████████████████████████████████████████████████████████████████████████████| 20/20 [00:42<00:00,  2.10s/it]
Requested to load AutoencodingEngine
0 models unloaded.
loaded completely 3654.15390625 319.7467155456543 True
Prompt executed in 61.74 seconds
got prompt
loaded partially 10809.430449218751 10806.891662597656 0
100%|███████████████████████████████████████████████████████████████████████████████████| 20/20 [00:39<00:00,  1.97s/it]
Requested to load AutoencodingEngine
0 models unloaded.
loaded completely 3664.833203125 319.7467155456543 True
Prompt executed in 40.71 seconds
```

</details><br>


## STEP 2 - GPU and ROCm drivers

This is a critical step, installing the GPU and ROCm drivers. You need to read very cerfully the guides, because there are IF involved, and guides for Ubuntu 24 and Ubuntu 22.

[GPU Linux Driver for WSL](https://rocm.docs.amd.com/projects/radeon/en/latest/docs/install/wsl/install-radeon.html)


Reminder: update will update the location of the packages. upgrade will actualy upgrade the system.

```
sudo apt update

sudo apt upgrade

wget https://repo.radeon.com/amdgpu-install/6.3.4/ubuntu/jammy/amdgpu-install_6.3.60304-1_all.deb

sudo apt install ./amdgpu-install_6.3.60304-1_all.deb

sudo reboot
```

<details>
<summary>GPU install logs</summary>

```
meridia@TowerOfBabel:~$ sudo apt update
[sudo] password for meridia:
Hit:1 http://archive.ubuntu.com/ubuntu jammy InRelease
Get:2 http://archive.ubuntu.com/ubuntu jammy-updates InRelease [128 kB]
Get:3 http://security.ubuntu.com/ubuntu jammy-security InRelease [129 kB]
Get:4 http://archive.ubuntu.com/ubuntu jammy-backports InRelease [127 kB]
Get:5 http://archive.ubuntu.com/ubuntu jammy/universe amd64 Packages [14.1 MB]
Get:6 http://security.ubuntu.com/ubuntu jammy-security/main amd64 Packages [2338 kB]
Get:7 http://security.ubuntu.com/ubuntu jammy-security/main Translation-en [354 kB]
Get:8 http://archive.ubuntu.com/ubuntu jammy/universe Translation-en [5652 kB]
Get:9 http://security.ubuntu.com/ubuntu jammy-security/main amd64 c-n-f Metadata [13.6 kB]
Get:10 http://security.ubuntu.com/ubuntu jammy-security/restricted amd64 Packages [3429 kB]
Get:11 http://archive.ubuntu.com/ubuntu jammy/universe amd64 c-n-f Metadata [286 kB]
Get:12 http://archive.ubuntu.com/ubuntu jammy/multiverse amd64 Packages [217 kB]
Get:13 http://archive.ubuntu.com/ubuntu jammy/multiverse Translation-en [112 kB]
Get:14 http://archive.ubuntu.com/ubuntu jammy/multiverse amd64 c-n-f Metadata [8372 B]
Get:15 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 Packages [2583 kB]
Get:16 http://security.ubuntu.com/ubuntu jammy-security/restricted Translation-en [613 kB]
Get:17 http://security.ubuntu.com/ubuntu jammy-security/restricted amd64 c-n-f Metadata [624 B]
Get:18 http://security.ubuntu.com/ubuntu jammy-security/universe amd64 Packages [974 kB]
Get:19 http://security.ubuntu.com/ubuntu jammy-security/universe Translation-en [210 kB]
Get:20 http://security.ubuntu.com/ubuntu jammy-security/universe amd64 c-n-f Metadata [21.7 kB]
Get:21 http://security.ubuntu.com/ubuntu jammy-security/multiverse amd64 Packages [39.6 kB]
Get:22 http://security.ubuntu.com/ubuntu jammy-security/multiverse Translation-en [8716 B]
Get:23 http://security.ubuntu.com/ubuntu jammy-security/multiverse amd64 c-n-f Metadata [368 B]
Get:24 http://archive.ubuntu.com/ubuntu jammy-updates/main Translation-en [418 kB]
Get:25 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 c-n-f Metadata [18.5 kB]
Get:26 http://archive.ubuntu.com/ubuntu jammy-updates/restricted amd64 Packages [3549 kB]
Get:27 http://archive.ubuntu.com/ubuntu jammy-updates/restricted Translation-en [631 kB]
Get:28 http://archive.ubuntu.com/ubuntu jammy-updates/restricted amd64 c-n-f Metadata [676 B]
Get:29 http://archive.ubuntu.com/ubuntu jammy-updates/universe amd64 Packages [1203 kB]
Get:30 http://archive.ubuntu.com/ubuntu jammy-updates/universe Translation-en [297 kB]
Get:31 http://archive.ubuntu.com/ubuntu jammy-updates/universe amd64 c-n-f Metadata [28.7 kB]
Get:32 http://archive.ubuntu.com/ubuntu jammy-updates/multiverse amd64 Packages [46.5 kB]
Get:33 http://archive.ubuntu.com/ubuntu jammy-updates/multiverse Translation-en [11.8 kB]
Get:34 http://archive.ubuntu.com/ubuntu jammy-updates/multiverse amd64 c-n-f Metadata [592 B]
Get:35 http://archive.ubuntu.com/ubuntu jammy-backports/main amd64 Packages [68.8 kB]
Get:36 http://archive.ubuntu.com/ubuntu jammy-backports/main Translation-en [11.4 kB]
Get:37 http://archive.ubuntu.com/ubuntu jammy-backports/main amd64 c-n-f Metadata [392 B]
Get:38 http://archive.ubuntu.com/ubuntu jammy-backports/restricted amd64 c-n-f Metadata [116 B]
Get:39 http://archive.ubuntu.com/ubuntu jammy-backports/universe amd64 Packages [30.0 kB]
Get:40 http://archive.ubuntu.com/ubuntu jammy-backports/universe Translation-en [16.5 kB]
Get:41 http://archive.ubuntu.com/ubuntu jammy-backports/universe amd64 c-n-f Metadata [672 B]
Get:42 http://archive.ubuntu.com/ubuntu jammy-backports/multiverse amd64 c-n-f Metadata [116 B]
Fetched 37.7 MB in 5s (7454 kB/s)
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
87 packages can be upgraded. Run 'apt list --upgradable' to see them.

meridia@TowerOfBabel:~$ sudo apt upgrade
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
Calculating upgrade... Done
The following packages will be upgraded:
  bind9-dnsutils bind9-host bind9-libs binutils binutils-common binutils-x86-64-linux-gnu cloud-init dirmngr
  distro-info-data dmsetup git git-man gnupg gnupg-l10n gnupg-utils gpg gpg-agent gpg-wks-client gpg-wks-server
  gpgconf gpgsm gpgv landscape-client landscape-common libbinutils libc-bin libc6 libcap2 libcap2-bin libcryptsetup12
  libctf-nobfd0 libctf0 libdevmapper1.02.1 libdw1 libelf1 libexpat1 libfreetype6 libgnutls30 libgssapi-krb5-2
  libharfbuzz0b libk5crypto3 libkrb5-3 libkrb5support0 libldap-2.5-0 libldap-common libnss-systemd libpam-cap
  libpam-modules libpam-modules-bin libpam-runtime libpam-systemd libpam0g libperl5.34 libpython3.10
  libpython3.10-minimal libpython3.10-stdlib libseccomp2 libssl3 libsystemd0 libtasn1-6 libudev1 libxml2 locales
  openssh-client openssl pci.ids perl perl-base perl-modules-5.34 python3-jinja2 python3.10 python3.10-minimal rsync
  snapd systemd systemd-sysv systemd-timesyncd tzdata ubuntu-advantage-tools ubuntu-pro-client ubuntu-pro-client-l10n
  udev vim vim-common vim-runtime vim-tiny xxd
87 upgraded, 0 newly installed, 0 to remove and 0 not upgraded.
60 standard LTS security updates
Need to get 91.6 MB of archives.
After this operation, 635 kB of additional disk space will be used.
Do you want to continue? [Y/n] y
Get:1 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libperl5.34 amd64 5.34.0-3ubuntu1.4 [4820 kB]
Get:2 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 perl amd64 5.34.0-3ubuntu1.4 [232 kB]
Get:3 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 perl-base amd64 5.34.0-3ubuntu1.4 [1759 kB]
Get:4 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 perl-modules-5.34 all 5.34.0-3ubuntu1.4 [2977 kB]
Get:5 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libc6 amd64 2.35-0ubuntu3.9 [3235 kB]
Get:6 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libc-bin amd64 2.35-0ubuntu3.9 [706 kB]
Get:7 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libpam0g amd64 1.4.0-11ubuntu2.5 [59.8 kB]
Get:8 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libpam-modules-bin amd64 1.4.0-11ubuntu2.5 [37.4 kB]
Get:9 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libpam-modules amd64 1.4.0-11ubuntu2.5 [280 kB]
Get:10 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libnss-systemd amd64 249.11-0ubuntu3.15 [133 kB]
Get:11 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libsystemd0 amd64 249.11-0ubuntu3.15 [317 kB]
Get:12 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 systemd-timesyncd amd64 249.11-0ubuntu3.15 [31.2 kB]
Get:13 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 systemd-sysv amd64 249.11-0ubuntu3.15 [10.5 kB]
Get:14 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libpam-systemd amd64 249.11-0ubuntu3.15 [203 kB]
Get:15 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 systemd amd64 249.11-0ubuntu3.15 [4581 kB]
Get:16 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 udev amd64 249.11-0ubuntu3.15 [1557 kB]
Get:17 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libudev1 amd64 249.11-0ubuntu3.15 [76.6 kB]
Get:18 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libcap2 amd64 1:2.44-1ubuntu0.22.04.2 [18.3 kB]
Get:19 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libpam-runtime all 1.4.0-11ubuntu2.5 [40.2 kB]
Get:20 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libdevmapper1.02.1 amd64 2:1.02.175-2.1ubuntu5 [139 kB]
Get:21 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libssl3 amd64 3.0.2-0ubuntu1.19 [1905 kB]
Get:22 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libcryptsetup12 amd64 2:2.4.3-1ubuntu1.3 [211 kB]
Get:23 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libtasn1-6 amd64 4.18.0-4ubuntu0.1 [43.5 kB]
Get:24 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libgnutls30 amd64 3.7.3-4ubuntu1.6 [969 kB]
Get:25 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libseccomp2 amd64 2.5.3-2ubuntu3~22.04.1 [47.4 kB]
Get:26 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libexpat1 amd64 2.4.7-1ubuntu0.6 [92.1 kB]
Get:27 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libpython3.10 amd64 3.10.12-1~22.04.9 [1949 kB]
Get:28 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 python3.10 amd64 3.10.12-1~22.04.9 [508 kB]
Get:29 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libpython3.10-stdlib amd64 3.10.12-1~22.04.9 [1850 kB]
Get:30 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 python3.10-minimal amd64 3.10.12-1~22.04.9 [2263 kB]
Get:31 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libpython3.10-minimal amd64 3.10.12-1~22.04.9 [815 kB]
Get:32 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 rsync amd64 3.2.7-0ubuntu0.22.04.4 [437 kB]
Get:33 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libk5crypto3 amd64 1.19.2-2ubuntu0.6 [86.5 kB]
Get:34 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libkrb5support0 amd64 1.19.2-2ubuntu0.6 [32.5 kB]
Get:35 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libkrb5-3 amd64 1.19.2-2ubuntu0.6 [357 kB]
Get:36 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libgssapi-krb5-2 amd64 1.19.2-2ubuntu0.6 [145 kB]
Get:37 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 gpg-wks-client amd64 2.2.27-3ubuntu2.3 [62.7 kB]
Get:38 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 dirmngr amd64 2.2.27-3ubuntu2.3 [293 kB]
Get:39 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 gpg-wks-server amd64 2.2.27-3ubuntu2.3 [57.6 kB]
Get:40 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 gnupg-utils amd64 2.2.27-3ubuntu2.3 [309 kB]
Get:41 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 gpg-agent amd64 2.2.27-3ubuntu2.3 [209 kB]
Get:42 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 gpg amd64 2.2.27-3ubuntu2.3 [519 kB]
Get:43 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 gpgconf amd64 2.2.27-3ubuntu2.3 [94.4 kB]
Get:44 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 gnupg-l10n all 2.2.27-3ubuntu2.3 [54.6 kB]
Get:45 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 gnupg all 2.2.27-3ubuntu2.3 [315 kB]
Get:46 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 gpgsm amd64 2.2.27-3ubuntu2.3 [198 kB]
Get:47 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libldap-2.5-0 amd64 2.5.19+dfsg-0ubuntu0.22.04.1 [184 kB]
Get:48 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 gpgv amd64 2.2.27-3ubuntu2.3 [137 kB]
Get:49 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 distro-info-data all 0.52ubuntu0.9 [5336 B]
Get:50 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 dmsetup amd64 2:1.02.175-2.1ubuntu5 [81.7 kB]
Get:51 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libpam-cap amd64 1:2.44-1ubuntu0.22.04.2 [7930 B]
Get:52 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libcap2-bin amd64 1:2.44-1ubuntu0.22.04.2 [26.0 kB]
Get:53 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libdw1 amd64 0.186-1ubuntu0.1 [251 kB]
Get:54 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libelf1 amd64 0.186-1ubuntu0.1 [51.1 kB]
Get:55 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libxml2 amd64 2.9.13+dfsg-1ubuntu0.7 [763 kB]
Get:56 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 locales all 2.35-0ubuntu3.9 [4248 kB]
Get:57 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 openssl amd64 3.0.2-0ubuntu1.19 [1186 kB]
Get:58 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 tzdata all 2025b-0ubuntu0.22.04 [347 kB]
Get:59 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 ubuntu-pro-client-l10n amd64 35.1ubuntu0~22.04 [20.6 kB]
Get:60 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 ubuntu-pro-client amd64 35.1ubuntu0~22.04 [236 kB]
Get:61 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 ubuntu-advantage-tools all 35.1ubuntu0~22.04 [10.9 kB]
Get:62 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 vim amd64 2:8.2.3995-1ubuntu2.24 [1728 kB]
Get:63 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 vim-tiny amd64 2:8.2.3995-1ubuntu2.24 [707 kB]
Get:64 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 vim-runtime all 2:8.2.3995-1ubuntu2.24 [6833 kB]
Get:65 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 xxd amd64 2:8.2.3995-1ubuntu2.24 [51.4 kB]
Get:66 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 vim-common all 2:8.2.3995-1ubuntu2.24 [81.5 kB]
Get:67 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 bind9-dnsutils amd64 1:9.18.30-0ubuntu0.22.04.2 [158 kB]
Get:68 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 bind9-host amd64 1:9.18.30-0ubuntu0.22.04.2 [52.6 kB]
Get:69 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 bind9-libs amd64 1:9.18.30-0ubuntu0.22.04.2 [1259 kB]
Get:70 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 git-man all 1:2.34.1-1ubuntu1.12 [955 kB]
Get:71 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 git amd64 1:2.34.1-1ubuntu1.12 [3165 kB]
Get:72 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 openssh-client amd64 1:8.9p1-3ubuntu0.13 [903 kB]
Get:73 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 pci.ids all 0.0~2022.01.22-1ubuntu0.1 [251 kB]
Get:74 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libctf0 amd64 2.38-4ubuntu2.8 [103 kB]
Get:75 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libctf-nobfd0 amd64 2.38-4ubuntu2.8 [108 kB]
Get:76 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 binutils-x86-64-linux-gnu amd64 2.38-4ubuntu2.8 [2324 kB]
Get:77 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libbinutils amd64 2.38-4ubuntu2.8 [661 kB]
Get:78 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 binutils amd64 2.38-4ubuntu2.8 [3196 B]
Get:79 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 binutils-common amd64 2.38-4ubuntu2.8 [223 kB]
Get:80 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 landscape-client amd64 23.02-0ubuntu1~22.04.4 [113 kB]
Get:81 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 landscape-common amd64 23.02-0ubuntu1~22.04.4 [88.8 kB]
Get:82 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libfreetype6 amd64 2.11.1+dfsg-1ubuntu0.3 [388 kB]
Get:83 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libharfbuzz0b amd64 2.7.4-1ubuntu3.2 [353 kB]
Get:84 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libldap-common all 2.5.19+dfsg-0ubuntu0.22.04.1 [16.1 kB]
Get:85 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 python3-jinja2 all 3.0.3-1ubuntu0.4 [108 kB]
Get:86 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 snapd amd64 2.67.1+22.04 [27.8 MB]
Get:87 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 cloud-init all 24.4.1-0ubuntu0~22.04.2 [566 kB]
Fetched 91.6 MB in 12s (7596 kB/s)
Extracting templates from packages: 100%
Preconfiguring packages ...
(Reading database ... 42752 files and directories currently installed.)
Preparing to unpack .../libperl5.34_5.34.0-3ubuntu1.4_amd64.deb ...
Unpacking libperl5.34:amd64 (5.34.0-3ubuntu1.4) over (5.34.0-3ubuntu1.3) ...
Preparing to unpack .../perl_5.34.0-3ubuntu1.4_amd64.deb ...
Unpacking perl (5.34.0-3ubuntu1.4) over (5.34.0-3ubuntu1.3) ...
Preparing to unpack .../perl-base_5.34.0-3ubuntu1.4_amd64.deb ...
Unpacking perl-base (5.34.0-3ubuntu1.4) over (5.34.0-3ubuntu1.3) ...
Setting up perl-base (5.34.0-3ubuntu1.4) ...
(Reading database ... 42752 files and directories currently installed.)
Preparing to unpack .../perl-modules-5.34_5.34.0-3ubuntu1.4_all.deb ...
Unpacking perl-modules-5.34 (5.34.0-3ubuntu1.4) over (5.34.0-3ubuntu1.3) ...
Preparing to unpack .../libc6_2.35-0ubuntu3.9_amd64.deb ...
Unpacking libc6:amd64 (2.35-0ubuntu3.9) over (2.35-0ubuntu3.8) ...
Setting up libc6:amd64 (2.35-0ubuntu3.9) ...
(Reading database ... 42752 files and directories currently installed.)
Preparing to unpack .../libc-bin_2.35-0ubuntu3.9_amd64.deb ...
Unpacking libc-bin (2.35-0ubuntu3.9) over (2.35-0ubuntu3.8) ...
Setting up libc-bin (2.35-0ubuntu3.9) ...
(Reading database ... 42752 files and directories currently installed.)
Preparing to unpack .../libpam0g_1.4.0-11ubuntu2.5_amd64.deb ...
Unpacking libpam0g:amd64 (1.4.0-11ubuntu2.5) over (1.4.0-11ubuntu2.4) ...
Setting up libpam0g:amd64 (1.4.0-11ubuntu2.5) ...
(Reading database ... 42752 files and directories currently installed.)
Preparing to unpack .../libpam-modules-bin_1.4.0-11ubuntu2.5_amd64.deb ...
Unpacking libpam-modules-bin (1.4.0-11ubuntu2.5) over (1.4.0-11ubuntu2.4) ...
Setting up libpam-modules-bin (1.4.0-11ubuntu2.5) ...
(Reading database ... 42752 files and directories currently installed.)
Preparing to unpack .../libpam-modules_1.4.0-11ubuntu2.5_amd64.deb ...
Unpacking libpam-modules:amd64 (1.4.0-11ubuntu2.5) over (1.4.0-11ubuntu2.4) ...
Setting up libpam-modules:amd64 (1.4.0-11ubuntu2.5) ...
(Reading database ... 42752 files and directories currently installed.)
Preparing to unpack .../libnss-systemd_249.11-0ubuntu3.15_amd64.deb ...
Unpacking libnss-systemd:amd64 (249.11-0ubuntu3.15) over (249.11-0ubuntu3.12) ...
Preparing to unpack .../libsystemd0_249.11-0ubuntu3.15_amd64.deb ...
Unpacking libsystemd0:amd64 (249.11-0ubuntu3.15) over (249.11-0ubuntu3.12) ...
Setting up libsystemd0:amd64 (249.11-0ubuntu3.15) ...
(Reading database ... 42752 files and directories currently installed.)
Preparing to unpack .../0-systemd-timesyncd_249.11-0ubuntu3.15_amd64.deb ...
Unpacking systemd-timesyncd (249.11-0ubuntu3.15) over (249.11-0ubuntu3.12) ...
Preparing to unpack .../1-systemd-sysv_249.11-0ubuntu3.15_amd64.deb ...
Unpacking systemd-sysv (249.11-0ubuntu3.15) over (249.11-0ubuntu3.12) ...
Preparing to unpack .../2-libpam-systemd_249.11-0ubuntu3.15_amd64.deb ...
Unpacking libpam-systemd:amd64 (249.11-0ubuntu3.15) over (249.11-0ubuntu3.12) ...
Preparing to unpack .../3-systemd_249.11-0ubuntu3.15_amd64.deb ...
Unpacking systemd (249.11-0ubuntu3.15) over (249.11-0ubuntu3.12) ...
Preparing to unpack .../4-udev_249.11-0ubuntu3.15_amd64.deb ...
Unpacking udev (249.11-0ubuntu3.15) over (249.11-0ubuntu3.12) ...
Preparing to unpack .../5-libudev1_249.11-0ubuntu3.15_amd64.deb ...
Unpacking libudev1:amd64 (249.11-0ubuntu3.15) over (249.11-0ubuntu3.12) ...
Setting up libudev1:amd64 (249.11-0ubuntu3.15) ...
(Reading database ... 42752 files and directories currently installed.)
Preparing to unpack .../libcap2_1%3a2.44-1ubuntu0.22.04.2_amd64.deb ...
Unpacking libcap2:amd64 (1:2.44-1ubuntu0.22.04.2) over (1:2.44-1ubuntu0.22.04.1) ...
Setting up libcap2:amd64 (1:2.44-1ubuntu0.22.04.2) ...
(Reading database ... 42752 files and directories currently installed.)
Preparing to unpack .../libpam-runtime_1.4.0-11ubuntu2.5_all.deb ...
Unpacking libpam-runtime (1.4.0-11ubuntu2.5) over (1.4.0-11ubuntu2.4) ...
Setting up libpam-runtime (1.4.0-11ubuntu2.5) ...
(Reading database ... 42752 files and directories currently installed.)
Preparing to unpack .../libdevmapper1.02.1_2%3a1.02.175-2.1ubuntu5_amd64.deb ...
Unpacking libdevmapper1.02.1:amd64 (2:1.02.175-2.1ubuntu5) over (2:1.02.175-2.1ubuntu4) ...
Preparing to unpack .../libssl3_3.0.2-0ubuntu1.19_amd64.deb ...
Unpacking libssl3:amd64 (3.0.2-0ubuntu1.19) over (3.0.2-0ubuntu1.18) ...
Setting up libssl3:amd64 (3.0.2-0ubuntu1.19) ...
(Reading database ... 42752 files and directories currently installed.)
Preparing to unpack .../libcryptsetup12_2%3a2.4.3-1ubuntu1.3_amd64.deb ...
Unpacking libcryptsetup12:amd64 (2:2.4.3-1ubuntu1.3) over (2:2.4.3-1ubuntu1.2) ...
Preparing to unpack .../libtasn1-6_4.18.0-4ubuntu0.1_amd64.deb ...
Unpacking libtasn1-6:amd64 (4.18.0-4ubuntu0.1) over (4.18.0-4build1) ...
Setting up libtasn1-6:amd64 (4.18.0-4ubuntu0.1) ...
(Reading database ... 42752 files and directories currently installed.)
Preparing to unpack .../libgnutls30_3.7.3-4ubuntu1.6_amd64.deb ...
Unpacking libgnutls30:amd64 (3.7.3-4ubuntu1.6) over (3.7.3-4ubuntu1.5) ...
Setting up libgnutls30:amd64 (3.7.3-4ubuntu1.6) ...
(Reading database ... 42752 files and directories currently installed.)
Preparing to unpack .../libseccomp2_2.5.3-2ubuntu3~22.04.1_amd64.deb ...
Unpacking libseccomp2:amd64 (2.5.3-2ubuntu3~22.04.1) over (2.5.3-2ubuntu2) ...
Setting up libseccomp2:amd64 (2.5.3-2ubuntu3~22.04.1) ...
(Reading database ... 42752 files and directories currently installed.)
Preparing to unpack .../0-libexpat1_2.4.7-1ubuntu0.6_amd64.deb ...
Unpacking libexpat1:amd64 (2.4.7-1ubuntu0.6) over (2.4.7-1ubuntu0.5) ...
Preparing to unpack .../1-libpython3.10_3.10.12-1~22.04.9_amd64.deb ...
Unpacking libpython3.10:amd64 (3.10.12-1~22.04.9) over (3.10.12-1~22.04.7) ...
Preparing to unpack .../2-python3.10_3.10.12-1~22.04.9_amd64.deb ...
Unpacking python3.10 (3.10.12-1~22.04.9) over (3.10.12-1~22.04.7) ...
Preparing to unpack .../3-libpython3.10-stdlib_3.10.12-1~22.04.9_amd64.deb ...
Unpacking libpython3.10-stdlib:amd64 (3.10.12-1~22.04.9) over (3.10.12-1~22.04.7) ...
Preparing to unpack .../4-python3.10-minimal_3.10.12-1~22.04.9_amd64.deb ...
Unpacking python3.10-minimal (3.10.12-1~22.04.9) over (3.10.12-1~22.04.7) ...
Preparing to unpack .../5-libpython3.10-minimal_3.10.12-1~22.04.9_amd64.deb ...
Unpacking libpython3.10-minimal:amd64 (3.10.12-1~22.04.9) over (3.10.12-1~22.04.7) ...
Preparing to unpack .../6-rsync_3.2.7-0ubuntu0.22.04.4_amd64.deb ...
Unpacking rsync (3.2.7-0ubuntu0.22.04.4) over (3.2.7-0ubuntu0.22.04.2) ...
Preparing to unpack .../7-libk5crypto3_1.19.2-2ubuntu0.6_amd64.deb ...
Unpacking libk5crypto3:amd64 (1.19.2-2ubuntu0.6) over (1.19.2-2ubuntu0.4) ...
Setting up libk5crypto3:amd64 (1.19.2-2ubuntu0.6) ...
(Reading database ... 42752 files and directories currently installed.)
Preparing to unpack .../libkrb5support0_1.19.2-2ubuntu0.6_amd64.deb ...
Unpacking libkrb5support0:amd64 (1.19.2-2ubuntu0.6) over (1.19.2-2ubuntu0.4) ...
Setting up libkrb5support0:amd64 (1.19.2-2ubuntu0.6) ...
(Reading database ... 42752 files and directories currently installed.)
Preparing to unpack .../libkrb5-3_1.19.2-2ubuntu0.6_amd64.deb ...
Unpacking libkrb5-3:amd64 (1.19.2-2ubuntu0.6) over (1.19.2-2ubuntu0.4) ...
Setting up libkrb5-3:amd64 (1.19.2-2ubuntu0.6) ...
(Reading database ... 42752 files and directories currently installed.)
Preparing to unpack .../libgssapi-krb5-2_1.19.2-2ubuntu0.6_amd64.deb ...
Unpacking libgssapi-krb5-2:amd64 (1.19.2-2ubuntu0.6) over (1.19.2-2ubuntu0.4) ...
Setting up libgssapi-krb5-2:amd64 (1.19.2-2ubuntu0.6) ...
(Reading database ... 42752 files and directories currently installed.)
Preparing to unpack .../00-gpg-wks-client_2.2.27-3ubuntu2.3_amd64.deb ...
Unpacking gpg-wks-client (2.2.27-3ubuntu2.3) over (2.2.27-3ubuntu2.1) ...
Preparing to unpack .../01-dirmngr_2.2.27-3ubuntu2.3_amd64.deb ...
Unpacking dirmngr (2.2.27-3ubuntu2.3) over (2.2.27-3ubuntu2.1) ...
Preparing to unpack .../02-gpg-wks-server_2.2.27-3ubuntu2.3_amd64.deb ...
Unpacking gpg-wks-server (2.2.27-3ubuntu2.3) over (2.2.27-3ubuntu2.1) ...
Preparing to unpack .../03-gnupg-utils_2.2.27-3ubuntu2.3_amd64.deb ...
Unpacking gnupg-utils (2.2.27-3ubuntu2.3) over (2.2.27-3ubuntu2.1) ...
Preparing to unpack .../04-gpg-agent_2.2.27-3ubuntu2.3_amd64.deb ...
Unpacking gpg-agent (2.2.27-3ubuntu2.3) over (2.2.27-3ubuntu2.1) ...
Preparing to unpack .../05-gpg_2.2.27-3ubuntu2.3_amd64.deb ...
Unpacking gpg (2.2.27-3ubuntu2.3) over (2.2.27-3ubuntu2.1) ...
Preparing to unpack .../06-gpgconf_2.2.27-3ubuntu2.3_amd64.deb ...
Unpacking gpgconf (2.2.27-3ubuntu2.3) over (2.2.27-3ubuntu2.1) ...
Preparing to unpack .../07-gnupg-l10n_2.2.27-3ubuntu2.3_all.deb ...
Unpacking gnupg-l10n (2.2.27-3ubuntu2.3) over (2.2.27-3ubuntu2.1) ...
Preparing to unpack .../08-gnupg_2.2.27-3ubuntu2.3_all.deb ...
Unpacking gnupg (2.2.27-3ubuntu2.3) over (2.2.27-3ubuntu2.1) ...
Preparing to unpack .../09-gpgsm_2.2.27-3ubuntu2.3_amd64.deb ...
Unpacking gpgsm (2.2.27-3ubuntu2.3) over (2.2.27-3ubuntu2.1) ...
Preparing to unpack .../10-libldap-2.5-0_2.5.19+dfsg-0ubuntu0.22.04.1_amd64.deb ...
Unpacking libldap-2.5-0:amd64 (2.5.19+dfsg-0ubuntu0.22.04.1) over (2.5.18+dfsg-0ubuntu0.22.04.2) ...
Preparing to unpack .../11-gpgv_2.2.27-3ubuntu2.3_amd64.deb ...
Unpacking gpgv (2.2.27-3ubuntu2.3) over (2.2.27-3ubuntu2.1) ...
Setting up gpgv (2.2.27-3ubuntu2.3) ...
(Reading database ... 42752 files and directories currently installed.)
Preparing to unpack .../00-distro-info-data_0.52ubuntu0.9_all.deb ...
Unpacking distro-info-data (0.52ubuntu0.9) over (0.52ubuntu0.8) ...
Preparing to unpack .../01-dmsetup_2%3a1.02.175-2.1ubuntu5_amd64.deb ...
Unpacking dmsetup (2:1.02.175-2.1ubuntu5) over (2:1.02.175-2.1ubuntu4) ...
Preparing to unpack .../02-libpam-cap_1%3a2.44-1ubuntu0.22.04.2_amd64.deb ...
Unpacking libpam-cap:amd64 (1:2.44-1ubuntu0.22.04.2) over (1:2.44-1ubuntu0.22.04.1) ...
Preparing to unpack .../03-libcap2-bin_1%3a2.44-1ubuntu0.22.04.2_amd64.deb ...
Unpacking libcap2-bin (1:2.44-1ubuntu0.22.04.2) over (1:2.44-1ubuntu0.22.04.1) ...
Preparing to unpack .../04-libdw1_0.186-1ubuntu0.1_amd64.deb ...
Unpacking libdw1:amd64 (0.186-1ubuntu0.1) over (0.186-1build1) ...
Preparing to unpack .../05-libelf1_0.186-1ubuntu0.1_amd64.deb ...
Unpacking libelf1:amd64 (0.186-1ubuntu0.1) over (0.186-1build1) ...
Preparing to unpack .../06-libxml2_2.9.13+dfsg-1ubuntu0.7_amd64.deb ...
Unpacking libxml2:amd64 (2.9.13+dfsg-1ubuntu0.7) over (2.9.13+dfsg-1ubuntu0.4) ...
Preparing to unpack .../07-locales_2.35-0ubuntu3.9_all.deb ...
Unpacking locales (2.35-0ubuntu3.9) over (2.35-0ubuntu3.8) ...
Preparing to unpack .../08-openssl_3.0.2-0ubuntu1.19_amd64.deb ...
Unpacking openssl (3.0.2-0ubuntu1.19) over (3.0.2-0ubuntu1.18) ...
Preparing to unpack .../09-tzdata_2025b-0ubuntu0.22.04_all.deb ...
Unpacking tzdata (2025b-0ubuntu0.22.04) over (2024a-0ubuntu0.22.04.1) ...
Preparing to unpack .../10-ubuntu-pro-client-l10n_35.1ubuntu0~22.04_amd64.deb ...
Unpacking ubuntu-pro-client-l10n (35.1ubuntu0~22.04) over (34~22.04) ...
Preparing to unpack .../11-ubuntu-pro-client_35.1ubuntu0~22.04_amd64.deb ...
Unpacking ubuntu-pro-client (35.1ubuntu0~22.04) over (34~22.04) ...
Preparing to unpack .../12-ubuntu-advantage-tools_35.1ubuntu0~22.04_all.deb ...
Unpacking ubuntu-advantage-tools (35.1ubuntu0~22.04) over (34~22.04) ...
Preparing to unpack .../13-vim_2%3a8.2.3995-1ubuntu2.24_amd64.deb ...
Unpacking vim (2:8.2.3995-1ubuntu2.24) over (2:8.2.3995-1ubuntu2.21) ...
Preparing to unpack .../14-vim-tiny_2%3a8.2.3995-1ubuntu2.24_amd64.deb ...
Unpacking vim-tiny (2:8.2.3995-1ubuntu2.24) over (2:8.2.3995-1ubuntu2.21) ...
Preparing to unpack .../15-vim-runtime_2%3a8.2.3995-1ubuntu2.24_all.deb ...
Unpacking vim-runtime (2:8.2.3995-1ubuntu2.24) over (2:8.2.3995-1ubuntu2.21) ...
Preparing to unpack .../16-xxd_2%3a8.2.3995-1ubuntu2.24_amd64.deb ...
Unpacking xxd (2:8.2.3995-1ubuntu2.24) over (2:8.2.3995-1ubuntu2.21) ...
Preparing to unpack .../17-vim-common_2%3a8.2.3995-1ubuntu2.24_all.deb ...
Unpacking vim-common (2:8.2.3995-1ubuntu2.24) over (2:8.2.3995-1ubuntu2.21) ...
Preparing to unpack .../18-bind9-dnsutils_1%3a9.18.30-0ubuntu0.22.04.2_amd64.deb ...
Unpacking bind9-dnsutils (1:9.18.30-0ubuntu0.22.04.2) over (1:9.18.28-0ubuntu0.22.04.1) ...
Preparing to unpack .../19-bind9-host_1%3a9.18.30-0ubuntu0.22.04.2_amd64.deb ...
Unpacking bind9-host (1:9.18.30-0ubuntu0.22.04.2) over (1:9.18.28-0ubuntu0.22.04.1) ...
Preparing to unpack .../20-bind9-libs_1%3a9.18.30-0ubuntu0.22.04.2_amd64.deb ...
Unpacking bind9-libs:amd64 (1:9.18.30-0ubuntu0.22.04.2) over (1:9.18.28-0ubuntu0.22.04.1) ...
Preparing to unpack .../21-git-man_1%3a2.34.1-1ubuntu1.12_all.deb ...
Unpacking git-man (1:2.34.1-1ubuntu1.12) over (1:2.34.1-1ubuntu1.11) ...
Preparing to unpack .../22-git_1%3a2.34.1-1ubuntu1.12_amd64.deb ...
Unpacking git (1:2.34.1-1ubuntu1.12) over (1:2.34.1-1ubuntu1.11) ...
Preparing to unpack .../23-openssh-client_1%3a8.9p1-3ubuntu0.13_amd64.deb ...
Unpacking openssh-client (1:8.9p1-3ubuntu0.13) over (1:8.9p1-3ubuntu0.10) ...
Preparing to unpack .../24-pci.ids_0.0~2022.01.22-1ubuntu0.1_all.deb ...
Unpacking pci.ids (0.0~2022.01.22-1ubuntu0.1) over (0.0~2022.01.22-1) ...
Preparing to unpack .../25-libctf0_2.38-4ubuntu2.8_amd64.deb ...
Unpacking libctf0:amd64 (2.38-4ubuntu2.8) over (2.38-4ubuntu2.6) ...
Preparing to unpack .../26-libctf-nobfd0_2.38-4ubuntu2.8_amd64.deb ...
Unpacking libctf-nobfd0:amd64 (2.38-4ubuntu2.8) over (2.38-4ubuntu2.6) ...
Preparing to unpack .../27-binutils-x86-64-linux-gnu_2.38-4ubuntu2.8_amd64.deb ...
Unpacking binutils-x86-64-linux-gnu (2.38-4ubuntu2.8) over (2.38-4ubuntu2.6) ...
Preparing to unpack .../28-libbinutils_2.38-4ubuntu2.8_amd64.deb ...
Unpacking libbinutils:amd64 (2.38-4ubuntu2.8) over (2.38-4ubuntu2.6) ...
Preparing to unpack .../29-binutils_2.38-4ubuntu2.8_amd64.deb ...
Unpacking binutils (2.38-4ubuntu2.8) over (2.38-4ubuntu2.6) ...
Preparing to unpack .../30-binutils-common_2.38-4ubuntu2.8_amd64.deb ...
Unpacking binutils-common:amd64 (2.38-4ubuntu2.8) over (2.38-4ubuntu2.6) ...
Preparing to unpack .../31-landscape-client_23.02-0ubuntu1~22.04.4_amd64.deb ...
Unpacking landscape-client (23.02-0ubuntu1~22.04.4) over (23.02-0ubuntu1~22.04.3) ...
Preparing to unpack .../32-landscape-common_23.02-0ubuntu1~22.04.4_amd64.deb ...
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 5.15.167.4-microsoft-standard-WSL2 x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

Unpacking landscape-common (23.02-0ubuntu1~22.04.4) over (23.02-0ubuntu1~22.04.3) ...
Preparing to unpack .../33-libfreetype6_2.11.1+dfsg-1ubuntu0.3_amd64.deb ...
Unpacking libfreetype6:amd64 (2.11.1+dfsg-1ubuntu0.3) over (2.11.1+dfsg-1ubuntu0.2) ...
Preparing to unpack .../34-libharfbuzz0b_2.7.4-1ubuntu3.2_amd64.deb ...
Unpacking libharfbuzz0b:amd64 (2.7.4-1ubuntu3.2) over (2.7.4-1ubuntu3.1) ...
Preparing to unpack .../35-libldap-common_2.5.19+dfsg-0ubuntu0.22.04.1_all.deb ...
Unpacking libldap-common (2.5.19+dfsg-0ubuntu0.22.04.1) over (2.5.18+dfsg-0ubuntu0.22.04.2) ...
Preparing to unpack .../36-python3-jinja2_3.0.3-1ubuntu0.4_all.deb ...
Unpacking python3-jinja2 (3.0.3-1ubuntu0.4) over (3.0.3-1ubuntu0.2) ...
Preparing to unpack .../37-snapd_2.67.1+22.04_amd64.deb ...
Unpacking snapd (2.67.1+22.04) over (2.66.1+22.04) ...
Preparing to unpack .../38-cloud-init_24.4.1-0ubuntu0~22.04.2_all.deb ...
Unpacking cloud-init (24.4.1-0ubuntu0~22.04.2) over (24.4-0ubuntu1~22.04.1) ...
Setting up libexpat1:amd64 (2.4.7-1ubuntu0.6) ...
Setting up pci.ids (0.0~2022.01.22-1ubuntu0.1) ...
Setting up distro-info-data (0.52ubuntu0.9) ...
Setting up openssh-client (1:8.9p1-3ubuntu0.13) ...
Setting up binutils-common:amd64 (2.38-4ubuntu2.8) ...
Setting up libctf-nobfd0:amd64 (2.38-4ubuntu2.8) ...
Setting up perl-modules-5.34 (5.34.0-3ubuntu1.4) ...
Setting up locales (2.35-0ubuntu3.9) ...
Generating locales (this might take a while)...
Generation complete.
Setting up landscape-common (23.02-0ubuntu1~22.04.4) ...
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 5.15.167.4-microsoft-standard-WSL2 x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 System information as of Fri May 16 10:14:55 CEST 2025

  System load:  0.58                Processes:             39
  Usage of /:   0.1% of 1006.85GB   Users logged in:       2
  Memory usage: 2%                  IPv4 address for eth0: 172.28.206.241
  Swap usage:   0%

Setting up libldap-common (2.5.19+dfsg-0ubuntu0.22.04.1) ...
Setting up libldap-2.5-0:amd64 (2.5.19+dfsg-0ubuntu0.22.04.1) ...
Setting up xxd (2:8.2.3995-1ubuntu2.24) ...
Setting up tzdata (2025b-0ubuntu0.22.04) ...

Current default time zone: 'Europe/Rome'
Local time is now:      Fri May 16 10:14:56 CEST 2025.
Universal Time is now:  Fri May 16 08:14:56 UTC 2025.
Run 'dpkg-reconfigure tzdata' if you wish to change it.

Setting up libcap2-bin (1:2.44-1ubuntu0.22.04.2) ...
Setting up python3-jinja2 (3.0.3-1ubuntu0.4) ...
Setting up vim-common (2:8.2.3995-1ubuntu2.24) ...
Setting up libfreetype6:amd64 (2.11.1+dfsg-1ubuntu0.3) ...
Setting up gnupg-l10n (2.2.27-3ubuntu2.3) ...
Setting up landscape-client (23.02-0ubuntu1~22.04.4) ...
Setting up udev (249.11-0ubuntu3.15) ...
Setting up libpython3.10-minimal:amd64 (3.10.12-1~22.04.9) ...
Setting up libdevmapper1.02.1:amd64 (2:1.02.175-2.1ubuntu5) ...
Setting up dmsetup (2:1.02.175-2.1ubuntu5) ...
Setting up gpgconf (2.2.27-3ubuntu2.3) ...
Setting up git-man (1:2.34.1-1ubuntu1.12) ...
Setting up libharfbuzz0b:amd64 (2.7.4-1ubuntu3.2) ...
Setting up libcryptsetup12:amd64 (2:2.4.3-1ubuntu1.3) ...
Setting up libbinutils:amd64 (2.38-4ubuntu2.8) ...
Setting up vim-runtime (2:8.2.3995-1ubuntu2.24) ...
Setting up openssl (3.0.2-0ubuntu1.19) ...
Setting up libelf1:amd64 (0.186-1ubuntu0.1) ...
Setting up libpam-cap:amd64 (1:2.44-1ubuntu0.22.04.2) ...
Setting up libxml2:amd64 (2.9.13+dfsg-1ubuntu0.7) ...
Setting up ubuntu-pro-client (35.1ubuntu0~22.04) ...
Installing new version of config file /etc/apparmor.d/ubuntu_pro_apt_news ...
Installing new version of config file /etc/apt/apt.conf.d/20apt-esm-hook.conf ...
Setting up gpg (2.2.27-3ubuntu2.3) ...
Setting up rsync (3.2.7-0ubuntu0.22.04.4) ...
rsync.service is a disabled or a static unit not running, not starting it.
Setting up gnupg-utils (2.2.27-3ubuntu2.3) ...
Setting up libctf0:amd64 (2.38-4ubuntu2.8) ...
Setting up ubuntu-pro-client-l10n (35.1ubuntu0~22.04) ...
Setting up libdw1:amd64 (0.186-1ubuntu0.1) ...
Setting up libperl5.34:amd64 (5.34.0-3ubuntu1.4) ...
Setting up cloud-init (24.4.1-0ubuntu0~22.04.2) ...
Setting up gpg-agent (2.2.27-3ubuntu2.3) ...
Setting up bind9-libs:amd64 (1:9.18.30-0ubuntu0.22.04.2) ...
Setting up gpgsm (2.2.27-3ubuntu2.3) ...
Setting up systemd (249.11-0ubuntu3.15) ...
Setting up vim-tiny (2:8.2.3995-1ubuntu2.24) ...
Setting up python3.10-minimal (3.10.12-1~22.04.9) ...
Setting up libpython3.10-stdlib:amd64 (3.10.12-1~22.04.9) ...
Setting up dirmngr (2.2.27-3ubuntu2.3) ...
Setting up perl (5.34.0-3ubuntu1.4) ...
Setting up systemd-timesyncd (249.11-0ubuntu3.15) ...
Setting up git (1:2.34.1-1ubuntu1.12) ...
Setting up gpg-wks-server (2.2.27-3ubuntu2.3) ...
Setting up ubuntu-advantage-tools (35.1ubuntu0~22.04) ...
Setting up bind9-host (1:9.18.30-0ubuntu0.22.04.2) ...
Setting up binutils-x86-64-linux-gnu (2.38-4ubuntu2.8) ...
Setting up snapd (2.67.1+22.04) ...
snapd.failure.service is a disabled or a static unit not running, not starting it.
snapd.snap-repair.service is a disabled or a static unit not running, not starting it.
Setting up libpython3.10:amd64 (3.10.12-1~22.04.9) ...
Setting up systemd-sysv (249.11-0ubuntu3.15) ...
Setting up vim (2:8.2.3995-1ubuntu2.24) ...
Setting up python3.10 (3.10.12-1~22.04.9) ...
Setting up gpg-wks-client (2.2.27-3ubuntu2.3) ...
Setting up libnss-systemd:amd64 (249.11-0ubuntu3.15) ...
Setting up binutils (2.38-4ubuntu2.8) ...
Setting up gnupg (2.2.27-3ubuntu2.3) ...
Setting up libpam-systemd:amd64 (249.11-0ubuntu3.15) ...
Setting up bind9-dnsutils (1:9.18.30-0ubuntu0.22.04.2) ...
Processing triggers for hicolor-icon-theme (0.17-2) ...
Processing triggers for libc-bin (2.35-0ubuntu3.9) ...
Processing triggers for rsyslog (8.2112.0-2ubuntu2.2) ...
Processing triggers for man-db (2.10.2-1) ...
Processing triggers for dbus (1.12.20-2ubuntu4.1) ...
Processing triggers for install-info (6.8-4build1) ...

meridia@TowerOfBabel:~$ sudo apt install ./amdgpu-install_6.3.60304-1_all.deb
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
Note, selecting 'amdgpu-install' instead of './amdgpu-install_6.3.60304-1_all.deb'
The following additional packages will be installed:
  dialog
The following NEW packages will be installed:
  amdgpu-install dialog
0 upgraded, 2 newly installed, 0 to remove and 87 not upgraded.
Need to get 303 kB/320 kB of archives.
After this operation, 1333 kB of additional disk space will be used.
Do you want to continue? [Y/n] y
Get:1 /home/meridia/amdgpu-install_6.3.60304-1_all.deb amdgpu-install all 6.3.60304-2125197.22.04 [17.0 kB]
Get:2 http://archive.ubuntu.com/ubuntu jammy/universe amd64 dialog amd64 1.3-20211214-1 [303 kB]
Fetched 303 kB in 1s (354 kB/s)
Selecting previously unselected package amdgpu-install.
(Reading database ... 42578 files and directories currently installed.)
Preparing to unpack .../amdgpu-install_6.3.60304-1_all.deb ...
Unpacking amdgpu-install (6.3.60304-2125197.22.04) ...
Selecting previously unselected package dialog.
Preparing to unpack .../dialog_1.3-20211214-1_amd64.deb ...
Unpacking dialog (1.3-20211214-1) ...
Setting up dialog (1.3-20211214-1) ...
Setting up amdgpu-install (6.3.60304-2125197.22.04) ...
Processing triggers for man-db (2.10.2-1) ...
N: Download is performed unsandboxed as root as file '/home/meridia/amdgpu-install_6.3.60304-1_all.deb' couldn't be accessed by user '_apt'. - pkgAcquire::Run (13: Permission denied)

meridia@TowerOfBabel:~$ sudo reboot

Session terminated, killing shell... ...killed.
Terminated
root@TowerOfBabel:/mnt/c/Users/FatherOfMachines#

C:\Users\FatherOfMachines>
```
</details>

Wait some seconds before rebooting wsl, it's a VM that is doing hidden things in the background, if you call it too early, you will get some errors when it spins up. It will still spin up fine

```
wsl
su USER
PASSWORD
cd
```

Now it's to continue the GPU installation with the ROCm driver, it's what later pytorch will use to get GPU acceleration.

```
sudo amdgpu-install --list-usecase

amdgpu-install -y --usecase=wsl,rocm --no-dkms

```

<details>
<summary>GPU install logs</summary>

```
meridia@TowerOfBabel:~$ sudo amdgpu-install --list-usecase
[sudo] password for meridia:
If --usecase option is not present, the default selection is
"dkms,graphics,opencl,hip"
Available use cases:
dkms            (to only install the kernel mode driver)
  - Kernel mode driver (included in all usecases)
graphics        (for users of graphics applications)
  - Open source Mesa 3D graphics and multimedia libraries
multimedia      (for users of open source multimedia)
  - Open source Mesa 3D multimedia libraries
workstation     (for users of legacy WS applications)
  - Open source multimedia libraries
  - Closed source (legacy) OpenGL
rocm            (for users and developers requiring full ROCm stack)
  - OpenCL (ROCr/KFD based) runtime
  - HIP runtimes
  - Machine learning framework
  - All ROCm libraries and applications
wsl             (for using ROCm in a WSL context)
  - ROCr WSL runtime library (Ubuntu 22.04 only)
rocmdev         (for developers requiring ROCm runtime and
                profiling/debugging tools)
  - HIP runtimes
  - OpenCL runtime
  - Profiler, Tracer and Debugger tools
rocmdevtools    (for developers requiring ROCm profiling/debugging tools)
  - Profiler, Tracer and Debugger tools
amf             (for users of AMF based multimedia)
  - AMF closed source multimedia library
lrt             (for users of applications requiring ROCm runtime)
  - ROCm Compiler and device libraries
  - ROCr runtime and thunk
opencl          (for users of applications requiring OpenCL on Vega or later
                products)
  - ROCr based OpenCL
  - ROCm Language runtime
openclsdk       (for application developers requiring ROCr based OpenCL)
  - ROCr based OpenCL
  - ROCm Language runtime
  - development and SDK files for ROCr based OpenCL
hip             (for users of HIP runtime on AMD products)
  - HIP runtimes
hiplibsdk       (for application developers requiring HIP on AMD products)
  - HIP runtimes
  - ROCm math libraries
  - HIP development libraries
openmpsdk       (for users of openmp/flang on AMD products)
  - OpenMP runtime and devel packages
mllib           (for users executing machine learning workloads)
  - MIOpen hip/tensile libraries
  - Clang OpenCL
  - MIOpen kernels
mlsdk           (for developers executing machine learning workloads)
  - MIOpen development libraries
  - Clang OpenCL development libraries
  - MIOpen kernels
asan            (for users of ASAN enabled ROCm packages)
  - ASAN enabled OpenCL (ROCr/KFD based) runtime
  - ASAN enabled HIP runtimes
  - ASAN enabled Machine learning framework
  - ASAN enabled ROCm libraries

meridia@TowerOfBabel:~$ amdgpu-install -y --usecase=wsl,rocm --no-dkms
Hit:1 http://archive.ubuntu.com/ubuntu jammy InRelease
Hit:2 http://security.ubuntu.com/ubuntu jammy-security InRelease
Hit:3 http://archive.ubuntu.com/ubuntu jammy-updates InRelease
Hit:4 http://archive.ubuntu.com/ubuntu jammy-backports InRelease
Get:5 https://repo.radeon.com/amdgpu/6.3.4/ubuntu jammy InRelease [5465 B]
Get:6 https://repo.radeon.com/rocm/apt/6.3.4 jammy InRelease [2605 B]
Get:7 https://repo.radeon.com/amdgpu/6.3.4/ubuntu jammy/main i386 Packages [12.2 kB]
Get:8 https://repo.radeon.com/amdgpu/6.3.4/ubuntu jammy/main amd64 Packages [14.6 kB]
Get:9 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 Packages [75.3 kB]
Fetched 110 kB in 1s (147 kB/s)
Reading package lists... Done
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following additional packages will be installed:
  alsa-topology-conf alsa-ucm-conf amd-smi-lib amdgpu-core build-essential bzip2 comgr composablekernel-dev cpp cpp-11
  dpkg-dev fakeroot ffmpeg g++ g++-11 g++-11-multilib g++-multilib gcc gcc-11 gcc-11-base gcc-11-multilib gcc-multilib
  gdb half hip-dev hip-doc hip-runtime-amd hip-samples hipblas hipblas-common-dev hipblas-dev hipblaslt hipblaslt-dev
  hipcc hipcub-dev hipfft hipfft-dev hipfort-dev hipify-clang hiprand hiprand-dev hipsolver hipsolver-dev hipsparse
  hipsparse-dev hipsparselt hipsparselt-dev hiptensor hiptensor-dev hsa-amd-aqlprofile hsa-rocr-dev i965-va-driver
  icu-devtools intel-media-va-driver javascript-common lib32asan6 lib32atomic1 lib32gcc-11-dev lib32gcc-s1 lib32gomp1
  lib32itm1 lib32quadmath0 lib32stdc++-11-dev lib32stdc++6 lib32ubsan1 libaacs0 libalgorithm-diff-perl
  libalgorithm-diff-xs-perl libalgorithm-merge-perl libamd2 libaom3 libasan6 libasound2 libasound2-data libass9
  libasyncns0 libatomic1 libavc1394-0 libavcodec-dev libavcodec58 libavdevice58 libavfilter7 libavformat-dev
  libavformat58 libavutil-dev libavutil56 libbabeltrace1 libbdplus0 libblas3 libbluray2 libboost-regex1.74.0 libbs2b0
  libc-dev-bin libc-devtools libc6-dbg libc6-dev libc6-dev-i386 libc6-dev-x32 libc6-i386 libc6-x32 libcaca0 libcamd2
  libcc1-0 libccolamd2 libcdio-cdda2 libcdio-paranoia2 libcdio19 libcholmod3 libchromaprint1 libcodec2-1.0 libcolamd2
  libcrypt-dev libdav1d5 libdc1394-25 libdebuginfod-common libdebuginfod1 libdecor-0-0 libdecor-0-plugin-1-cairo
  libdpkg-perl libdrm-amdgpu-amdgpu1 libdrm-amdgpu-common libdrm-amdgpu-dev libdrm-amdgpu-radeon1 libdrm-dev
  libdrm2-amdgpu libelf-dev libexpat1-dev libfakeroot libfile-copy-recursive-perl libfile-fcntllock-perl
  libfile-listing-perl libfile-which-perl libflac8 libflite1 libgcc-11-dev libgd3 libgfortran5 libgl-dev libglx-dev
  libgme0 libgomp1 libgsm1 libhttp-date-perl libicu-dev libiec61883-0 libigdgmm12 libipt2 libisl23 libitm1
  libjack-jackd2-0 libjs-jquery libjs-sphinxdoc libjs-underscore liblapack3 liblilv-0-0 liblsan0 libmetis5 libmfx1
  libmp3lame0 libmpc3 libmpg123-0 libmysofa1 libncurses-dev libnorm1 libnsl-dev libnuma-dev libogg0 libopenal-data
  libopenal1 libopenjp2-7 libopenmpt0 libopus0 libpciaccess-dev libpgm-5.3-0 libpocketsphinx3 libpostproc55
  libpthread-stubs0-dev libpulse0 libpython3-dev libpython3.10-dev libquadmath0 librabbitmq4 libraw1394-11
  librubberband2 libsamplerate0 libsdl2-2.0-0 libserd-0-0 libshine3 libsnappy1v5 libsndfile1 libsndio7.0 libsord-0-0
  libsource-highlight-common libsource-highlight4v5 libsoxr0 libspeex1 libsphinxbase3 libsratom-0-0 libsrt1.4-gnutls
  libssh-gcrypt-4 libstdc++-11-dev libsuitesparseconfig5 libswresample-dev libswresample3 libswscale-dev libswscale5
  libtheora0 libtimedate-perl libtinfo-dev libtirpc-dev libtsan0 libtwolame0 libubsan1 libudfread0 liburi-perl
  libva-drm2 libva-x11-2 libva2 libvdpau1 libvidstab1.1 libvorbis0a libvorbisenc2 libvorbisfile3 libvpx7 libwebpmux3
  libx11-dev libx264-163 libx265-199 libx32asan6 libx32atomic1 libx32gcc-11-dev libx32gcc-s1 libx32gomp1 libx32itm1
  libx32quadmath0 libx32stdc++-11-dev libx32stdc++6 libx32ubsan1 libxau-dev libxcb-shape0 libxcb1-dev libxdmcp-dev
  libxml2-dev libxpm4 libxss1 libxv1 libxvidcore4 libzimg2 libzmq5 libzvbi-common libzvbi0 linux-libc-dev
  lto-disabled-list make manpages-dev mesa-common-dev mesa-va-drivers mesa-vdpau-drivers migraphx migraphx-dev
  miopen-hip miopen-hip-dev mivisionx mivisionx-dev ocl-icd-libopencl1 openmp-extras-dev openmp-extras-runtime
  pocketsphinx-en-us python3-argcomplete python3-dev python3-pip python3-wheel python3.10-dev rccl rccl-dev rocalution
  rocalution-dev rocblas rocblas-dev rocfft rocfft-dev rocm-cmake rocm-core rocm-dbgapi rocm-debug-agent
  rocm-developer-tools rocm-device-libs rocm-gdb rocm-hip-libraries rocm-hip-runtime rocm-hip-runtime-dev rocm-hip-sdk
  rocm-language-runtime rocm-llvm rocm-ml-libraries rocm-ml-sdk rocm-opencl rocm-opencl-dev rocm-opencl-runtime
  rocm-opencl-sdk rocm-openmp-sdk rocm-smi-lib rocm-utils rocminfo rocprim-dev rocprofiler rocprofiler-dev
  rocprofiler-plugins rocprofiler-register rocprofiler-sdk rocprofiler-sdk-roctx rocrand rocrand-dev rocsolver
  rocsolver-dev rocsparse rocsparse-dev rocthrust-dev roctracer roctracer-dev rocwmma-dev rpcsvc-proto rpp rpp-dev
  va-driver-all valgrind vdpau-driver-all x11proto-dev xorg-sgml-doctools xtrans-dev zlib1g-dev
Suggested packages:
  bzip2-doc cpp-doc gcc-11-locales debian-keyring ffmpeg-doc gcc-11-doc lib32stdc++6-11-dbg libx32stdc++6-11-dbg
  autoconf automake libtool flex bison gcc-doc gdb-doc gdbserver i965-va-driver-shaders apache2 | lighttpd | httpd
  libasound2-plugins alsa-utils libcuda1 libnvcuvid1 libnvidia-encode1 libbluray-bdj glibc-doc bzr libgd-tools icu-doc
  jackd2 ncurses-doc libportaudio2 opus-tools pulseaudio libraw1394-doc xdg-utils serdi sndiod sordi speex
  libstdc++-11-doc libbusiness-isbn-perl libwww-perl libx11-doc libxcb-doc pkg-config make-doc opencl-icd valgrind-dbg
  valgrind-mpi kcachegrind alleyoop valkyrie libvdpau-va-gl1
The following NEW packages will be installed:
  alsa-topology-conf alsa-ucm-conf amd-smi-lib amdgpu-core build-essential bzip2 comgr composablekernel-dev cpp cpp-11
  dpkg-dev fakeroot ffmpeg g++ g++-11 g++-11-multilib g++-multilib gcc gcc-11 gcc-11-base gcc-11-multilib gcc-multilib
  gdb half hip-dev hip-doc hip-runtime-amd hip-samples hipblas hipblas-common-dev hipblas-dev hipblaslt hipblaslt-dev
  hipcc hipcub-dev hipfft hipfft-dev hipfort-dev hipify-clang hiprand hiprand-dev hipsolver hipsolver-dev hipsparse
  hipsparse-dev hipsparselt hipsparselt-dev hiptensor hiptensor-dev hsa-amd-aqlprofile hsa-rocr-dev
  hsa-runtime-rocr4wsl-amdgpu i965-va-driver icu-devtools intel-media-va-driver javascript-common lib32asan6
  lib32atomic1 lib32gcc-11-dev lib32gcc-s1 lib32gomp1 lib32itm1 lib32quadmath0 lib32stdc++-11-dev lib32stdc++6
  lib32ubsan1 libaacs0 libalgorithm-diff-perl libalgorithm-diff-xs-perl libalgorithm-merge-perl libamd2 libaom3
  libasan6 libasound2 libasound2-data libass9 libasyncns0 libatomic1 libavc1394-0 libavcodec-dev libavcodec58
  libavdevice58 libavfilter7 libavformat-dev libavformat58 libavutil-dev libavutil56 libbabeltrace1 libbdplus0
  libblas3 libbluray2 libboost-regex1.74.0 libbs2b0 libc-dev-bin libc-devtools libc6-dbg libc6-dev libc6-dev-i386
  libc6-dev-x32 libc6-i386 libc6-x32 libcaca0 libcamd2 libcc1-0 libccolamd2 libcdio-cdda2 libcdio-paranoia2 libcdio19
  libcholmod3 libchromaprint1 libcodec2-1.0 libcolamd2 libcrypt-dev libdav1d5 libdc1394-25 libdebuginfod-common
  libdebuginfod1 libdecor-0-0 libdecor-0-plugin-1-cairo libdpkg-perl libdrm-amdgpu-amdgpu1 libdrm-amdgpu-common
  libdrm-amdgpu-dev libdrm-amdgpu-radeon1 libdrm-dev libdrm2-amdgpu libelf-dev libexpat1-dev libfakeroot
  libfile-copy-recursive-perl libfile-fcntllock-perl libfile-listing-perl libfile-which-perl libflac8 libflite1
  libgcc-11-dev libgd3 libgfortran5 libgl-dev libglx-dev libgme0 libgomp1 libgsm1 libhttp-date-perl libicu-dev
  libiec61883-0 libigdgmm12 libipt2 libisl23 libitm1 libjack-jackd2-0 libjs-jquery libjs-sphinxdoc libjs-underscore
  liblapack3 liblilv-0-0 liblsan0 libmetis5 libmfx1 libmp3lame0 libmpc3 libmpg123-0 libmysofa1 libncurses-dev libnorm1
  libnsl-dev libnuma-dev libogg0 libopenal-data libopenal1 libopenjp2-7 libopenmpt0 libopus0 libpciaccess-dev
  libpgm-5.3-0 libpocketsphinx3 libpostproc55 libpthread-stubs0-dev libpulse0 libpython3-dev libpython3.10-dev
  libquadmath0 librabbitmq4 libraw1394-11 librubberband2 libsamplerate0 libsdl2-2.0-0 libserd-0-0 libshine3
  libsnappy1v5 libsndfile1 libsndio7.0 libsord-0-0 libsource-highlight-common libsource-highlight4v5 libsoxr0
  libspeex1 libsphinxbase3 libsratom-0-0 libsrt1.4-gnutls libssh-gcrypt-4 libstdc++-11-dev libsuitesparseconfig5
  libswresample-dev libswresample3 libswscale-dev libswscale5 libtheora0 libtimedate-perl libtinfo-dev libtirpc-dev
  libtsan0 libtwolame0 libubsan1 libudfread0 liburi-perl libva-drm2 libva-x11-2 libva2 libvdpau1 libvidstab1.1
  libvorbis0a libvorbisenc2 libvorbisfile3 libvpx7 libwebpmux3 libx11-dev libx264-163 libx265-199 libx32asan6
  libx32atomic1 libx32gcc-11-dev libx32gcc-s1 libx32gomp1 libx32itm1 libx32quadmath0 libx32stdc++-11-dev libx32stdc++6
  libx32ubsan1 libxau-dev libxcb-shape0 libxcb1-dev libxdmcp-dev libxml2-dev libxpm4 libxss1 libxv1 libxvidcore4
  libzimg2 libzmq5 libzvbi-common libzvbi0 linux-libc-dev lto-disabled-list make manpages-dev mesa-common-dev
  mesa-va-drivers mesa-vdpau-drivers migraphx migraphx-dev miopen-hip miopen-hip-dev mivisionx mivisionx-dev
  ocl-icd-libopencl1 openmp-extras-dev openmp-extras-runtime pocketsphinx-en-us python3-argcomplete python3-dev
  python3-pip python3-wheel python3.10-dev rccl rccl-dev rocalution rocalution-dev rocblas rocblas-dev rocfft
  rocfft-dev rocm rocm-cmake rocm-core rocm-dbgapi rocm-debug-agent rocm-developer-tools rocm-device-libs rocm-gdb
  rocm-hip-libraries rocm-hip-runtime rocm-hip-runtime-dev rocm-hip-sdk rocm-language-runtime rocm-llvm
  rocm-ml-libraries rocm-ml-sdk rocm-opencl rocm-opencl-dev rocm-opencl-runtime rocm-opencl-sdk rocm-openmp-sdk
  rocm-smi-lib rocm-utils rocminfo rocprim-dev rocprofiler rocprofiler-dev rocprofiler-plugins rocprofiler-register
  rocprofiler-sdk rocprofiler-sdk-roctx rocrand rocrand-dev rocsolver rocsolver-dev rocsparse rocsparse-dev
  rocthrust-dev roctracer roctracer-dev rocwmma-dev rpcsvc-proto rpp rpp-dev va-driver-all valgrind vdpau-driver-all
  x11proto-dev xorg-sgml-doctools xtrans-dev zlib1g-dev
0 upgraded, 333 newly installed, 0 to remove and 0 not upgraded.
Need to get 3018 MB of archives.
After this operation, 35.5 GB of additional disk space will be used.
Get:1 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 rocm-core amd64 6.3.4.60304-76~22.04 [14.0 kB]
Get:2 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libdebuginfod-common all 0.186-1ubuntu0.1 [7996 B]
Get:3 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 amd-smi-lib amd64 25.1.0.60304-76~22.04 [1392 kB]
Get:4 http://archive.ubuntu.com/ubuntu jammy/main amd64 alsa-topology-conf all 1.2.5.1-2 [15.5 kB]
Get:5 http://archive.ubuntu.com/ubuntu jammy/main amd64 libasound2-data all 1.2.6.1-1ubuntu1 [19.1 kB]
Get:6 http://archive.ubuntu.com/ubuntu jammy/main amd64 libasound2 amd64 1.2.6.1-1ubuntu1 [390 kB]
Get:7 https://repo.radeon.com/amdgpu/6.3.4/ubuntu jammy/main amd64 amdgpu-core all 1:6.3.60304-2125197.22.04 [2224 B]
Get:8 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 comgr amd64 2.8.0.60304-76~22.04 [54.4 MB]
Get:9 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 alsa-ucm-conf all 1.2.6.3-1ubuntu1.12 [43.5 kB]
Get:10 http://archive.ubuntu.com/ubuntu jammy-updates/universe amd64 python3-wheel all 0.37.1-2ubuntu0.22.04.1 [32.0 kB]
Get:11 http://archive.ubuntu.com/ubuntu jammy-updates/universe amd64 python3-pip all 22.0.2+dfsg-1ubuntu0.5 [1306 kB]
Get:12 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libc-dev-bin amd64 2.35-0ubuntu3.9 [20.3 kB]
Get:13 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 linux-libc-dev amd64 5.15.0-140.150 [1313 kB]
Get:14 http://archive.ubuntu.com/ubuntu jammy/main amd64 libcrypt-dev amd64 1:4.4.27-1 [112 kB]
Get:15 http://archive.ubuntu.com/ubuntu jammy/main amd64 rpcsvc-proto amd64 1.4.2-0ubuntu6 [68.5 kB]
Get:16 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libtirpc-dev amd64 1.3.2-2ubuntu0.1 [192 kB]
Get:17 http://archive.ubuntu.com/ubuntu jammy/main amd64 libnsl-dev amd64 1.3.0-2build2 [71.3 kB]
Get:18 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libc6-dev amd64 2.35-0ubuntu3.9 [2100 kB]
Get:19 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 gcc-11-base amd64 11.4.0-1ubuntu1~22.04 [20.2 kB]
Get:20 http://archive.ubuntu.com/ubuntu jammy/main amd64 libisl23 amd64 0.24-2build1 [727 kB]
Get:21 http://archive.ubuntu.com/ubuntu jammy/main amd64 libmpc3 amd64 1.2.1-2build1 [46.9 kB]
Get:22 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 cpp-11 amd64 11.4.0-1ubuntu1~22.04 [10.0 MB]
Get:23 http://archive.ubuntu.com/ubuntu jammy/main amd64 cpp amd64 4:11.2.0-1ubuntu1 [27.7 kB]
Get:24 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libcc1-0 amd64 12.3.0-1ubuntu1~22.04 [48.3 kB]
Get:25 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libgomp1 amd64 12.3.0-1ubuntu1~22.04 [126 kB]
Get:26 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libitm1 amd64 12.3.0-1ubuntu1~22.04 [30.2 kB]
Get:27 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libatomic1 amd64 12.3.0-1ubuntu1~22.04 [10.4 kB]
Get:28 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libasan6 amd64 11.4.0-1ubuntu1~22.04 [2282 kB]
Get:29 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 liblsan0 amd64 12.3.0-1ubuntu1~22.04 [1069 kB]
Get:30 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libtsan0 amd64 11.4.0-1ubuntu1~22.04 [2260 kB]
Get:31 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libubsan1 amd64 12.3.0-1ubuntu1~22.04 [976 kB]
Get:32 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libquadmath0 amd64 12.3.0-1ubuntu1~22.04 [154 kB]
Get:33 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libgcc-11-dev amd64 11.4.0-1ubuntu1~22.04 [2517 kB]
Get:34 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 gcc-11 amd64 11.4.0-1ubuntu1~22.04 [20.1 MB]
Get:35 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 composablekernel-dev amd64 1.1.0.60304-76~22.04 [522 MB]
Get:36 http://archive.ubuntu.com/ubuntu jammy/main amd64 gcc amd64 4:11.2.0-1ubuntu1 [5112 B]
Get:37 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libstdc++-11-dev amd64 11.4.0-1ubuntu1~22.04 [2101 kB]
Get:38 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 g++-11 amd64 11.4.0-1ubuntu1~22.04 [11.4 MB]
Get:39 http://archive.ubuntu.com/ubuntu jammy/main amd64 g++ amd64 4:11.2.0-1ubuntu1 [1412 B]
Get:40 http://archive.ubuntu.com/ubuntu jammy/main amd64 make amd64 4.3-4.1build1 [180 kB]
Get:41 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libdpkg-perl all 1.21.1ubuntu2.3 [237 kB]
Get:42 http://archive.ubuntu.com/ubuntu jammy/main amd64 bzip2 amd64 1.0.8-5build1 [34.8 kB]
Get:43 http://archive.ubuntu.com/ubuntu jammy/main amd64 lto-disabled-list all 24 [12.5 kB]
Get:44 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 dpkg-dev all 1.21.1ubuntu2.3 [922 kB]
Get:45 http://archive.ubuntu.com/ubuntu jammy/main amd64 build-essential amd64 12.9ubuntu3 [4744 B]
Get:46 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libncurses-dev amd64 6.3-2ubuntu0.1 [381 kB]
Get:47 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libtinfo-dev amd64 6.3-2ubuntu0.1 [780 B]
Get:48 http://archive.ubuntu.com/ubuntu jammy/main amd64 libfakeroot amd64 1.28-1ubuntu1 [31.5 kB]
Get:49 http://archive.ubuntu.com/ubuntu jammy/main amd64 fakeroot amd64 1.28-1ubuntu1 [60.4 kB]
Get:50 http://archive.ubuntu.com/ubuntu jammy-updates/universe amd64 libaom3 amd64 3.3.0-1ubuntu0.1 [1748 kB]
Get:51 http://archive.ubuntu.com/ubuntu jammy/universe amd64 libva2 amd64 2.14.0-1 [65.0 kB]
Get:52 http://archive.ubuntu.com/ubuntu jammy/universe amd64 libmfx1 amd64 22.3.0-1 [3105 kB]
Get:53 http://archive.ubuntu.com/ubuntu jammy/universe amd64 libva-drm2 amd64 2.14.0-1 [7502 B]
Get:54 http://archive.ubuntu.com/ubuntu jammy/universe amd64 libva-x11-2 amd64 2.14.0-1 [12.6 kB]
Get:55 http://archive.ubuntu.com/ubuntu jammy/main amd64 libvdpau1 amd64 1.4-3build2 [27.0 kB]
Get:56 http://archive.ubuntu.com/ubuntu jammy/universe amd64 ocl-icd-libopencl1 amd64 2.2.14-3 [39.1 kB]
Get:57 http://archive.ubuntu.com/ubuntu jammy-updates/universe amd64 libavutil56 amd64 7:4.4.2-0ubuntu0.22.04.1 [290 kB]
Get:58 http://archive.ubuntu.com/ubuntu jammy/universe amd64 libcodec2-1.0 amd64 1.0.1-3 [8435 kB]
Get:59 http://archive.ubuntu.com/ubuntu jammy/universe amd64 libdav1d5 amd64 0.9.2-1 [463 kB]
Get:60 http://archive.ubuntu.com/ubuntu jammy/universe amd64 libgsm1 amd64 1.0.19-1 [27.7 kB]
Get:61 http://archive.ubuntu.com/ubuntu jammy/main amd64 libmp3lame0 amd64 3.100-3build2 [141 kB]
Get:62 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libopenjp2-7 amd64 2.4.0-6ubuntu0.3 [158 kB]
Get:63 http://archive.ubuntu.com/ubuntu jammy/main amd64 libopus0 amd64 1.3.1-0.1build2 [203 kB]
Get:64 http://archive.ubuntu.com/ubuntu jammy/universe amd64 libshine3 amd64 3.1.1-2 [23.2 kB]
Get:65 http://archive.ubuntu.com/ubuntu jammy/main amd64 libsnappy1v5 amd64 1.1.8-1build3 [17.5 kB]
Get:66 http://archive.ubuntu.com/ubuntu jammy/main amd64 libspeex1 amd64 1.2~rc1.2-1.1ubuntu3 [57.9 kB]
Get:67 http://archive.ubuntu.com/ubuntu jammy/main amd64 libsoxr0 amd64 0.1.3-4build2 [79.8 kB]
Get:68 http://archive.ubuntu.com/ubuntu jammy-updates/universe amd64 libswresample3 amd64 7:4.4.2-0ubuntu0.22.04.1 [62.2 kB]
Get:69 http://archive.ubuntu.com/ubuntu jammy/main amd64 libogg0 amd64 1.3.5-0ubuntu3 [22.9 kB]
Get:70 http://archive.ubuntu.com/ubuntu jammy/main amd64 libtheora0 amd64 1.1.1+dfsg.1-15ubuntu4 [209 kB]
Get:71 http://archive.ubuntu.com/ubuntu jammy/main amd64 libtwolame0 amd64 0.4.0-2build2 [52.5 kB]
Get:72 http://archive.ubuntu.com/ubuntu jammy/main amd64 libvorbis0a amd64 1.3.7-1build2 [99.2 kB]
Get:73 http://archive.ubuntu.com/ubuntu jammy/main amd64 libvorbisenc2 amd64 1.3.7-1build2 [82.6 kB]
Get:74 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libvpx7 amd64 1.11.0-2ubuntu2.3 [1078 kB]
Get:75 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libwebpmux3 amd64 1.2.2-2ubuntu0.22.04.2 [20.5 kB]
Get:76 http://archive.ubuntu.com/ubuntu jammy/universe amd64 libx264-163 amd64 2:0.163.3060+git5db6aa6-2build1 [591 kB]
Get:77 http://archive.ubuntu.com/ubuntu jammy/universe amd64 libx265-199 amd64 3.5-2 [1170 kB]
Get:78 http://archive.ubuntu.com/ubuntu jammy/universe amd64 libxvidcore4 amd64 2:1.3.7-1 [201 kB]
Get:79 http://archive.ubuntu.com/ubuntu jammy/universe amd64 libzvbi-common all 0.2.35-19 [35.5 kB]
Get:80 http://archive.ubuntu.com/ubuntu jammy/universe amd64 libzvbi0 amd64 0.2.35-19 [262 kB]
Get:81 http://archive.ubuntu.com/ubuntu jammy-updates/universe amd64 libavcodec58 amd64 7:4.4.2-0ubuntu0.22.04.1 [5567 kB]
Get:82 http://archive.ubuntu.com/ubuntu jammy/main amd64 libraw1394-11 amd64 2.1.2-2build2 [27.0 kB]
Get:83 http://archive.ubuntu.com/ubuntu jammy/main amd64 libavc1394-0 amd64 0.5.4-5build2 [17.0 kB]
Get:84 http://archive.ubuntu.com/ubuntu jammy/universe amd64 libass9 amd64 1:0.15.2-1 [97.5 kB]
Get:85 http://archive.ubuntu.com/ubuntu jammy/universe amd64 libudfread0 amd64 1.1.2-1 [16.2 kB]
Get:86 http://archive.ubuntu.com/ubuntu jammy/universe amd64 libbluray2 amd64 1:1.3.1-1 [159 kB]
Get:87 http://archive.ubuntu.com/ubuntu jammy/universe amd64 libchromaprint1 amd64 1.5.1-2 [28.4 kB]
Get:88 http://archive.ubuntu.com/ubuntu jammy/universe amd64 libgme0 amd64 0.6.3-2 [127 kB]
Get:89 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libmpg123-0 amd64 1.29.3-1ubuntu0.1 [172 kB]
Get:90 http://archive.ubuntu.com/ubuntu jammy/main amd64 libvorbisfile3 amd64 1.3.7-1build2 [17.1 kB]
Get:91 http://archive.ubuntu.com/ubuntu jammy/universe amd64 libopenmpt0 amd64 0.6.1-1 [592 kB]
Get:92 http://archive.ubuntu.com/ubuntu jammy/main amd64 librabbitmq4 amd64 0.10.0-1ubuntu2 [39.3 kB]
Get:93 http://archive.ubuntu.com/ubuntu jammy/universe amd64 libsrt1.4-gnutls amd64 1.4.4-4 [309 kB]
Get:94 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libssh-gcrypt-4 amd64 0.9.6-2ubuntu0.22.04.3 [223 kB]
Get:95 http://archive.ubuntu.com/ubuntu jammy/universe amd64 libnorm1 amd64 1.5.9+dfsg-2 [221 kB]
Get:96 http://archive.ubuntu.com/ubuntu jammy/universe amd64 libpgm-5.3-0 amd64 5.3.128~dfsg-2 [161 kB]
Get:97 http://archive.ubuntu.com/ubuntu jammy/universe amd64 libzmq5 amd64 4.3.4-2 [256 kB]
Get:98 http://archive.ubuntu.com/ubuntu jammy-updates/universe amd64 libavformat58 amd64 7:4.4.2-0ubuntu0.22.04.1 [1103 kB]
Get:99 http://archive.ubuntu.com/ubuntu jammy/universe amd64 libbs2b0 amd64 3.1.0+dfsg-2.2build1 [10.2 kB]
Get:100 http://archive.ubuntu.com/ubuntu jammy/universe amd64 libflite1 amd64 2.2-3 [13.7 MB]
Get:101 http://archive.ubuntu.com/ubuntu jammy/universe amd64 libserd-0-0 amd64 0.30.10-2 [40.8 kB]
Get:102 http://archive.ubuntu.com/ubuntu jammy/universe amd64 libsord-0-0 amd64 0.16.8-2 [21.2 kB]
Get:103 http://archive.ubuntu.com/ubuntu jammy/universe amd64 libsratom-0-0 amd64 0.6.8-1 [17.0 kB]
Get:104 http://archive.ubuntu.com/ubuntu jammy/universe amd64 liblilv-0-0 amd64 0.24.12-2 [42.8 kB]
Get:105 http://archive.ubuntu.com/ubuntu jammy/universe amd64 libmysofa1 amd64 1.2.1~dfsg0-1 [1157 kB]
Get:106 http://archive.ubuntu.com/ubuntu jammy/main amd64 libblas3 amd64 3.10.0-2ubuntu1 [228 kB]
Get:107 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libgfortran5 amd64 12.3.0-1ubuntu1~22.04 [879 kB]
Get:108 http://archive.ubuntu.com/ubuntu jammy/main amd64 liblapack3 amd64 3.10.0-2ubuntu1 [2504 kB]
Get:109 http://archive.ubuntu.com/ubuntu jammy/main amd64 libasyncns0 amd64 0.8-6build2 [12.8 kB]
Get:110 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libflac8 amd64 1.3.3-2ubuntu0.2 [111 kB]
Get:111 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libsndfile1 amd64 1.0.31-2ubuntu0.2 [196 kB]
Get:112 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libpulse0 amd64 1:15.99.1+dfsg1-1ubuntu2.2 [298 kB]
Get:113 http://archive.ubuntu.com/ubuntu jammy/universe amd64 libsphinxbase3 amd64 0.8+5prealpha+1-13build1 [126 kB]
Get:114 http://archive.ubuntu.com/ubuntu jammy/universe amd64 libpocketsphinx3 amd64 0.8.0+real5prealpha+1-14ubuntu1 [132 kB]
Get:115 http://archive.ubuntu.com/ubuntu jammy-updates/universe amd64 libpostproc55 amd64 7:4.4.2-0ubuntu0.22.04.1 [60.1 kB]
Get:116 http://archive.ubuntu.com/ubuntu jammy/main amd64 libsamplerate0 amd64 0.2.2-1build1 [1359 kB]
Get:117 http://archive.ubuntu.com/ubuntu jammy/universe amd64 librubberband2 amd64 2.0.0-2 [90.0 kB]
Get:118 http://archive.ubuntu.com/ubuntu jammy-updates/universe amd64 libswscale5 amd64 7:4.4.2-0ubuntu0.22.04.1 [180 kB]
Get:119 http://archive.ubuntu.com/ubuntu jammy/universe amd64 libvidstab1.1 amd64 1.1.0-2 [35.0 kB]
Get:120 http://archive.ubuntu.com/ubuntu jammy/universe amd64 libzimg2 amd64 3.0.3+ds1-1 [241 kB]
Get:121 http://archive.ubuntu.com/ubuntu jammy-updates/universe amd64 libavfilter7 amd64 7:4.4.2-0ubuntu0.22.04.1 [1496 kB]
Get:122 http://archive.ubuntu.com/ubuntu jammy/main amd64 libcaca0 amd64 0.99.beta19-2.2ubuntu4 [224 kB]
Get:123 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libcdio19 amd64 2.1.0-3ubuntu0.2 [63.6 kB]
Get:124 http://archive.ubuntu.com/ubuntu jammy/main amd64 libcdio-cdda2 amd64 10.2+2.0.0-1build3 [16.7 kB]
Get:125 http://archive.ubuntu.com/ubuntu jammy/main amd64 libcdio-paranoia2 amd64 10.2+2.0.0-1build3 [15.9 kB]
Get:126 http://archive.ubuntu.com/ubuntu jammy/universe amd64 libdc1394-25 amd64 2.2.6-4 [88.8 kB]
Get:127 http://archive.ubuntu.com/ubuntu jammy/main amd64 libiec61883-0 amd64 1.2.0-4build3 [25.9 kB]
Get:128 http://archive.ubuntu.com/ubuntu jammy/main amd64 libjack-jackd2-0 amd64 1.9.20~dfsg-1 [293 kB]
Get:129 http://archive.ubuntu.com/ubuntu jammy/universe amd64 libopenal-data all 1:1.19.1-2build3 [164 kB]
Get:130 http://archive.ubuntu.com/ubuntu jammy/universe amd64 libsndio7.0 amd64 1.8.1-1.1 [29.3 kB]
Get:131 http://archive.ubuntu.com/ubuntu jammy/universe amd64 libopenal1 amd64 1:1.19.1-2build3 [535 kB]
Get:132 http://archive.ubuntu.com/ubuntu jammy/main amd64 libdecor-0-0 amd64 0.1.0-3build1 [15.1 kB]
Get:133 http://archive.ubuntu.com/ubuntu jammy/main amd64 libxss1 amd64 1:1.2.3-1build2 [8476 B]
Get:134 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libsdl2-2.0-0 amd64 2.0.20+dfsg-2ubuntu1.22.04.1 [582 kB]
Get:135 http://archive.ubuntu.com/ubuntu jammy/main amd64 libxcb-shape0 amd64 1.14-3ubuntu3 [6158 B]
Get:136 http://archive.ubuntu.com/ubuntu jammy/main amd64 libxv1 amd64 2:1.0.11-1build2 [11.2 kB]
Get:137 http://archive.ubuntu.com/ubuntu jammy-updates/universe amd64 libavdevice58 amd64 7:4.4.2-0ubuntu0.22.04.1 [87.5 kB]
Get:138 http://archive.ubuntu.com/ubuntu jammy-updates/universe amd64 ffmpeg amd64 7:4.4.2-0ubuntu0.22.04.1 [1696 kB]
Get:139 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libc6-i386 amd64 2.35-0ubuntu3.9 [2838 kB]
Get:140 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libc6-dev-i386 amd64 2.35-0ubuntu3.9 [1445 kB]
Get:141 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libc6-x32 amd64 2.35-0ubuntu3.9 [2978 kB]
Get:142 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libc6-dev-x32 amd64 2.35-0ubuntu3.9 [1632 kB]
Get:143 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 lib32gcc-s1 amd64 12.3.0-1ubuntu1~22.04 [63.9 kB]
Get:144 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libx32gcc-s1 amd64 12.3.0-1ubuntu1~22.04 [54.0 kB]
Get:145 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 lib32gomp1 amd64 12.3.0-1ubuntu1~22.04 [133 kB]
Get:146 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libx32gomp1 amd64 12.3.0-1ubuntu1~22.04 [127 kB]
Get:147 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 lib32itm1 amd64 12.3.0-1ubuntu1~22.04 [32.0 kB]
Get:148 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libx32itm1 amd64 12.3.0-1ubuntu1~22.04 [30.2 kB]
Get:149 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 lib32atomic1 amd64 12.3.0-1ubuntu1~22.04 [8500 B]
Get:150 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libx32atomic1 amd64 12.3.0-1ubuntu1~22.04 [10.2 kB]
Get:151 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 lib32asan6 amd64 11.4.0-1ubuntu1~22.04 [2154 kB]
Get:152 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libx32asan6 amd64 11.4.0-1ubuntu1~22.04 [2128 kB]
Get:153 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 lib32stdc++6 amd64 12.3.0-1ubuntu1~22.04 [740 kB]
Get:154 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 lib32ubsan1 amd64 12.3.0-1ubuntu1~22.04 [959 kB]
Get:155 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libx32stdc++6 amd64 12.3.0-1ubuntu1~22.04 [682 kB]
Get:156 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libx32ubsan1 amd64 12.3.0-1ubuntu1~22.04 [963 kB]
Get:157 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 lib32quadmath0 amd64 12.3.0-1ubuntu1~22.04 [244 kB]
Get:158 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libx32quadmath0 amd64 12.3.0-1ubuntu1~22.04 [156 kB]
Get:159 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 lib32gcc-11-dev amd64 11.4.0-1ubuntu1~22.04 [2339 kB]
Get:160 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libx32gcc-11-dev amd64 11.4.0-1ubuntu1~22.04 [2107 kB]
Get:161 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 gcc-11-multilib amd64 11.4.0-1ubuntu1~22.04 [876 B]
Get:162 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 lib32stdc++-11-dev amd64 11.4.0-1ubuntu1~22.04 [989 kB]
Get:163 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libx32stdc++-11-dev amd64 11.4.0-1ubuntu1~22.04 [906 kB]
Get:164 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 g++-11-multilib amd64 11.4.0-1ubuntu1~22.04 [890 B]
Get:165 http://archive.ubuntu.com/ubuntu jammy/main amd64 gcc-multilib amd64 4:11.2.0-1ubuntu1 [1382 B]
Get:166 http://archive.ubuntu.com/ubuntu jammy/main amd64 g++-multilib amd64 4:11.2.0-1ubuntu1 [854 B]
Get:167 http://archive.ubuntu.com/ubuntu jammy/main amd64 libbabeltrace1 amd64 1.5.8-2build1 [160 kB]
Get:168 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libdebuginfod1 amd64 0.186-1ubuntu0.1 [12.8 kB]
Get:169 http://archive.ubuntu.com/ubuntu jammy/main amd64 libipt2 amd64 2.0.5-1 [46.4 kB]
Get:170 http://archive.ubuntu.com/ubuntu jammy/main amd64 libsource-highlight-common all 3.1.9-4.1build2 [64.5 kB]
Get:171 http://archive.ubuntu.com/ubuntu jammy/main amd64 libboost-regex1.74.0 amd64 1.74.0-14ubuntu3 [511 kB]
Get:172 http://archive.ubuntu.com/ubuntu jammy/main amd64 libsource-highlight4v5 amd64 3.1.9-4.1build2 [207 kB]
Get:173 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 gdb amd64 12.1-0ubuntu1~22.04.2 [3920 kB]
Get:174 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 half amd64 1.12.0.60304-76~22.04 [19.7 kB]
Get:175 https://repo.radeon.com/amdgpu/6.3.4/ubuntu jammy/main amd64 hsa-runtime-rocr4wsl-amdgpu amd64 24.30-2127960.22.04 [412 kB]
Get:176 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 rocminfo amd64 1.0.0.60304-76~22.04 [27.4 kB]
Get:177 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 rocprofiler-register amd64 0.4.0.60304-76~22.04 [223 kB]
Get:178 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 hip-runtime-amd amd64 6.3.42134.60304-76~22.04 [13.6 MB]
Get:179 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 rocm-llvm amd64 18.0.0.25012.60304-76~22.04 [326 MB]
Get:180 http://archive.ubuntu.com/ubuntu jammy/universe amd64 libfile-copy-recursive-perl all 0.45-1 [17.3 kB]
Get:181 http://archive.ubuntu.com/ubuntu jammy/main amd64 libtimedate-perl all 2.3300-2 [34.0 kB]
Get:182 http://archive.ubuntu.com/ubuntu jammy/main amd64 libhttp-date-perl all 6.05-1 [9920 B]
Get:183 http://archive.ubuntu.com/ubuntu jammy/main amd64 libfile-listing-perl all 6.14-1 [11.2 kB]
Get:184 http://archive.ubuntu.com/ubuntu jammy/main amd64 libfile-which-perl all 1.23-1 [13.8 kB]
Get:185 http://archive.ubuntu.com/ubuntu jammy/main amd64 liburi-perl all 5.10-1 [78.8 kB]
Get:186 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libc6-dbg amd64 2.35-0ubuntu3.9 [13.8 MB]
Get:187 http://archive.ubuntu.com/ubuntu jammy/main amd64 valgrind amd64 1:3.18.1-1ubuntu2 [14.1 MB]
Get:188 http://archive.ubuntu.com/ubuntu jammy/main amd64 libpciaccess-dev amd64 0.16-3 [21.9 kB]
Get:189 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libdrm-dev amd64 2.4.113-2~ubuntu0.22.04.1 [292 kB]
Get:190 http://archive.ubuntu.com/ubuntu jammy/main amd64 icu-devtools amd64 70.1-2 [197 kB]
Get:191 http://archive.ubuntu.com/ubuntu jammy/universe amd64 libigdgmm12 amd64 22.1.2+ds1-1 [139 kB]
Get:192 http://archive.ubuntu.com/ubuntu jammy-updates/universe amd64 intel-media-va-driver amd64 22.3.1+dfsg1-1ubuntu2 [2283 kB]
Get:193 http://archive.ubuntu.com/ubuntu jammy/main amd64 javascript-common all 11+nmu1 [5936 B]
Get:194 http://archive.ubuntu.com/ubuntu jammy/universe amd64 libaacs0 amd64 0.11.1-1 [64.1 kB]
Get:195 http://archive.ubuntu.com/ubuntu jammy/main amd64 libalgorithm-diff-perl all 1.201-1 [41.8 kB]
Get:196 http://archive.ubuntu.com/ubuntu jammy/main amd64 libalgorithm-diff-xs-perl amd64 0.04-6build3 [11.9 kB]
Get:197 http://archive.ubuntu.com/ubuntu jammy/main amd64 libalgorithm-merge-perl all 0.08-3 [12.0 kB]
Get:198 http://archive.ubuntu.com/ubuntu jammy/main amd64 libsuitesparseconfig5 amd64 1:5.10.1+dfsg-4build1 [10.4 kB]
Get:199 http://archive.ubuntu.com/ubuntu jammy/universe amd64 libamd2 amd64 1:5.10.1+dfsg-4build1 [21.6 kB]
Get:200 http://archive.ubuntu.com/ubuntu jammy-updates/universe amd64 libavutil-dev amd64 7:4.4.2-0ubuntu0.22.04.1 [427 kB]
Get:201 http://archive.ubuntu.com/ubuntu jammy-updates/universe amd64 libswresample-dev amd64 7:4.4.2-0ubuntu0.22.04.1 [78.0 kB]
Get:202 http://archive.ubuntu.com/ubuntu jammy-updates/universe amd64 libavcodec-dev amd64 7:4.4.2-0ubuntu0.22.04.1 [6221 kB]
Get:203 http://archive.ubuntu.com/ubuntu jammy-updates/universe amd64 libavformat-dev amd64 7:4.4.2-0ubuntu0.22.04.1 [1347 kB]
Get:204 http://archive.ubuntu.com/ubuntu jammy/universe amd64 libbdplus0 amd64 0.2.0-1 [52.2 kB]
Get:205 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libxpm4 amd64 1:3.5.12-1ubuntu0.22.04.2 [36.7 kB]
Get:206 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libgd3 amd64 2.3.0-2ubuntu2.3 [129 kB]
Get:207 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libc-devtools amd64 2.35-0ubuntu3.9 [29.0 kB]
Get:208 http://archive.ubuntu.com/ubuntu jammy/universe amd64 libcamd2 amd64 1:5.10.1+dfsg-4build1 [23.3 kB]
Get:209 http://archive.ubuntu.com/ubuntu jammy/universe amd64 libccolamd2 amd64 1:5.10.1+dfsg-4build1 [25.2 kB]
Get:210 http://archive.ubuntu.com/ubuntu jammy/main amd64 libcolamd2 amd64 1:5.10.1+dfsg-4build1 [18.0 kB]
Get:211 http://archive.ubuntu.com/ubuntu jammy/universe amd64 libmetis5 amd64 5.1.0.dfsg-7build2 [181 kB]
Get:212 http://archive.ubuntu.com/ubuntu jammy/universe amd64 libcholmod3 amd64 1:5.10.1+dfsg-4build1 [346 kB]
Get:213 http://archive.ubuntu.com/ubuntu jammy/main amd64 libdecor-0-plugin-1-cairo amd64 0.1.0-3build1 [20.4 kB]
Get:214 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 zlib1g-dev amd64 1:1.2.11.dfsg-2ubuntu9.2 [164 kB]
Get:215 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libelf-dev amd64 0.186-1ubuntu0.1 [64.4 kB]
Get:216 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libexpat1-dev amd64 2.4.7-1ubuntu0.6 [148 kB]
Get:217 http://archive.ubuntu.com/ubuntu jammy/main amd64 libfile-fcntllock-perl amd64 0.22-3build7 [33.9 kB]
Get:218 http://archive.ubuntu.com/ubuntu jammy/main amd64 xorg-sgml-doctools all 1:1.11-1.1 [10.9 kB]
Get:219 http://archive.ubuntu.com/ubuntu jammy/main amd64 x11proto-dev all 2021.5-1 [604 kB]
Get:220 http://archive.ubuntu.com/ubuntu jammy/main amd64 libxau-dev amd64 1:1.0.9-1build5 [9724 B]
Get:221 http://archive.ubuntu.com/ubuntu jammy/main amd64 libxdmcp-dev amd64 1:1.1.3-0ubuntu5 [26.5 kB]
Get:222 http://archive.ubuntu.com/ubuntu jammy/main amd64 xtrans-dev all 1.4.0-1 [68.9 kB]
Get:223 http://archive.ubuntu.com/ubuntu jammy/main amd64 libpthread-stubs0-dev amd64 0.4-1build2 [5516 B]
Get:224 http://archive.ubuntu.com/ubuntu jammy/main amd64 libxcb1-dev amd64 1.14-3ubuntu3 [86.5 kB]
Get:225 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libx11-dev amd64 2:1.7.5-1ubuntu0.3 [744 kB]
Get:226 http://archive.ubuntu.com/ubuntu jammy/main amd64 libglx-dev amd64 1.4.0-1 [14.1 kB]
Get:227 http://archive.ubuntu.com/ubuntu jammy/main amd64 libgl-dev amd64 1.4.0-1 [101 kB]
Get:228 http://archive.ubuntu.com/ubuntu jammy/main amd64 libicu-dev amd64 70.1-2 [11.6 MB]
Get:229 http://archive.ubuntu.com/ubuntu jammy/main amd64 libjs-jquery all 3.6.0+dfsg+~3.5.13-1 [321 kB]
Get:230 http://archive.ubuntu.com/ubuntu jammy/main amd64 libjs-underscore all 1.13.2~dfsg-2 [118 kB]
Get:231 http://archive.ubuntu.com/ubuntu jammy/main amd64 libjs-sphinxdoc all 4.3.2-1 [139 kB]
Get:232 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libpython3.10-dev amd64 3.10.12-1~22.04.9 [4763 kB]
Get:233 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libpython3-dev amd64 3.10.6-1~22.04.1 [7064 B]
Get:234 http://archive.ubuntu.com/ubuntu jammy-updates/universe amd64 libswscale-dev amd64 7:4.4.2-0ubuntu0.22.04.1 [206 kB]
Get:235 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libxml2-dev amd64 2.9.13+dfsg-1ubuntu0.7 [804 kB]
Get:236 http://archive.ubuntu.com/ubuntu jammy/main amd64 manpages-dev all 5.10-1ubuntu1 [2309 kB]
Get:237 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 mesa-common-dev amd64 23.2.1-1ubuntu3.1~22.04.3 [2208 kB]
Get:238 http://archive.ubuntu.com/ubuntu jammy-updates/universe amd64 mesa-va-drivers amd64 23.2.1-1ubuntu3.1~22.04.3 [4100 kB]
Get:239 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 mesa-vdpau-drivers amd64 23.2.1-1ubuntu3.1~22.04.3 [3820 kB]
Get:240 http://archive.ubuntu.com/ubuntu jammy/universe amd64 python3-argcomplete all 1.8.1-1.5 [27.2 kB]
Get:241 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 python3.10-dev amd64 3.10.12-1~22.04.9 [508 kB]
Get:242 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 python3-dev amd64 3.10.6-1~22.04.1 [26.0 kB]
Get:243 http://archive.ubuntu.com/ubuntu jammy/main amd64 libnuma-dev amd64 2.0.14-3ubuntu2 [35.9 kB]
Get:244 http://archive.ubuntu.com/ubuntu jammy/universe amd64 i965-va-driver amd64 2.4.1+dfsg1-1 [302 kB]
Get:245 http://archive.ubuntu.com/ubuntu jammy/universe amd64 va-driver-all amd64 2.14.0-1 [3984 B]
Get:246 http://archive.ubuntu.com/ubuntu jammy/main amd64 vdpau-driver-all amd64 1.4-3build2 [4510 B]
Get:247 http://archive.ubuntu.com/ubuntu jammy/universe amd64 pocketsphinx-en-us all 0.8.0+real5prealpha+1-14ubuntu1 [27.6 MB]
Get:248 https://repo.radeon.com/amdgpu/6.3.4/ubuntu jammy/main amd64 libdrm2-amdgpu amd64 1:2.4.123.60304-2125197.22.04 [37.9 kB]
Get:249 https://repo.radeon.com/amdgpu/6.3.4/ubuntu jammy/main amd64 libdrm-amdgpu-common all 1.0.0.60304-2125197.22.04 [5216 B]
Get:250 https://repo.radeon.com/amdgpu/6.3.4/ubuntu jammy/main amd64 libdrm-amdgpu-amdgpu1 amd64 1:2.4.123.60304-2125197.22.04 [22.1 kB]
Get:251 https://repo.radeon.com/amdgpu/6.3.4/ubuntu jammy/main amd64 libdrm-amdgpu-radeon1 amd64 1:2.4.123.60304-2125197.22.04 [22.7 kB]
Get:252 https://repo.radeon.com/amdgpu/6.3.4/ubuntu jammy/main amd64 libdrm-amdgpu-dev amd64 1:2.4.123.60304-2125197.22.04 [144 kB]
Get:253 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 hsa-rocr-dev amd64 1.14.0.60304-76~22.04 [135 kB]
Get:254 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 hip-dev amd64 6.3.42134.60304-76~22.04 [316 kB]
Get:255 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 hip-doc amd64 6.3.42134.60304-76~22.04 [89.8 kB]
Get:256 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 hipcc amd64 1.1.1.60304-76~22.04 [217 kB]
Get:257 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 hip-samples amd64 6.3.42134.60304-76~22.04 [51.7 kB]
Get:258 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 hipblaslt amd64 0.10.0.60304-76~22.04 [329 MB]
Get:259 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 rocblas amd64 4.3.0.60304-76~22.04 [149 MB]
Get:260 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 rocsolver amd64 3.27.0.60304-76~22.04 [262 MB]
Get:261 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 hipblas amd64 2.3.0.60304-76~22.04 [156 kB]
Get:262 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 hipblas-common-dev amd64 1.0.0.60304-76~22.04 [5764 B]
Get:263 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 hipblas-dev amd64 2.3.0.60304-76~22.04 [98.3 kB]
Get:264 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 hipblaslt-dev amd64 0.10.0.60304-76~22.04 [27.0 kB]
Get:265 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 rocprim-dev amd64 3.3.0.60304-76~22.04 [231 kB]
Get:266 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 hipcub-dev amd64 3.3.0.60304-76~22.04 [74.5 kB]
Get:267 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 rocfft amd64 1.0.31.60304-76~22.04 [120 MB]
Get:268 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 hipfft amd64 1.0.17.60304-76~22.04 [25.7 kB]
Get:269 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 hipfft-dev amd64 1.0.17.60304-76~22.04 [11.2 kB]
Get:270 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 hipfort-dev amd64 0.5.1.60304-76~22.04 [6667 kB]
Get:271 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 hipify-clang amd64 18.0.0.60304-76~22.04 [21.0 MB]
Get:272 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 hiprand amd64 2.11.1.60304-76~22.04 [5014 B]
Get:273 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 hiprand-dev amd64 2.11.1.60304-76~22.04 [21.2 kB]
Get:274 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 hipsolver amd64 2.3.0.60304-76~22.04 [54.6 kB]
Get:275 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 hipsolver-dev amd64 2.3.0.60304-76~22.04 [18.9 kB]
Get:276 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 rocsparse amd64 3.3.0.60304-76~22.04 [187 MB]
Get:277 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 hipsparse amd64 3.1.2.60304-76~22.04 [45.7 kB]
Get:278 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 hipsparse-dev amd64 3.1.2.60304-76~22.04 [49.1 kB]
Get:279 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 hipsparselt amd64 0.2.2.60304-76~22.04 [10.4 MB]
Get:280 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 hipsparselt-dev amd64 0.2.2.60304-76~22.04 [11.7 kB]
Get:281 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 hiptensor amd64 1.4.0.60304-76~22.04 [34.3 MB]
Get:282 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 hiptensor-dev amd64 1.4.0.60304-76~22.04 [13.2 kB]
Get:283 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 hsa-amd-aqlprofile amd64 1.0.0.60304-76~22.04 [492 kB]
Get:284 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 roctracer amd64 4.1.60304.60304-76~22.04 [473 kB]
Get:285 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 rocrand amd64 3.2.0.60304-76~22.04 [23.1 MB]
Get:286 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 miopen-hip amd64 3.3.0.60304-76~22.04 [162 MB]
Get:287 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 migraphx amd64 2.11.0.60304-76~22.04 [48.9 MB]
Get:288 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 migraphx-dev amd64 2.11.0.60304-76~22.04 [172 kB]
Get:289 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 miopen-hip-dev amd64 3.3.0.60304-76~22.04 [47.7 kB]
Get:290 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 openmp-extras-runtime amd64 18.63.0.60304-76~22.04 [153 MB]
Get:291 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 rocm-language-runtime amd64 6.3.4.60304-76~22.04 [836 B]
Get:292 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 rocm-hip-runtime amd64 6.3.4.60304-76~22.04 [2106 B]
Get:293 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 rpp amd64 1.9.1.60304-76~22.04 [70.1 MB]
Get:294 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 mivisionx amd64 3.1.0.60304-76~22.04 [35.8 MB]
Get:295 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 rocm-device-libs amd64 1.0.0.60304-76~22.04 [720 kB]
Get:296 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 rocm-cmake amd64 0.14.0.60304-76~22.04 [24.7 kB]
Get:297 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 rocm-hip-runtime-dev amd64 6.3.4.60304-76~22.04 [2290 B]
Get:298 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 rpp-dev amd64 1.9.1.60304-76~22.04 [48.0 kB]
Get:299 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 rocblas-dev amd64 4.3.0.60304-76~22.04 [99.0 kB]
Get:300 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 mivisionx-dev amd64 3.1.0.60304-76~22.04 [23.6 MB]
Get:301 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 openmp-extras-dev amd64 18.63.0.60304-76~22.04 [51.0 MB]
Get:302 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 rocm-smi-lib amd64 7.4.0.60304-76~22.04 [1032 kB]
Get:303 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 rccl amd64 2.21.5.60304-76~22.04 [53.8 MB]
Get:304 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 rccl-dev amd64 2.21.5.60304-76~22.04 [107 kB]
Get:305 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 rocalution amd64 3.2.1.60304-76~22.04 [4958 kB]
Get:306 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 rocalution-dev amd64 3.2.1.60304-76~22.04 [43.0 kB]
Get:307 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 rocfft-dev amd64 1.0.31.60304-76~22.04 [10.7 kB]
Get:308 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 rocm-utils amd64 6.3.4.60304-76~22.04 [810 B]
Get:309 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 rocm-dbgapi amd64 0.77.0.60304-76~22.04 [1788 kB]
Get:310 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 rocm-debug-agent amd64 2.0.3.60304-76~22.04 [56.6 kB]
Get:311 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 rocm-gdb amd64 15.2.60304-76~22.04 [92.3 MB]
Get:312 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 rocprofiler amd64 2.0.60304.60304-76~22.04 [929 kB]
Get:313 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 rocprofiler-plugins amd64 2.0.60304.60304-76~22.04 [1056 kB]
Get:314 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 rocprofiler-sdk-roctx amd64 0.5.0-76~22.04 [192 kB]
Get:315 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 rocprofiler-sdk amd64 0.5.0-76~22.04 [3648 kB]
Get:316 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 rocprofiler-dev amd64 2.0.60304.60304-76~22.04 [23.9 kB]
Get:317 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 roctracer-dev amd64 4.1.60304.60304-76~22.04 [461 kB]
Get:318 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 rocm-developer-tools amd64 6.3.4.60304-76~22.04 [2250 B]
Get:319 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 rocm-openmp-sdk amd64 6.3.4.60304-76~22.04 [874 B]
Get:320 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 rocm-opencl amd64 2.0.0.60304-76~22.04 [636 kB]
Get:321 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 rocm-opencl-runtime amd64 6.3.4.60304-76~22.04 [2074 B]
Get:322 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 rocm-opencl-dev amd64 2.0.0.60304-76~22.04 [121 kB]
Get:323 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 rocm-opencl-sdk amd64 6.3.4.60304-76~22.04 [828 B]
Get:324 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 rocm-hip-libraries amd64 6.3.4.60304-76~22.04 [948 B]
Get:325 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 rocm-ml-libraries amd64 6.3.4.60304-76~22.04 [850 B]
Get:326 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 rocrand-dev amd64 3.2.0.60304-76~22.04 [545 kB]
Get:327 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 rocsolver-dev amd64 3.27.0.60304-76~22.04 [52.1 kB]
Get:328 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 rocsparse-dev amd64 3.3.0.60304-76~22.04 [95.8 kB]
Get:329 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 rocthrust-dev amd64 3.3.0.60304-76~22.04 [420 kB]
Get:330 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 rocwmma-dev amd64 1.6.0.60304-76~22.04 [70.2 kB]
Get:331 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 rocm-hip-sdk amd64 6.3.4.60304-76~22.04 [2268 B]
Get:332 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 rocm-ml-sdk amd64 6.3.4.60304-76~22.04 [828 B]
Get:333 https://repo.radeon.com/rocm/apt/6.3.4 jammy/main amd64 rocm amd64 6.3.4.60304-76~22.04 [2138 B]
Fetched 3018 MB in 6min 16s (8025 kB/s)
Extracting templates from packages: 100%
Preconfiguring packages ...
Selecting previously unselected package libdebuginfod-common.
(Reading database ... 42772 files and directories currently installed.)
Preparing to unpack .../000-libdebuginfod-common_0.186-1ubuntu0.1_all.deb ...
Unpacking libdebuginfod-common (0.186-1ubuntu0.1) ...
Selecting previously unselected package alsa-topology-conf.
Preparing to unpack .../001-alsa-topology-conf_1.2.5.1-2_all.deb ...
Unpacking alsa-topology-conf (1.2.5.1-2) ...
Selecting previously unselected package libasound2-data.
Preparing to unpack .../002-libasound2-data_1.2.6.1-1ubuntu1_all.deb ...
Unpacking libasound2-data (1.2.6.1-1ubuntu1) ...
Selecting previously unselected package libasound2:amd64.
Preparing to unpack .../003-libasound2_1.2.6.1-1ubuntu1_amd64.deb ...
Unpacking libasound2:amd64 (1.2.6.1-1ubuntu1) ...
Selecting previously unselected package alsa-ucm-conf.
Preparing to unpack .../004-alsa-ucm-conf_1.2.6.3-1ubuntu1.12_all.deb ...
Unpacking alsa-ucm-conf (1.2.6.3-1ubuntu1.12) ...
Selecting previously unselected package python3-wheel.
Preparing to unpack .../005-python3-wheel_0.37.1-2ubuntu0.22.04.1_all.deb ...
Unpacking python3-wheel (0.37.1-2ubuntu0.22.04.1) ...
Selecting previously unselected package python3-pip.
Preparing to unpack .../006-python3-pip_22.0.2+dfsg-1ubuntu0.5_all.deb ...
Unpacking python3-pip (22.0.2+dfsg-1ubuntu0.5) ...
Selecting previously unselected package rocm-core.
Preparing to unpack .../007-rocm-core_6.3.4.60304-76~22.04_amd64.deb ...
Unpacking rocm-core (6.3.4.60304-76~22.04) ...
Selecting previously unselected package amd-smi-lib.
Preparing to unpack .../008-amd-smi-lib_25.1.0.60304-76~22.04_amd64.deb ...
Unpacking amd-smi-lib (25.1.0.60304-76~22.04) ...
Selecting previously unselected package amdgpu-core.
Preparing to unpack .../009-amdgpu-core_1%3a6.3.60304-2125197.22.04_all.deb ...
Unpacking amdgpu-core (1:6.3.60304-2125197.22.04) ...
Selecting previously unselected package libc-dev-bin.
Preparing to unpack .../010-libc-dev-bin_2.35-0ubuntu3.9_amd64.deb ...
Unpacking libc-dev-bin (2.35-0ubuntu3.9) ...
Selecting previously unselected package linux-libc-dev:amd64.
Preparing to unpack .../011-linux-libc-dev_5.15.0-140.150_amd64.deb ...
Unpacking linux-libc-dev:amd64 (5.15.0-140.150) ...
Selecting previously unselected package libcrypt-dev:amd64.
Preparing to unpack .../012-libcrypt-dev_1%3a4.4.27-1_amd64.deb ...
Unpacking libcrypt-dev:amd64 (1:4.4.27-1) ...
Selecting previously unselected package rpcsvc-proto.
Preparing to unpack .../013-rpcsvc-proto_1.4.2-0ubuntu6_amd64.deb ...
Unpacking rpcsvc-proto (1.4.2-0ubuntu6) ...
Selecting previously unselected package libtirpc-dev:amd64.
Preparing to unpack .../014-libtirpc-dev_1.3.2-2ubuntu0.1_amd64.deb ...
Unpacking libtirpc-dev:amd64 (1.3.2-2ubuntu0.1) ...
Selecting previously unselected package libnsl-dev:amd64.
Preparing to unpack .../015-libnsl-dev_1.3.0-2build2_amd64.deb ...
Unpacking libnsl-dev:amd64 (1.3.0-2build2) ...
Selecting previously unselected package libc6-dev:amd64.
Preparing to unpack .../016-libc6-dev_2.35-0ubuntu3.9_amd64.deb ...
Unpacking libc6-dev:amd64 (2.35-0ubuntu3.9) ...
Selecting previously unselected package gcc-11-base:amd64.
Preparing to unpack .../017-gcc-11-base_11.4.0-1ubuntu1~22.04_amd64.deb ...
Unpacking gcc-11-base:amd64 (11.4.0-1ubuntu1~22.04) ...
Selecting previously unselected package libisl23:amd64.
Preparing to unpack .../018-libisl23_0.24-2build1_amd64.deb ...
Unpacking libisl23:amd64 (0.24-2build1) ...
Selecting previously unselected package libmpc3:amd64.
Preparing to unpack .../019-libmpc3_1.2.1-2build1_amd64.deb ...
Unpacking libmpc3:amd64 (1.2.1-2build1) ...
Selecting previously unselected package cpp-11.
Preparing to unpack .../020-cpp-11_11.4.0-1ubuntu1~22.04_amd64.deb ...
Unpacking cpp-11 (11.4.0-1ubuntu1~22.04) ...
Selecting previously unselected package cpp.
Preparing to unpack .../021-cpp_4%3a11.2.0-1ubuntu1_amd64.deb ...
Unpacking cpp (4:11.2.0-1ubuntu1) ...
Selecting previously unselected package libcc1-0:amd64.
Preparing to unpack .../022-libcc1-0_12.3.0-1ubuntu1~22.04_amd64.deb ...
Unpacking libcc1-0:amd64 (12.3.0-1ubuntu1~22.04) ...
Selecting previously unselected package libgomp1:amd64.
Preparing to unpack .../023-libgomp1_12.3.0-1ubuntu1~22.04_amd64.deb ...
Unpacking libgomp1:amd64 (12.3.0-1ubuntu1~22.04) ...
Selecting previously unselected package libitm1:amd64.
Preparing to unpack .../024-libitm1_12.3.0-1ubuntu1~22.04_amd64.deb ...
Unpacking libitm1:amd64 (12.3.0-1ubuntu1~22.04) ...
Selecting previously unselected package libatomic1:amd64.
Preparing to unpack .../025-libatomic1_12.3.0-1ubuntu1~22.04_amd64.deb ...
Unpacking libatomic1:amd64 (12.3.0-1ubuntu1~22.04) ...
Selecting previously unselected package libasan6:amd64.
Preparing to unpack .../026-libasan6_11.4.0-1ubuntu1~22.04_amd64.deb ...
Unpacking libasan6:amd64 (11.4.0-1ubuntu1~22.04) ...
Selecting previously unselected package liblsan0:amd64.
Preparing to unpack .../027-liblsan0_12.3.0-1ubuntu1~22.04_amd64.deb ...
Unpacking liblsan0:amd64 (12.3.0-1ubuntu1~22.04) ...
Selecting previously unselected package libtsan0:amd64.
Preparing to unpack .../028-libtsan0_11.4.0-1ubuntu1~22.04_amd64.deb ...
Unpacking libtsan0:amd64 (11.4.0-1ubuntu1~22.04) ...
Selecting previously unselected package libubsan1:amd64.
Preparing to unpack .../029-libubsan1_12.3.0-1ubuntu1~22.04_amd64.deb ...
Unpacking libubsan1:amd64 (12.3.0-1ubuntu1~22.04) ...
Selecting previously unselected package libquadmath0:amd64.
Preparing to unpack .../030-libquadmath0_12.3.0-1ubuntu1~22.04_amd64.deb ...
Unpacking libquadmath0:amd64 (12.3.0-1ubuntu1~22.04) ...
Selecting previously unselected package libgcc-11-dev:amd64.
Preparing to unpack .../031-libgcc-11-dev_11.4.0-1ubuntu1~22.04_amd64.deb ...
Unpacking libgcc-11-dev:amd64 (11.4.0-1ubuntu1~22.04) ...
Selecting previously unselected package gcc-11.
Preparing to unpack .../032-gcc-11_11.4.0-1ubuntu1~22.04_amd64.deb ...
Unpacking gcc-11 (11.4.0-1ubuntu1~22.04) ...
Selecting previously unselected package gcc.
Preparing to unpack .../033-gcc_4%3a11.2.0-1ubuntu1_amd64.deb ...
Unpacking gcc (4:11.2.0-1ubuntu1) ...
Selecting previously unselected package libstdc++-11-dev:amd64.
Preparing to unpack .../034-libstdc++-11-dev_11.4.0-1ubuntu1~22.04_amd64.deb ...
Unpacking libstdc++-11-dev:amd64 (11.4.0-1ubuntu1~22.04) ...
Selecting previously unselected package g++-11.
Preparing to unpack .../035-g++-11_11.4.0-1ubuntu1~22.04_amd64.deb ...
Unpacking g++-11 (11.4.0-1ubuntu1~22.04) ...
Selecting previously unselected package g++.
Preparing to unpack .../036-g++_4%3a11.2.0-1ubuntu1_amd64.deb ...
Unpacking g++ (4:11.2.0-1ubuntu1) ...
Selecting previously unselected package make.
Preparing to unpack .../037-make_4.3-4.1build1_amd64.deb ...
Unpacking make (4.3-4.1build1) ...
Selecting previously unselected package libdpkg-perl.
Preparing to unpack .../038-libdpkg-perl_1.21.1ubuntu2.3_all.deb ...
Unpacking libdpkg-perl (1.21.1ubuntu2.3) ...
Selecting previously unselected package bzip2.
Preparing to unpack .../039-bzip2_1.0.8-5build1_amd64.deb ...
Unpacking bzip2 (1.0.8-5build1) ...
Selecting previously unselected package lto-disabled-list.
Preparing to unpack .../040-lto-disabled-list_24_all.deb ...
Unpacking lto-disabled-list (24) ...
Selecting previously unselected package dpkg-dev.
Preparing to unpack .../041-dpkg-dev_1.21.1ubuntu2.3_all.deb ...
Unpacking dpkg-dev (1.21.1ubuntu2.3) ...
Selecting previously unselected package build-essential.
Preparing to unpack .../042-build-essential_12.9ubuntu3_amd64.deb ...
Unpacking build-essential (12.9ubuntu3) ...
Selecting previously unselected package libncurses-dev:amd64.
Preparing to unpack .../043-libncurses-dev_6.3-2ubuntu0.1_amd64.deb ...
Unpacking libncurses-dev:amd64 (6.3-2ubuntu0.1) ...
Selecting previously unselected package libtinfo-dev:amd64.
Preparing to unpack .../044-libtinfo-dev_6.3-2ubuntu0.1_amd64.deb ...
Unpacking libtinfo-dev:amd64 (6.3-2ubuntu0.1) ...
Selecting previously unselected package comgr.
Preparing to unpack .../045-comgr_2.8.0.60304-76~22.04_amd64.deb ...
Unpacking comgr (2.8.0.60304-76~22.04) ...
Selecting previously unselected package composablekernel-dev.
Preparing to unpack .../046-composablekernel-dev_1.1.0.60304-76~22.04_amd64.deb ...
Unpacking composablekernel-dev (1.1.0.60304-76~22.04) ...
Selecting previously unselected package libfakeroot:amd64.
Preparing to unpack .../047-libfakeroot_1.28-1ubuntu1_amd64.deb ...
Unpacking libfakeroot:amd64 (1.28-1ubuntu1) ...
Selecting previously unselected package fakeroot.
Preparing to unpack .../048-fakeroot_1.28-1ubuntu1_amd64.deb ...
Unpacking fakeroot (1.28-1ubuntu1) ...
Selecting previously unselected package libaom3:amd64.
Preparing to unpack .../049-libaom3_3.3.0-1ubuntu0.1_amd64.deb ...
Unpacking libaom3:amd64 (3.3.0-1ubuntu0.1) ...
Selecting previously unselected package libva2:amd64.
Preparing to unpack .../050-libva2_2.14.0-1_amd64.deb ...
Unpacking libva2:amd64 (2.14.0-1) ...
Selecting previously unselected package libmfx1:amd64.
Preparing to unpack .../051-libmfx1_22.3.0-1_amd64.deb ...
Unpacking libmfx1:amd64 (22.3.0-1) ...
Selecting previously unselected package libva-drm2:amd64.
Preparing to unpack .../052-libva-drm2_2.14.0-1_amd64.deb ...
Unpacking libva-drm2:amd64 (2.14.0-1) ...
Selecting previously unselected package libva-x11-2:amd64.
Preparing to unpack .../053-libva-x11-2_2.14.0-1_amd64.deb ...
Unpacking libva-x11-2:amd64 (2.14.0-1) ...
Selecting previously unselected package libvdpau1:amd64.
Preparing to unpack .../054-libvdpau1_1.4-3build2_amd64.deb ...
Unpacking libvdpau1:amd64 (1.4-3build2) ...
Selecting previously unselected package ocl-icd-libopencl1:amd64.
Preparing to unpack .../055-ocl-icd-libopencl1_2.2.14-3_amd64.deb ...
Unpacking ocl-icd-libopencl1:amd64 (2.2.14-3) ...
Selecting previously unselected package libavutil56:amd64.
Preparing to unpack .../056-libavutil56_7%3a4.4.2-0ubuntu0.22.04.1_amd64.deb ...
Unpacking libavutil56:amd64 (7:4.4.2-0ubuntu0.22.04.1) ...
Selecting previously unselected package libcodec2-1.0:amd64.
Preparing to unpack .../057-libcodec2-1.0_1.0.1-3_amd64.deb ...
Unpacking libcodec2-1.0:amd64 (1.0.1-3) ...
Selecting previously unselected package libdav1d5:amd64.
Preparing to unpack .../058-libdav1d5_0.9.2-1_amd64.deb ...
Unpacking libdav1d5:amd64 (0.9.2-1) ...
Selecting previously unselected package libgsm1:amd64.
Preparing to unpack .../059-libgsm1_1.0.19-1_amd64.deb ...
Unpacking libgsm1:amd64 (1.0.19-1) ...
Selecting previously unselected package libmp3lame0:amd64.
Preparing to unpack .../060-libmp3lame0_3.100-3build2_amd64.deb ...
Unpacking libmp3lame0:amd64 (3.100-3build2) ...
Selecting previously unselected package libopenjp2-7:amd64.
Preparing to unpack .../061-libopenjp2-7_2.4.0-6ubuntu0.3_amd64.deb ...
Unpacking libopenjp2-7:amd64 (2.4.0-6ubuntu0.3) ...
Selecting previously unselected package libopus0:amd64.
Preparing to unpack .../062-libopus0_1.3.1-0.1build2_amd64.deb ...
Unpacking libopus0:amd64 (1.3.1-0.1build2) ...
Selecting previously unselected package libshine3:amd64.
Preparing to unpack .../063-libshine3_3.1.1-2_amd64.deb ...
Unpacking libshine3:amd64 (3.1.1-2) ...
Selecting previously unselected package libsnappy1v5:amd64.
Preparing to unpack .../064-libsnappy1v5_1.1.8-1build3_amd64.deb ...
Unpacking libsnappy1v5:amd64 (1.1.8-1build3) ...
Selecting previously unselected package libspeex1:amd64.
Preparing to unpack .../065-libspeex1_1.2~rc1.2-1.1ubuntu3_amd64.deb ...
Unpacking libspeex1:amd64 (1.2~rc1.2-1.1ubuntu3) ...
Selecting previously unselected package libsoxr0:amd64.
Preparing to unpack .../066-libsoxr0_0.1.3-4build2_amd64.deb ...
Unpacking libsoxr0:amd64 (0.1.3-4build2) ...
Selecting previously unselected package libswresample3:amd64.
Preparing to unpack .../067-libswresample3_7%3a4.4.2-0ubuntu0.22.04.1_amd64.deb ...
Unpacking libswresample3:amd64 (7:4.4.2-0ubuntu0.22.04.1) ...
Selecting previously unselected package libogg0:amd64.
Preparing to unpack .../068-libogg0_1.3.5-0ubuntu3_amd64.deb ...
Unpacking libogg0:amd64 (1.3.5-0ubuntu3) ...
Selecting previously unselected package libtheora0:amd64.
Preparing to unpack .../069-libtheora0_1.1.1+dfsg.1-15ubuntu4_amd64.deb ...
Unpacking libtheora0:amd64 (1.1.1+dfsg.1-15ubuntu4) ...
Selecting previously unselected package libtwolame0:amd64.
Preparing to unpack .../070-libtwolame0_0.4.0-2build2_amd64.deb ...
Unpacking libtwolame0:amd64 (0.4.0-2build2) ...
Selecting previously unselected package libvorbis0a:amd64.
Preparing to unpack .../071-libvorbis0a_1.3.7-1build2_amd64.deb ...
Unpacking libvorbis0a:amd64 (1.3.7-1build2) ...
Selecting previously unselected package libvorbisenc2:amd64.
Preparing to unpack .../072-libvorbisenc2_1.3.7-1build2_amd64.deb ...
Unpacking libvorbisenc2:amd64 (1.3.7-1build2) ...
Selecting previously unselected package libvpx7:amd64.
Preparing to unpack .../073-libvpx7_1.11.0-2ubuntu2.3_amd64.deb ...
Unpacking libvpx7:amd64 (1.11.0-2ubuntu2.3) ...
Selecting previously unselected package libwebpmux3:amd64.
Preparing to unpack .../074-libwebpmux3_1.2.2-2ubuntu0.22.04.2_amd64.deb ...
Unpacking libwebpmux3:amd64 (1.2.2-2ubuntu0.22.04.2) ...
Selecting previously unselected package libx264-163:amd64.
Preparing to unpack .../075-libx264-163_2%3a0.163.3060+git5db6aa6-2build1_amd64.deb ...
Unpacking libx264-163:amd64 (2:0.163.3060+git5db6aa6-2build1) ...
Selecting previously unselected package libx265-199:amd64.
Preparing to unpack .../076-libx265-199_3.5-2_amd64.deb ...
Unpacking libx265-199:amd64 (3.5-2) ...
Selecting previously unselected package libxvidcore4:amd64.
Preparing to unpack .../077-libxvidcore4_2%3a1.3.7-1_amd64.deb ...
Unpacking libxvidcore4:amd64 (2:1.3.7-1) ...
Selecting previously unselected package libzvbi-common.
Preparing to unpack .../078-libzvbi-common_0.2.35-19_all.deb ...
Unpacking libzvbi-common (0.2.35-19) ...
Selecting previously unselected package libzvbi0:amd64.
Preparing to unpack .../079-libzvbi0_0.2.35-19_amd64.deb ...
Unpacking libzvbi0:amd64 (0.2.35-19) ...
Selecting previously unselected package libavcodec58:amd64.
Preparing to unpack .../080-libavcodec58_7%3a4.4.2-0ubuntu0.22.04.1_amd64.deb ...
Unpacking libavcodec58:amd64 (7:4.4.2-0ubuntu0.22.04.1) ...
Selecting previously unselected package libraw1394-11:amd64.
Preparing to unpack .../081-libraw1394-11_2.1.2-2build2_amd64.deb ...
Unpacking libraw1394-11:amd64 (2.1.2-2build2) ...
Selecting previously unselected package libavc1394-0:amd64.
Preparing to unpack .../082-libavc1394-0_0.5.4-5build2_amd64.deb ...
Unpacking libavc1394-0:amd64 (0.5.4-5build2) ...
Selecting previously unselected package libass9:amd64.
Preparing to unpack .../083-libass9_1%3a0.15.2-1_amd64.deb ...
Unpacking libass9:amd64 (1:0.15.2-1) ...
Selecting previously unselected package libudfread0:amd64.
Preparing to unpack .../084-libudfread0_1.1.2-1_amd64.deb ...
Unpacking libudfread0:amd64 (1.1.2-1) ...
Selecting previously unselected package libbluray2:amd64.
Preparing to unpack .../085-libbluray2_1%3a1.3.1-1_amd64.deb ...
Unpacking libbluray2:amd64 (1:1.3.1-1) ...
Selecting previously unselected package libchromaprint1:amd64.
Preparing to unpack .../086-libchromaprint1_1.5.1-2_amd64.deb ...
Unpacking libchromaprint1:amd64 (1.5.1-2) ...
Selecting previously unselected package libgme0:amd64.
Preparing to unpack .../087-libgme0_0.6.3-2_amd64.deb ...
Unpacking libgme0:amd64 (0.6.3-2) ...
Selecting previously unselected package libmpg123-0:amd64.
Preparing to unpack .../088-libmpg123-0_1.29.3-1ubuntu0.1_amd64.deb ...
Unpacking libmpg123-0:amd64 (1.29.3-1ubuntu0.1) ...
Selecting previously unselected package libvorbisfile3:amd64.
Preparing to unpack .../089-libvorbisfile3_1.3.7-1build2_amd64.deb ...
Unpacking libvorbisfile3:amd64 (1.3.7-1build2) ...
Selecting previously unselected package libopenmpt0:amd64.
Preparing to unpack .../090-libopenmpt0_0.6.1-1_amd64.deb ...
Unpacking libopenmpt0:amd64 (0.6.1-1) ...
Selecting previously unselected package librabbitmq4:amd64.
Preparing to unpack .../091-librabbitmq4_0.10.0-1ubuntu2_amd64.deb ...
Unpacking librabbitmq4:amd64 (0.10.0-1ubuntu2) ...
Selecting previously unselected package libsrt1.4-gnutls:amd64.
Preparing to unpack .../092-libsrt1.4-gnutls_1.4.4-4_amd64.deb ...
Unpacking libsrt1.4-gnutls:amd64 (1.4.4-4) ...
Selecting previously unselected package libssh-gcrypt-4:amd64.
Preparing to unpack .../093-libssh-gcrypt-4_0.9.6-2ubuntu0.22.04.3_amd64.deb ...
Unpacking libssh-gcrypt-4:amd64 (0.9.6-2ubuntu0.22.04.3) ...
Selecting previously unselected package libnorm1:amd64.
Preparing to unpack .../094-libnorm1_1.5.9+dfsg-2_amd64.deb ...
Unpacking libnorm1:amd64 (1.5.9+dfsg-2) ...
Selecting previously unselected package libpgm-5.3-0:amd64.
Preparing to unpack .../095-libpgm-5.3-0_5.3.128~dfsg-2_amd64.deb ...
Unpacking libpgm-5.3-0:amd64 (5.3.128~dfsg-2) ...
Selecting previously unselected package libzmq5:amd64.
Preparing to unpack .../096-libzmq5_4.3.4-2_amd64.deb ...
Unpacking libzmq5:amd64 (4.3.4-2) ...
Selecting previously unselected package libavformat58:amd64.
Preparing to unpack .../097-libavformat58_7%3a4.4.2-0ubuntu0.22.04.1_amd64.deb ...
Unpacking libavformat58:amd64 (7:4.4.2-0ubuntu0.22.04.1) ...
Selecting previously unselected package libbs2b0:amd64.
Preparing to unpack .../098-libbs2b0_3.1.0+dfsg-2.2build1_amd64.deb ...
Unpacking libbs2b0:amd64 (3.1.0+dfsg-2.2build1) ...
Selecting previously unselected package libflite1:amd64.
Preparing to unpack .../099-libflite1_2.2-3_amd64.deb ...
Unpacking libflite1:amd64 (2.2-3) ...
Selecting previously unselected package libserd-0-0:amd64.
Preparing to unpack .../100-libserd-0-0_0.30.10-2_amd64.deb ...
Unpacking libserd-0-0:amd64 (0.30.10-2) ...
Selecting previously unselected package libsord-0-0:amd64.
Preparing to unpack .../101-libsord-0-0_0.16.8-2_amd64.deb ...
Unpacking libsord-0-0:amd64 (0.16.8-2) ...
Selecting previously unselected package libsratom-0-0:amd64.
Preparing to unpack .../102-libsratom-0-0_0.6.8-1_amd64.deb ...
Unpacking libsratom-0-0:amd64 (0.6.8-1) ...
Selecting previously unselected package liblilv-0-0:amd64.
Preparing to unpack .../103-liblilv-0-0_0.24.12-2_amd64.deb ...
Unpacking liblilv-0-0:amd64 (0.24.12-2) ...
Selecting previously unselected package libmysofa1:amd64.
Preparing to unpack .../104-libmysofa1_1.2.1~dfsg0-1_amd64.deb ...
Unpacking libmysofa1:amd64 (1.2.1~dfsg0-1) ...
Selecting previously unselected package libblas3:amd64.
Preparing to unpack .../105-libblas3_3.10.0-2ubuntu1_amd64.deb ...
Unpacking libblas3:amd64 (3.10.0-2ubuntu1) ...
Selecting previously unselected package libgfortran5:amd64.
Preparing to unpack .../106-libgfortran5_12.3.0-1ubuntu1~22.04_amd64.deb ...
Unpacking libgfortran5:amd64 (12.3.0-1ubuntu1~22.04) ...
Selecting previously unselected package liblapack3:amd64.
Preparing to unpack .../107-liblapack3_3.10.0-2ubuntu1_amd64.deb ...
Unpacking liblapack3:amd64 (3.10.0-2ubuntu1) ...
Selecting previously unselected package libasyncns0:amd64.
Preparing to unpack .../108-libasyncns0_0.8-6build2_amd64.deb ...
Unpacking libasyncns0:amd64 (0.8-6build2) ...
Selecting previously unselected package libflac8:amd64.
Preparing to unpack .../109-libflac8_1.3.3-2ubuntu0.2_amd64.deb ...
Unpacking libflac8:amd64 (1.3.3-2ubuntu0.2) ...
Selecting previously unselected package libsndfile1:amd64.
Preparing to unpack .../110-libsndfile1_1.0.31-2ubuntu0.2_amd64.deb ...
Unpacking libsndfile1:amd64 (1.0.31-2ubuntu0.2) ...
Selecting previously unselected package libpulse0:amd64.
Preparing to unpack .../111-libpulse0_1%3a15.99.1+dfsg1-1ubuntu2.2_amd64.deb ...
Unpacking libpulse0:amd64 (1:15.99.1+dfsg1-1ubuntu2.2) ...
Selecting previously unselected package libsphinxbase3:amd64.
Preparing to unpack .../112-libsphinxbase3_0.8+5prealpha+1-13build1_amd64.deb ...
Unpacking libsphinxbase3:amd64 (0.8+5prealpha+1-13build1) ...
Selecting previously unselected package libpocketsphinx3:amd64.
Preparing to unpack .../113-libpocketsphinx3_0.8.0+real5prealpha+1-14ubuntu1_amd64.deb ...
Unpacking libpocketsphinx3:amd64 (0.8.0+real5prealpha+1-14ubuntu1) ...
Selecting previously unselected package libpostproc55:amd64.
Preparing to unpack .../114-libpostproc55_7%3a4.4.2-0ubuntu0.22.04.1_amd64.deb ...
Unpacking libpostproc55:amd64 (7:4.4.2-0ubuntu0.22.04.1) ...
Selecting previously unselected package libsamplerate0:amd64.
Preparing to unpack .../115-libsamplerate0_0.2.2-1build1_amd64.deb ...
Unpacking libsamplerate0:amd64 (0.2.2-1build1) ...
Selecting previously unselected package librubberband2:amd64.
Preparing to unpack .../116-librubberband2_2.0.0-2_amd64.deb ...
Unpacking librubberband2:amd64 (2.0.0-2) ...
Selecting previously unselected package libswscale5:amd64.
Preparing to unpack .../117-libswscale5_7%3a4.4.2-0ubuntu0.22.04.1_amd64.deb ...
Unpacking libswscale5:amd64 (7:4.4.2-0ubuntu0.22.04.1) ...
Selecting previously unselected package libvidstab1.1:amd64.
Preparing to unpack .../118-libvidstab1.1_1.1.0-2_amd64.deb ...
Unpacking libvidstab1.1:amd64 (1.1.0-2) ...
Selecting previously unselected package libzimg2:amd64.
Preparing to unpack .../119-libzimg2_3.0.3+ds1-1_amd64.deb ...
Unpacking libzimg2:amd64 (3.0.3+ds1-1) ...
Selecting previously unselected package libavfilter7:amd64.
Preparing to unpack .../120-libavfilter7_7%3a4.4.2-0ubuntu0.22.04.1_amd64.deb ...
Unpacking libavfilter7:amd64 (7:4.4.2-0ubuntu0.22.04.1) ...
Selecting previously unselected package libcaca0:amd64.
Preparing to unpack .../121-libcaca0_0.99.beta19-2.2ubuntu4_amd64.deb ...
Unpacking libcaca0:amd64 (0.99.beta19-2.2ubuntu4) ...
Selecting previously unselected package libcdio19:amd64.
Preparing to unpack .../122-libcdio19_2.1.0-3ubuntu0.2_amd64.deb ...
Unpacking libcdio19:amd64 (2.1.0-3ubuntu0.2) ...
Selecting previously unselected package libcdio-cdda2:amd64.
Preparing to unpack .../123-libcdio-cdda2_10.2+2.0.0-1build3_amd64.deb ...
Unpacking libcdio-cdda2:amd64 (10.2+2.0.0-1build3) ...
Selecting previously unselected package libcdio-paranoia2:amd64.
Preparing to unpack .../124-libcdio-paranoia2_10.2+2.0.0-1build3_amd64.deb ...
Unpacking libcdio-paranoia2:amd64 (10.2+2.0.0-1build3) ...
Selecting previously unselected package libdc1394-25:amd64.
Preparing to unpack .../125-libdc1394-25_2.2.6-4_amd64.deb ...
Unpacking libdc1394-25:amd64 (2.2.6-4) ...
Selecting previously unselected package libiec61883-0:amd64.
Preparing to unpack .../126-libiec61883-0_1.2.0-4build3_amd64.deb ...
Unpacking libiec61883-0:amd64 (1.2.0-4build3) ...
Selecting previously unselected package libjack-jackd2-0:amd64.
Preparing to unpack .../127-libjack-jackd2-0_1.9.20~dfsg-1_amd64.deb ...
Unpacking libjack-jackd2-0:amd64 (1.9.20~dfsg-1) ...
Selecting previously unselected package libopenal-data.
Preparing to unpack .../128-libopenal-data_1%3a1.19.1-2build3_all.deb ...
Unpacking libopenal-data (1:1.19.1-2build3) ...
Selecting previously unselected package libsndio7.0:amd64.
Preparing to unpack .../129-libsndio7.0_1.8.1-1.1_amd64.deb ...
Unpacking libsndio7.0:amd64 (1.8.1-1.1) ...
Selecting previously unselected package libopenal1:amd64.
Preparing to unpack .../130-libopenal1_1%3a1.19.1-2build3_amd64.deb ...
Unpacking libopenal1:amd64 (1:1.19.1-2build3) ...
Selecting previously unselected package libdecor-0-0:amd64.
Preparing to unpack .../131-libdecor-0-0_0.1.0-3build1_amd64.deb ...
Unpacking libdecor-0-0:amd64 (0.1.0-3build1) ...
Selecting previously unselected package libxss1:amd64.
Preparing to unpack .../132-libxss1_1%3a1.2.3-1build2_amd64.deb ...
Unpacking libxss1:amd64 (1:1.2.3-1build2) ...
Selecting previously unselected package libsdl2-2.0-0:amd64.
Preparing to unpack .../133-libsdl2-2.0-0_2.0.20+dfsg-2ubuntu1.22.04.1_amd64.deb ...
Unpacking libsdl2-2.0-0:amd64 (2.0.20+dfsg-2ubuntu1.22.04.1) ...
Selecting previously unselected package libxcb-shape0:amd64.
Preparing to unpack .../134-libxcb-shape0_1.14-3ubuntu3_amd64.deb ...
Unpacking libxcb-shape0:amd64 (1.14-3ubuntu3) ...
Selecting previously unselected package libxv1:amd64.
Preparing to unpack .../135-libxv1_2%3a1.0.11-1build2_amd64.deb ...
Unpacking libxv1:amd64 (2:1.0.11-1build2) ...
Selecting previously unselected package libavdevice58:amd64.
Preparing to unpack .../136-libavdevice58_7%3a4.4.2-0ubuntu0.22.04.1_amd64.deb ...
Unpacking libavdevice58:amd64 (7:4.4.2-0ubuntu0.22.04.1) ...
Selecting previously unselected package ffmpeg.
Preparing to unpack .../137-ffmpeg_7%3a4.4.2-0ubuntu0.22.04.1_amd64.deb ...
Unpacking ffmpeg (7:4.4.2-0ubuntu0.22.04.1) ...
Selecting previously unselected package libc6-i386.
Preparing to unpack .../138-libc6-i386_2.35-0ubuntu3.9_amd64.deb ...
Unpacking libc6-i386 (2.35-0ubuntu3.9) ...
Selecting previously unselected package libc6-dev-i386.
Preparing to unpack .../139-libc6-dev-i386_2.35-0ubuntu3.9_amd64.deb ...
Unpacking libc6-dev-i386 (2.35-0ubuntu3.9) ...
Selecting previously unselected package libc6-x32.
Preparing to unpack .../140-libc6-x32_2.35-0ubuntu3.9_amd64.deb ...
Unpacking libc6-x32 (2.35-0ubuntu3.9) ...
Selecting previously unselected package libc6-dev-x32.
Preparing to unpack .../141-libc6-dev-x32_2.35-0ubuntu3.9_amd64.deb ...
Unpacking libc6-dev-x32 (2.35-0ubuntu3.9) ...
Selecting previously unselected package lib32gcc-s1.
Preparing to unpack .../142-lib32gcc-s1_12.3.0-1ubuntu1~22.04_amd64.deb ...
Unpacking lib32gcc-s1 (12.3.0-1ubuntu1~22.04) ...
Selecting previously unselected package libx32gcc-s1.
Preparing to unpack .../143-libx32gcc-s1_12.3.0-1ubuntu1~22.04_amd64.deb ...
Unpacking libx32gcc-s1 (12.3.0-1ubuntu1~22.04) ...
Selecting previously unselected package lib32gomp1.
Preparing to unpack .../144-lib32gomp1_12.3.0-1ubuntu1~22.04_amd64.deb ...
Unpacking lib32gomp1 (12.3.0-1ubuntu1~22.04) ...
Selecting previously unselected package libx32gomp1.
Preparing to unpack .../145-libx32gomp1_12.3.0-1ubuntu1~22.04_amd64.deb ...
Unpacking libx32gomp1 (12.3.0-1ubuntu1~22.04) ...
Selecting previously unselected package lib32itm1.
Preparing to unpack .../146-lib32itm1_12.3.0-1ubuntu1~22.04_amd64.deb ...
Unpacking lib32itm1 (12.3.0-1ubuntu1~22.04) ...
Selecting previously unselected package libx32itm1.
Preparing to unpack .../147-libx32itm1_12.3.0-1ubuntu1~22.04_amd64.deb ...
Unpacking libx32itm1 (12.3.0-1ubuntu1~22.04) ...
Selecting previously unselected package lib32atomic1.
Preparing to unpack .../148-lib32atomic1_12.3.0-1ubuntu1~22.04_amd64.deb ...
Unpacking lib32atomic1 (12.3.0-1ubuntu1~22.04) ...
Selecting previously unselected package libx32atomic1.
Preparing to unpack .../149-libx32atomic1_12.3.0-1ubuntu1~22.04_amd64.deb ...
Unpacking libx32atomic1 (12.3.0-1ubuntu1~22.04) ...
Selecting previously unselected package lib32asan6.
Preparing to unpack .../150-lib32asan6_11.4.0-1ubuntu1~22.04_amd64.deb ...
Unpacking lib32asan6 (11.4.0-1ubuntu1~22.04) ...
Selecting previously unselected package libx32asan6.
Preparing to unpack .../151-libx32asan6_11.4.0-1ubuntu1~22.04_amd64.deb ...
Unpacking libx32asan6 (11.4.0-1ubuntu1~22.04) ...
Selecting previously unselected package lib32stdc++6.
Preparing to unpack .../152-lib32stdc++6_12.3.0-1ubuntu1~22.04_amd64.deb ...
Unpacking lib32stdc++6 (12.3.0-1ubuntu1~22.04) ...
Selecting previously unselected package lib32ubsan1.
Preparing to unpack .../153-lib32ubsan1_12.3.0-1ubuntu1~22.04_amd64.deb ...
Unpacking lib32ubsan1 (12.3.0-1ubuntu1~22.04) ...
Selecting previously unselected package libx32stdc++6.
Preparing to unpack .../154-libx32stdc++6_12.3.0-1ubuntu1~22.04_amd64.deb ...
Unpacking libx32stdc++6 (12.3.0-1ubuntu1~22.04) ...
Selecting previously unselected package libx32ubsan1.
Preparing to unpack .../155-libx32ubsan1_12.3.0-1ubuntu1~22.04_amd64.deb ...
Unpacking libx32ubsan1 (12.3.0-1ubuntu1~22.04) ...
Selecting previously unselected package lib32quadmath0.
Preparing to unpack .../156-lib32quadmath0_12.3.0-1ubuntu1~22.04_amd64.deb ...
Unpacking lib32quadmath0 (12.3.0-1ubuntu1~22.04) ...
Selecting previously unselected package libx32quadmath0.
Preparing to unpack .../157-libx32quadmath0_12.3.0-1ubuntu1~22.04_amd64.deb ...
Unpacking libx32quadmath0 (12.3.0-1ubuntu1~22.04) ...
Selecting previously unselected package lib32gcc-11-dev.
Preparing to unpack .../158-lib32gcc-11-dev_11.4.0-1ubuntu1~22.04_amd64.deb ...
Unpacking lib32gcc-11-dev (11.4.0-1ubuntu1~22.04) ...
Selecting previously unselected package libx32gcc-11-dev.
Preparing to unpack .../159-libx32gcc-11-dev_11.4.0-1ubuntu1~22.04_amd64.deb ...
Unpacking libx32gcc-11-dev (11.4.0-1ubuntu1~22.04) ...
Selecting previously unselected package gcc-11-multilib.
Preparing to unpack .../160-gcc-11-multilib_11.4.0-1ubuntu1~22.04_amd64.deb ...
Unpacking gcc-11-multilib (11.4.0-1ubuntu1~22.04) ...
Selecting previously unselected package lib32stdc++-11-dev.
Preparing to unpack .../161-lib32stdc++-11-dev_11.4.0-1ubuntu1~22.04_amd64.deb ...
Unpacking lib32stdc++-11-dev (11.4.0-1ubuntu1~22.04) ...
Selecting previously unselected package libx32stdc++-11-dev.
Preparing to unpack .../162-libx32stdc++-11-dev_11.4.0-1ubuntu1~22.04_amd64.deb ...
Unpacking libx32stdc++-11-dev (11.4.0-1ubuntu1~22.04) ...
Selecting previously unselected package g++-11-multilib.
Preparing to unpack .../163-g++-11-multilib_11.4.0-1ubuntu1~22.04_amd64.deb ...
Unpacking g++-11-multilib (11.4.0-1ubuntu1~22.04) ...
Selecting previously unselected package gcc-multilib.
Preparing to unpack .../164-gcc-multilib_4%3a11.2.0-1ubuntu1_amd64.deb ...
Unpacking gcc-multilib (4:11.2.0-1ubuntu1) ...
Selecting previously unselected package g++-multilib.
Preparing to unpack .../165-g++-multilib_4%3a11.2.0-1ubuntu1_amd64.deb ...
Unpacking g++-multilib (4:11.2.0-1ubuntu1) ...
Selecting previously unselected package libbabeltrace1:amd64.
Preparing to unpack .../166-libbabeltrace1_1.5.8-2build1_amd64.deb ...
Unpacking libbabeltrace1:amd64 (1.5.8-2build1) ...
Selecting previously unselected package libdebuginfod1:amd64.
Preparing to unpack .../167-libdebuginfod1_0.186-1ubuntu0.1_amd64.deb ...
Unpacking libdebuginfod1:amd64 (0.186-1ubuntu0.1) ...
Selecting previously unselected package libipt2.
Preparing to unpack .../168-libipt2_2.0.5-1_amd64.deb ...
Unpacking libipt2 (2.0.5-1) ...
Selecting previously unselected package libsource-highlight-common.
Preparing to unpack .../169-libsource-highlight-common_3.1.9-4.1build2_all.deb ...
Unpacking libsource-highlight-common (3.1.9-4.1build2) ...
Selecting previously unselected package libboost-regex1.74.0:amd64.
Preparing to unpack .../170-libboost-regex1.74.0_1.74.0-14ubuntu3_amd64.deb ...
Unpacking libboost-regex1.74.0:amd64 (1.74.0-14ubuntu3) ...
Selecting previously unselected package libsource-highlight4v5.
Preparing to unpack .../171-libsource-highlight4v5_3.1.9-4.1build2_amd64.deb ...
Unpacking libsource-highlight4v5 (3.1.9-4.1build2) ...
Selecting previously unselected package gdb.
Preparing to unpack .../172-gdb_12.1-0ubuntu1~22.04.2_amd64.deb ...
Unpacking gdb (12.1-0ubuntu1~22.04.2) ...
Selecting previously unselected package half.
Preparing to unpack .../173-half_1.12.0.60304-76~22.04_amd64.deb ...
Unpacking half (1.12.0.60304-76~22.04) ...
Selecting previously unselected package libfile-copy-recursive-perl.
Preparing to unpack .../174-libfile-copy-recursive-perl_0.45-1_all.deb ...
Unpacking libfile-copy-recursive-perl (0.45-1) ...
Selecting previously unselected package libtimedate-perl.
Preparing to unpack .../175-libtimedate-perl_2.3300-2_all.deb ...
Unpacking libtimedate-perl (2.3300-2) ...
Selecting previously unselected package libhttp-date-perl.
Preparing to unpack .../176-libhttp-date-perl_6.05-1_all.deb ...
Unpacking libhttp-date-perl (6.05-1) ...
Selecting previously unselected package libfile-listing-perl.
Preparing to unpack .../177-libfile-listing-perl_6.14-1_all.deb ...
Unpacking libfile-listing-perl (6.14-1) ...
Selecting previously unselected package libfile-which-perl.
Preparing to unpack .../178-libfile-which-perl_1.23-1_all.deb ...
Unpacking libfile-which-perl (1.23-1) ...
Selecting previously unselected package liburi-perl.
Preparing to unpack .../179-liburi-perl_5.10-1_all.deb ...
Unpacking liburi-perl (5.10-1) ...
Selecting previously unselected package hsa-runtime-rocr4wsl-amdgpu:amd64.
Preparing to unpack .../180-hsa-runtime-rocr4wsl-amdgpu_24.30-2127960.22.04_amd64.deb ...
Unpacking hsa-runtime-rocr4wsl-amdgpu:amd64 (24.30-2127960.22.04) ...
Selecting previously unselected package rocminfo.
Preparing to unpack .../181-rocminfo_1.0.0.60304-76~22.04_amd64.deb ...
Unpacking rocminfo (1.0.0.60304-76~22.04) ...
Selecting previously unselected package rocprofiler-register.
Preparing to unpack .../182-rocprofiler-register_0.4.0.60304-76~22.04_amd64.deb ...
Unpacking rocprofiler-register (0.4.0.60304-76~22.04) ...
Selecting previously unselected package hip-runtime-amd.
Preparing to unpack .../183-hip-runtime-amd_6.3.42134.60304-76~22.04_amd64.deb ...
Unpacking hip-runtime-amd (6.3.42134.60304-76~22.04) ...
Selecting previously unselected package rocm-llvm.
Preparing to unpack .../184-rocm-llvm_18.0.0.25012.60304-76~22.04_amd64.deb ...
Unpacking rocm-llvm (18.0.0.25012.60304-76~22.04) ...
Selecting previously unselected package libdrm2-amdgpu:amd64.
Preparing to unpack .../185-libdrm2-amdgpu_1%3a2.4.123.60304-2125197.22.04_amd64.deb ...
Unpacking libdrm2-amdgpu:amd64 (1:2.4.123.60304-2125197.22.04) ...
Selecting previously unselected package libdrm-amdgpu-common.
Preparing to unpack .../186-libdrm-amdgpu-common_1.0.0.60304-2125197.22.04_all.deb ...
Unpacking libdrm-amdgpu-common (1.0.0.60304-2125197.22.04) ...
Selecting previously unselected package libdrm-amdgpu-amdgpu1:amd64.
Preparing to unpack .../187-libdrm-amdgpu-amdgpu1_1%3a2.4.123.60304-2125197.22.04_amd64.deb ...
Unpacking libdrm-amdgpu-amdgpu1:amd64 (1:2.4.123.60304-2125197.22.04) ...
Selecting previously unselected package libdrm-amdgpu-radeon1:amd64.
Preparing to unpack .../188-libdrm-amdgpu-radeon1_1%3a2.4.123.60304-2125197.22.04_amd64.deb ...
Unpacking libdrm-amdgpu-radeon1:amd64 (1:2.4.123.60304-2125197.22.04) ...
Selecting previously unselected package libc6-dbg:amd64.
Preparing to unpack .../189-libc6-dbg_2.35-0ubuntu3.9_amd64.deb ...
Unpacking libc6-dbg:amd64 (2.35-0ubuntu3.9) ...
Selecting previously unselected package valgrind.
Preparing to unpack .../190-valgrind_1%3a3.18.1-1ubuntu2_amd64.deb ...
Unpacking valgrind (1:3.18.1-1ubuntu2) ...
Selecting previously unselected package libdrm-amdgpu-dev:amd64.
Preparing to unpack .../191-libdrm-amdgpu-dev_1%3a2.4.123.60304-2125197.22.04_amd64.deb ...
Unpacking libdrm-amdgpu-dev:amd64 (1:2.4.123.60304-2125197.22.04) ...
Selecting previously unselected package libpciaccess-dev:amd64.
Preparing to unpack .../192-libpciaccess-dev_0.16-3_amd64.deb ...
Unpacking libpciaccess-dev:amd64 (0.16-3) ...
Selecting previously unselected package libdrm-dev:amd64.
Preparing to unpack .../193-libdrm-dev_2.4.113-2~ubuntu0.22.04.1_amd64.deb ...
Unpacking libdrm-dev:amd64 (2.4.113-2~ubuntu0.22.04.1) ...
Selecting previously unselected package hsa-rocr-dev.
Preparing to unpack .../194-hsa-rocr-dev_1.14.0.60304-76~22.04_amd64.deb ...
Unpacking hsa-rocr-dev (1.14.0.60304-76~22.04) ...
Selecting previously unselected package hip-dev.
Preparing to unpack .../195-hip-dev_6.3.42134.60304-76~22.04_amd64.deb ...
Unpacking hip-dev (6.3.42134.60304-76~22.04) ...
Selecting previously unselected package hip-doc.
Preparing to unpack .../196-hip-doc_6.3.42134.60304-76~22.04_amd64.deb ...
Unpacking hip-doc (6.3.42134.60304-76~22.04) ...
Selecting previously unselected package hipcc.
Preparing to unpack .../197-hipcc_1.1.1.60304-76~22.04_amd64.deb ...
Unpacking hipcc (1.1.1.60304-76~22.04) ...
Selecting previously unselected package hip-samples.
Preparing to unpack .../198-hip-samples_6.3.42134.60304-76~22.04_amd64.deb ...
Unpacking hip-samples (6.3.42134.60304-76~22.04) ...
Selecting previously unselected package hipblaslt.
Preparing to unpack .../199-hipblaslt_0.10.0.60304-76~22.04_amd64.deb ...
Unpacking hipblaslt (0.10.0.60304-76~22.04) ...
Selecting previously unselected package rocblas.
Preparing to unpack .../200-rocblas_4.3.0.60304-76~22.04_amd64.deb ...
Unpacking rocblas (4.3.0.60304-76~22.04) ...
Selecting previously unselected package rocsolver.
Preparing to unpack .../201-rocsolver_3.27.0.60304-76~22.04_amd64.deb ...
Unpacking rocsolver (3.27.0.60304-76~22.04) ...
Selecting previously unselected package hipblas.
Preparing to unpack .../202-hipblas_2.3.0.60304-76~22.04_amd64.deb ...
Unpacking hipblas (2.3.0.60304-76~22.04) ...
Selecting previously unselected package hipblas-common-dev.
Preparing to unpack .../203-hipblas-common-dev_1.0.0.60304-76~22.04_amd64.deb ...
Unpacking hipblas-common-dev (1.0.0.60304-76~22.04) ...
Selecting previously unselected package hipblas-dev.
Preparing to unpack .../204-hipblas-dev_2.3.0.60304-76~22.04_amd64.deb ...
Unpacking hipblas-dev (2.3.0.60304-76~22.04) ...
Selecting previously unselected package hipblaslt-dev.
Preparing to unpack .../205-hipblaslt-dev_0.10.0.60304-76~22.04_amd64.deb ...
Unpacking hipblaslt-dev (0.10.0.60304-76~22.04) ...
Selecting previously unselected package rocprim-dev.
Preparing to unpack .../206-rocprim-dev_3.3.0.60304-76~22.04_amd64.deb ...
Unpacking rocprim-dev (3.3.0.60304-76~22.04) ...
Selecting previously unselected package hipcub-dev.
Preparing to unpack .../207-hipcub-dev_3.3.0.60304-76~22.04_amd64.deb ...
Unpacking hipcub-dev (3.3.0.60304-76~22.04) ...
Selecting previously unselected package rocfft.
Preparing to unpack .../208-rocfft_1.0.31.60304-76~22.04_amd64.deb ...
Unpacking rocfft (1.0.31.60304-76~22.04) ...
Selecting previously unselected package hipfft.
Preparing to unpack .../209-hipfft_1.0.17.60304-76~22.04_amd64.deb ...
Unpacking hipfft (1.0.17.60304-76~22.04) ...
Selecting previously unselected package hipfft-dev.
Preparing to unpack .../210-hipfft-dev_1.0.17.60304-76~22.04_amd64.deb ...
Unpacking hipfft-dev (1.0.17.60304-76~22.04) ...
Selecting previously unselected package hipfort-dev.
Preparing to unpack .../211-hipfort-dev_0.5.1.60304-76~22.04_amd64.deb ...
Unpacking hipfort-dev (0.5.1.60304-76~22.04) ...
Selecting previously unselected package hipify-clang.
Preparing to unpack .../212-hipify-clang_18.0.0.60304-76~22.04_amd64.deb ...
Unpacking hipify-clang (18.0.0.60304-76~22.04) ...
Selecting previously unselected package hiprand.
Preparing to unpack .../213-hiprand_2.11.1.60304-76~22.04_amd64.deb ...
Unpacking hiprand (2.11.1.60304-76~22.04) ...
Selecting previously unselected package hiprand-dev.
Preparing to unpack .../214-hiprand-dev_2.11.1.60304-76~22.04_amd64.deb ...
Unpacking hiprand-dev (2.11.1.60304-76~22.04) ...
Selecting previously unselected package hipsolver.
Preparing to unpack .../215-hipsolver_2.3.0.60304-76~22.04_amd64.deb ...
Unpacking hipsolver (2.3.0.60304-76~22.04) ...
Selecting previously unselected package hipsolver-dev.
Preparing to unpack .../216-hipsolver-dev_2.3.0.60304-76~22.04_amd64.deb ...
Unpacking hipsolver-dev (2.3.0.60304-76~22.04) ...
Selecting previously unselected package rocsparse.
Preparing to unpack .../217-rocsparse_3.3.0.60304-76~22.04_amd64.deb ...
Unpacking rocsparse (3.3.0.60304-76~22.04) ...
Selecting previously unselected package hipsparse.
Preparing to unpack .../218-hipsparse_3.1.2.60304-76~22.04_amd64.deb ...
Unpacking hipsparse (3.1.2.60304-76~22.04) ...
Selecting previously unselected package hipsparse-dev.
Preparing to unpack .../219-hipsparse-dev_3.1.2.60304-76~22.04_amd64.deb ...
Unpacking hipsparse-dev (3.1.2.60304-76~22.04) ...
Selecting previously unselected package hipsparselt.
Preparing to unpack .../220-hipsparselt_0.2.2.60304-76~22.04_amd64.deb ...
Unpacking hipsparselt (0.2.2.60304-76~22.04) ...
Selecting previously unselected package hipsparselt-dev.
Preparing to unpack .../221-hipsparselt-dev_0.2.2.60304-76~22.04_amd64.deb ...
Unpacking hipsparselt-dev (0.2.2.60304-76~22.04) ...
Selecting previously unselected package hiptensor.
Preparing to unpack .../222-hiptensor_1.4.0.60304-76~22.04_amd64.deb ...
Unpacking hiptensor (1.4.0.60304-76~22.04) ...
Selecting previously unselected package hiptensor-dev.
Preparing to unpack .../223-hiptensor-dev_1.4.0.60304-76~22.04_amd64.deb ...
Unpacking hiptensor-dev (1.4.0.60304-76~22.04) ...
Selecting previously unselected package hsa-amd-aqlprofile.
Preparing to unpack .../224-hsa-amd-aqlprofile_1.0.0.60304-76~22.04_amd64.deb ...
Unpacking hsa-amd-aqlprofile (1.0.0.60304-76~22.04) ...
Selecting previously unselected package icu-devtools.
Preparing to unpack .../225-icu-devtools_70.1-2_amd64.deb ...
Unpacking icu-devtools (70.1-2) ...
Selecting previously unselected package libigdgmm12:amd64.
Preparing to unpack .../226-libigdgmm12_22.1.2+ds1-1_amd64.deb ...
Unpacking libigdgmm12:amd64 (22.1.2+ds1-1) ...
Selecting previously unselected package intel-media-va-driver:amd64.
Preparing to unpack .../227-intel-media-va-driver_22.3.1+dfsg1-1ubuntu2_amd64.deb ...
Unpacking intel-media-va-driver:amd64 (22.3.1+dfsg1-1ubuntu2) ...
Selecting previously unselected package javascript-common.
Preparing to unpack .../228-javascript-common_11+nmu1_all.deb ...
Unpacking javascript-common (11+nmu1) ...
Selecting previously unselected package libaacs0:amd64.
Preparing to unpack .../229-libaacs0_0.11.1-1_amd64.deb ...
Unpacking libaacs0:amd64 (0.11.1-1) ...
Selecting previously unselected package libalgorithm-diff-perl.
Preparing to unpack .../230-libalgorithm-diff-perl_1.201-1_all.deb ...
Unpacking libalgorithm-diff-perl (1.201-1) ...
Selecting previously unselected package libalgorithm-diff-xs-perl.
Preparing to unpack .../231-libalgorithm-diff-xs-perl_0.04-6build3_amd64.deb ...
Unpacking libalgorithm-diff-xs-perl (0.04-6build3) ...
Selecting previously unselected package libalgorithm-merge-perl.
Preparing to unpack .../232-libalgorithm-merge-perl_0.08-3_all.deb ...
Unpacking libalgorithm-merge-perl (0.08-3) ...
Selecting previously unselected package libsuitesparseconfig5:amd64.
Preparing to unpack .../233-libsuitesparseconfig5_1%3a5.10.1+dfsg-4build1_amd64.deb ...
Unpacking libsuitesparseconfig5:amd64 (1:5.10.1+dfsg-4build1) ...
Selecting previously unselected package libamd2:amd64.
Preparing to unpack .../234-libamd2_1%3a5.10.1+dfsg-4build1_amd64.deb ...
Unpacking libamd2:amd64 (1:5.10.1+dfsg-4build1) ...
Selecting previously unselected package libavutil-dev:amd64.
Preparing to unpack .../235-libavutil-dev_7%3a4.4.2-0ubuntu0.22.04.1_amd64.deb ...
Unpacking libavutil-dev:amd64 (7:4.4.2-0ubuntu0.22.04.1) ...
Selecting previously unselected package libswresample-dev:amd64.
Preparing to unpack .../236-libswresample-dev_7%3a4.4.2-0ubuntu0.22.04.1_amd64.deb ...
Unpacking libswresample-dev:amd64 (7:4.4.2-0ubuntu0.22.04.1) ...
Selecting previously unselected package libavcodec-dev:amd64.
Preparing to unpack .../237-libavcodec-dev_7%3a4.4.2-0ubuntu0.22.04.1_amd64.deb ...
Unpacking libavcodec-dev:amd64 (7:4.4.2-0ubuntu0.22.04.1) ...
Selecting previously unselected package libavformat-dev:amd64.
Preparing to unpack .../238-libavformat-dev_7%3a4.4.2-0ubuntu0.22.04.1_amd64.deb ...
Unpacking libavformat-dev:amd64 (7:4.4.2-0ubuntu0.22.04.1) ...
Selecting previously unselected package libbdplus0:amd64.
Preparing to unpack .../239-libbdplus0_0.2.0-1_amd64.deb ...
Unpacking libbdplus0:amd64 (0.2.0-1) ...
Selecting previously unselected package libxpm4:amd64.
Preparing to unpack .../240-libxpm4_1%3a3.5.12-1ubuntu0.22.04.2_amd64.deb ...
Unpacking libxpm4:amd64 (1:3.5.12-1ubuntu0.22.04.2) ...
Selecting previously unselected package libgd3:amd64.
Preparing to unpack .../241-libgd3_2.3.0-2ubuntu2.3_amd64.deb ...
Unpacking libgd3:amd64 (2.3.0-2ubuntu2.3) ...
Selecting previously unselected package libc-devtools.
Preparing to unpack .../242-libc-devtools_2.35-0ubuntu3.9_amd64.deb ...
Unpacking libc-devtools (2.35-0ubuntu3.9) ...
Selecting previously unselected package libcamd2:amd64.
Preparing to unpack .../243-libcamd2_1%3a5.10.1+dfsg-4build1_amd64.deb ...
Unpacking libcamd2:amd64 (1:5.10.1+dfsg-4build1) ...
Selecting previously unselected package libccolamd2:amd64.
Preparing to unpack .../244-libccolamd2_1%3a5.10.1+dfsg-4build1_amd64.deb ...
Unpacking libccolamd2:amd64 (1:5.10.1+dfsg-4build1) ...
Selecting previously unselected package libcolamd2:amd64.
Preparing to unpack .../245-libcolamd2_1%3a5.10.1+dfsg-4build1_amd64.deb ...
Unpacking libcolamd2:amd64 (1:5.10.1+dfsg-4build1) ...
Selecting previously unselected package libmetis5:amd64.
Preparing to unpack .../246-libmetis5_5.1.0.dfsg-7build2_amd64.deb ...
Unpacking libmetis5:amd64 (5.1.0.dfsg-7build2) ...
Selecting previously unselected package libcholmod3:amd64.
Preparing to unpack .../247-libcholmod3_1%3a5.10.1+dfsg-4build1_amd64.deb ...
Unpacking libcholmod3:amd64 (1:5.10.1+dfsg-4build1) ...
Selecting previously unselected package libdecor-0-plugin-1-cairo:amd64.
Preparing to unpack .../248-libdecor-0-plugin-1-cairo_0.1.0-3build1_amd64.deb ...
Unpacking libdecor-0-plugin-1-cairo:amd64 (0.1.0-3build1) ...
Selecting previously unselected package zlib1g-dev:amd64.
Preparing to unpack .../249-zlib1g-dev_1%3a1.2.11.dfsg-2ubuntu9.2_amd64.deb ...
Unpacking zlib1g-dev:amd64 (1:1.2.11.dfsg-2ubuntu9.2) ...
Selecting previously unselected package libelf-dev:amd64.
Preparing to unpack .../250-libelf-dev_0.186-1ubuntu0.1_amd64.deb ...
Unpacking libelf-dev:amd64 (0.186-1ubuntu0.1) ...
Selecting previously unselected package libexpat1-dev:amd64.
Preparing to unpack .../251-libexpat1-dev_2.4.7-1ubuntu0.6_amd64.deb ...
Unpacking libexpat1-dev:amd64 (2.4.7-1ubuntu0.6) ...
Selecting previously unselected package libfile-fcntllock-perl.
Preparing to unpack .../252-libfile-fcntllock-perl_0.22-3build7_amd64.deb ...
Unpacking libfile-fcntllock-perl (0.22-3build7) ...
Selecting previously unselected package xorg-sgml-doctools.
Preparing to unpack .../253-xorg-sgml-doctools_1%3a1.11-1.1_all.deb ...
Unpacking xorg-sgml-doctools (1:1.11-1.1) ...
Selecting previously unselected package x11proto-dev.
Preparing to unpack .../254-x11proto-dev_2021.5-1_all.deb ...
Unpacking x11proto-dev (2021.5-1) ...
Selecting previously unselected package libxau-dev:amd64.
Preparing to unpack .../255-libxau-dev_1%3a1.0.9-1build5_amd64.deb ...
Unpacking libxau-dev:amd64 (1:1.0.9-1build5) ...
Selecting previously unselected package libxdmcp-dev:amd64.
Preparing to unpack .../256-libxdmcp-dev_1%3a1.1.3-0ubuntu5_amd64.deb ...
Unpacking libxdmcp-dev:amd64 (1:1.1.3-0ubuntu5) ...
Selecting previously unselected package xtrans-dev.
Preparing to unpack .../257-xtrans-dev_1.4.0-1_all.deb ...
Unpacking xtrans-dev (1.4.0-1) ...
Selecting previously unselected package libpthread-stubs0-dev:amd64.
Preparing to unpack .../258-libpthread-stubs0-dev_0.4-1build2_amd64.deb ...
Unpacking libpthread-stubs0-dev:amd64 (0.4-1build2) ...
Selecting previously unselected package libxcb1-dev:amd64.
Preparing to unpack .../259-libxcb1-dev_1.14-3ubuntu3_amd64.deb ...
Unpacking libxcb1-dev:amd64 (1.14-3ubuntu3) ...
Selecting previously unselected package libx11-dev:amd64.
Preparing to unpack .../260-libx11-dev_2%3a1.7.5-1ubuntu0.3_amd64.deb ...
Unpacking libx11-dev:amd64 (2:1.7.5-1ubuntu0.3) ...
Selecting previously unselected package libglx-dev:amd64.
Preparing to unpack .../261-libglx-dev_1.4.0-1_amd64.deb ...
Unpacking libglx-dev:amd64 (1.4.0-1) ...
Selecting previously unselected package libgl-dev:amd64.
Preparing to unpack .../262-libgl-dev_1.4.0-1_amd64.deb ...
Unpacking libgl-dev:amd64 (1.4.0-1) ...
Selecting previously unselected package libicu-dev:amd64.
Preparing to unpack .../263-libicu-dev_70.1-2_amd64.deb ...
Unpacking libicu-dev:amd64 (70.1-2) ...
Selecting previously unselected package libjs-jquery.
Preparing to unpack .../264-libjs-jquery_3.6.0+dfsg+~3.5.13-1_all.deb ...
Unpacking libjs-jquery (3.6.0+dfsg+~3.5.13-1) ...
Selecting previously unselected package libjs-underscore.
Preparing to unpack .../265-libjs-underscore_1.13.2~dfsg-2_all.deb ...
Unpacking libjs-underscore (1.13.2~dfsg-2) ...
Selecting previously unselected package libjs-sphinxdoc.
Preparing to unpack .../266-libjs-sphinxdoc_4.3.2-1_all.deb ...
Unpacking libjs-sphinxdoc (4.3.2-1) ...
Selecting previously unselected package libpython3.10-dev:amd64.
Preparing to unpack .../267-libpython3.10-dev_3.10.12-1~22.04.9_amd64.deb ...
Unpacking libpython3.10-dev:amd64 (3.10.12-1~22.04.9) ...
Selecting previously unselected package libpython3-dev:amd64.
Preparing to unpack .../268-libpython3-dev_3.10.6-1~22.04.1_amd64.deb ...
Unpacking libpython3-dev:amd64 (3.10.6-1~22.04.1) ...
Selecting previously unselected package libswscale-dev:amd64.
Preparing to unpack .../269-libswscale-dev_7%3a4.4.2-0ubuntu0.22.04.1_amd64.deb ...
Unpacking libswscale-dev:amd64 (7:4.4.2-0ubuntu0.22.04.1) ...
Selecting previously unselected package libxml2-dev:amd64.
Preparing to unpack .../270-libxml2-dev_2.9.13+dfsg-1ubuntu0.7_amd64.deb ...
Unpacking libxml2-dev:amd64 (2.9.13+dfsg-1ubuntu0.7) ...
Selecting previously unselected package manpages-dev.
Preparing to unpack .../271-manpages-dev_5.10-1ubuntu1_all.deb ...
Unpacking manpages-dev (5.10-1ubuntu1) ...
Selecting previously unselected package mesa-common-dev:amd64.
Preparing to unpack .../272-mesa-common-dev_23.2.1-1ubuntu3.1~22.04.3_amd64.deb ...
Unpacking mesa-common-dev:amd64 (23.2.1-1ubuntu3.1~22.04.3) ...
Selecting previously unselected package mesa-va-drivers:amd64.
Preparing to unpack .../273-mesa-va-drivers_23.2.1-1ubuntu3.1~22.04.3_amd64.deb ...
Unpacking mesa-va-drivers:amd64 (23.2.1-1ubuntu3.1~22.04.3) ...
Selecting previously unselected package mesa-vdpau-drivers:amd64.
Preparing to unpack .../274-mesa-vdpau-drivers_23.2.1-1ubuntu3.1~22.04.3_amd64.deb ...
Unpacking mesa-vdpau-drivers:amd64 (23.2.1-1ubuntu3.1~22.04.3) ...
Selecting previously unselected package roctracer.
Preparing to unpack .../275-roctracer_4.1.60304.60304-76~22.04_amd64.deb ...
Unpacking roctracer (4.1.60304.60304-76~22.04) ...
Selecting previously unselected package rocrand.
Preparing to unpack .../276-rocrand_3.2.0.60304-76~22.04_amd64.deb ...
Unpacking rocrand (3.2.0.60304-76~22.04) ...
Selecting previously unselected package miopen-hip.
Preparing to unpack .../277-miopen-hip_3.3.0.60304-76~22.04_amd64.deb ...
Unpacking miopen-hip (3.3.0.60304-76~22.04) ...
Selecting previously unselected package migraphx.
Preparing to unpack .../278-migraphx_2.11.0.60304-76~22.04_amd64.deb ...
Unpacking migraphx (2.11.0.60304-76~22.04) ...
Selecting previously unselected package migraphx-dev.
Preparing to unpack .../279-migraphx-dev_2.11.0.60304-76~22.04_amd64.deb ...
Unpacking migraphx-dev (2.11.0.60304-76~22.04) ...
Selecting previously unselected package miopen-hip-dev.
Preparing to unpack .../280-miopen-hip-dev_3.3.0.60304-76~22.04_amd64.deb ...
Unpacking miopen-hip-dev (3.3.0.60304-76~22.04) ...
Selecting previously unselected package openmp-extras-runtime.
Preparing to unpack .../281-openmp-extras-runtime_18.63.0.60304-76~22.04_amd64.deb ...
Unpacking openmp-extras-runtime (18.63.0.60304-76~22.04) ...
Selecting previously unselected package rocm-language-runtime.
Preparing to unpack .../282-rocm-language-runtime_6.3.4.60304-76~22.04_amd64.deb ...
Unpacking rocm-language-runtime (6.3.4.60304-76~22.04) ...
Selecting previously unselected package rocm-hip-runtime.
Preparing to unpack .../283-rocm-hip-runtime_6.3.4.60304-76~22.04_amd64.deb ...
Unpacking rocm-hip-runtime (6.3.4.60304-76~22.04) ...
Selecting previously unselected package rpp.
Preparing to unpack .../284-rpp_1.9.1.60304-76~22.04_amd64.deb ...
Unpacking rpp (1.9.1.60304-76~22.04) ...
Selecting previously unselected package mivisionx.
Preparing to unpack .../285-mivisionx_3.1.0.60304-76~22.04_amd64.deb ...
Unpacking mivisionx (3.1.0.60304-76~22.04) ...
Selecting previously unselected package rocm-device-libs.
Preparing to unpack .../286-rocm-device-libs_1.0.0.60304-76~22.04_amd64.deb ...
Unpacking rocm-device-libs (1.0.0.60304-76~22.04) ...
Selecting previously unselected package rocm-cmake.
Preparing to unpack .../287-rocm-cmake_0.14.0.60304-76~22.04_amd64.deb ...
Unpacking rocm-cmake (0.14.0.60304-76~22.04) ...
Selecting previously unselected package rocm-hip-runtime-dev.
Preparing to unpack .../288-rocm-hip-runtime-dev_6.3.4.60304-76~22.04_amd64.deb ...
Unpacking rocm-hip-runtime-dev (6.3.4.60304-76~22.04) ...
Selecting previously unselected package rpp-dev.
Preparing to unpack .../289-rpp-dev_1.9.1.60304-76~22.04_amd64.deb ...
Unpacking rpp-dev (1.9.1.60304-76~22.04) ...
Selecting previously unselected package rocblas-dev.
Preparing to unpack .../290-rocblas-dev_4.3.0.60304-76~22.04_amd64.deb ...
Unpacking rocblas-dev (4.3.0.60304-76~22.04) ...
Selecting previously unselected package mivisionx-dev.
Preparing to unpack .../291-mivisionx-dev_3.1.0.60304-76~22.04_amd64.deb ...
Unpacking mivisionx-dev (3.1.0.60304-76~22.04) ...
Selecting previously unselected package openmp-extras-dev.
Preparing to unpack .../292-openmp-extras-dev_18.63.0.60304-76~22.04_amd64.deb ...
Unpacking openmp-extras-dev (18.63.0.60304-76~22.04) ...
Selecting previously unselected package python3-argcomplete.
Preparing to unpack .../293-python3-argcomplete_1.8.1-1.5_all.deb ...
Unpacking python3-argcomplete (1.8.1-1.5) ...
Selecting previously unselected package python3.10-dev.
Preparing to unpack .../294-python3.10-dev_3.10.12-1~22.04.9_amd64.deb ...
Unpacking python3.10-dev (3.10.12-1~22.04.9) ...
Selecting previously unselected package python3-dev.
Preparing to unpack .../295-python3-dev_3.10.6-1~22.04.1_amd64.deb ...
Unpacking python3-dev (3.10.6-1~22.04.1) ...
Selecting previously unselected package rocm-smi-lib.
Preparing to unpack .../296-rocm-smi-lib_7.4.0.60304-76~22.04_amd64.deb ...
Unpacking rocm-smi-lib (7.4.0.60304-76~22.04) ...
Selecting previously unselected package rccl.
Preparing to unpack .../297-rccl_2.21.5.60304-76~22.04_amd64.deb ...
Unpacking rccl (2.21.5.60304-76~22.04) ...
Selecting previously unselected package rccl-dev.
Preparing to unpack .../298-rccl-dev_2.21.5.60304-76~22.04_amd64.deb ...
Unpacking rccl-dev (2.21.5.60304-76~22.04) ...
Selecting previously unselected package rocalution.
Preparing to unpack .../299-rocalution_3.2.1.60304-76~22.04_amd64.deb ...
Unpacking rocalution (3.2.1.60304-76~22.04) ...
Selecting previously unselected package rocalution-dev.
Preparing to unpack .../300-rocalution-dev_3.2.1.60304-76~22.04_amd64.deb ...
Unpacking rocalution-dev (3.2.1.60304-76~22.04) ...
Selecting previously unselected package rocfft-dev.
Preparing to unpack .../301-rocfft-dev_1.0.31.60304-76~22.04_amd64.deb ...
Unpacking rocfft-dev (1.0.31.60304-76~22.04) ...
Selecting previously unselected package rocm-utils.
Preparing to unpack .../302-rocm-utils_6.3.4.60304-76~22.04_amd64.deb ...
Unpacking rocm-utils (6.3.4.60304-76~22.04) ...
Selecting previously unselected package rocm-dbgapi.
Preparing to unpack .../303-rocm-dbgapi_0.77.0.60304-76~22.04_amd64.deb ...
Unpacking rocm-dbgapi (0.77.0.60304-76~22.04) ...
Selecting previously unselected package rocm-debug-agent.
Preparing to unpack .../304-rocm-debug-agent_2.0.3.60304-76~22.04_amd64.deb ...
Unpacking rocm-debug-agent (2.0.3.60304-76~22.04) ...
Selecting previously unselected package rocm-gdb.
Preparing to unpack .../305-rocm-gdb_15.2.60304-76~22.04_amd64.deb ...
Unpacking rocm-gdb (15.2.60304-76~22.04) ...
Selecting previously unselected package libnuma-dev:amd64.
Preparing to unpack .../306-libnuma-dev_2.0.14-3ubuntu2_amd64.deb ...
Unpacking libnuma-dev:amd64 (2.0.14-3ubuntu2) ...
Selecting previously unselected package rocprofiler.
Preparing to unpack .../307-rocprofiler_2.0.60304.60304-76~22.04_amd64.deb ...
Unpacking rocprofiler (2.0.60304.60304-76~22.04) ...
Selecting previously unselected package rocprofiler-plugins.
Preparing to unpack .../308-rocprofiler-plugins_2.0.60304.60304-76~22.04_amd64.deb ...
Unpacking rocprofiler-plugins (2.0.60304.60304-76~22.04) ...
Selecting previously unselected package rocprofiler-sdk-roctx.
Preparing to unpack .../309-rocprofiler-sdk-roctx_0.5.0-76~22.04_amd64.deb ...
Unpacking rocprofiler-sdk-roctx (0.5.0-76~22.04) ...
Selecting previously unselected package rocprofiler-sdk.
Preparing to unpack .../310-rocprofiler-sdk_0.5.0-76~22.04_amd64.deb ...
Unpacking rocprofiler-sdk (0.5.0-76~22.04) ...
Selecting previously unselected package rocprofiler-dev.
Preparing to unpack .../311-rocprofiler-dev_2.0.60304.60304-76~22.04_amd64.deb ...
Unpacking rocprofiler-dev (2.0.60304.60304-76~22.04) ...
Selecting previously unselected package roctracer-dev.
Preparing to unpack .../312-roctracer-dev_4.1.60304.60304-76~22.04_amd64.deb ...
Unpacking roctracer-dev (4.1.60304.60304-76~22.04) ...
Selecting previously unselected package rocm-developer-tools.
Preparing to unpack .../313-rocm-developer-tools_6.3.4.60304-76~22.04_amd64.deb ...
Unpacking rocm-developer-tools (6.3.4.60304-76~22.04) ...
Selecting previously unselected package rocm-openmp-sdk.
Preparing to unpack .../314-rocm-openmp-sdk_6.3.4.60304-76~22.04_amd64.deb ...
Unpacking rocm-openmp-sdk (6.3.4.60304-76~22.04) ...
Selecting previously unselected package rocm-opencl.
Preparing to unpack .../315-rocm-opencl_2.0.0.60304-76~22.04_amd64.deb ...
Unpacking rocm-opencl (2.0.0.60304-76~22.04) ...
Selecting previously unselected package rocm-opencl-runtime.
Preparing to unpack .../316-rocm-opencl-runtime_6.3.4.60304-76~22.04_amd64.deb ...
Unpacking rocm-opencl-runtime (6.3.4.60304-76~22.04) ...
Selecting previously unselected package rocm-opencl-dev.
Preparing to unpack .../317-rocm-opencl-dev_2.0.0.60304-76~22.04_amd64.deb ...
Unpacking rocm-opencl-dev (2.0.0.60304-76~22.04) ...
Selecting previously unselected package rocm-opencl-sdk.
Preparing to unpack .../318-rocm-opencl-sdk_6.3.4.60304-76~22.04_amd64.deb ...
Unpacking rocm-opencl-sdk (6.3.4.60304-76~22.04) ...
Selecting previously unselected package rocm-hip-libraries.
Preparing to unpack .../319-rocm-hip-libraries_6.3.4.60304-76~22.04_amd64.deb ...
Unpacking rocm-hip-libraries (6.3.4.60304-76~22.04) ...
Selecting previously unselected package rocm-ml-libraries.
Preparing to unpack .../320-rocm-ml-libraries_6.3.4.60304-76~22.04_amd64.deb ...
Unpacking rocm-ml-libraries (6.3.4.60304-76~22.04) ...
Selecting previously unselected package rocrand-dev.
Preparing to unpack .../321-rocrand-dev_3.2.0.60304-76~22.04_amd64.deb ...
Unpacking rocrand-dev (3.2.0.60304-76~22.04) ...
Selecting previously unselected package rocsolver-dev.
Preparing to unpack .../322-rocsolver-dev_3.27.0.60304-76~22.04_amd64.deb ...
Unpacking rocsolver-dev (3.27.0.60304-76~22.04) ...
Selecting previously unselected package rocsparse-dev.
Preparing to unpack .../323-rocsparse-dev_3.3.0.60304-76~22.04_amd64.deb ...
Unpacking rocsparse-dev (3.3.0.60304-76~22.04) ...
Selecting previously unselected package rocthrust-dev.
Preparing to unpack .../324-rocthrust-dev_3.3.0.60304-76~22.04_amd64.deb ...
Unpacking rocthrust-dev (3.3.0.60304-76~22.04) ...
Selecting previously unselected package rocwmma-dev.
Preparing to unpack .../325-rocwmma-dev_1.6.0.60304-76~22.04_amd64.deb ...
Unpacking rocwmma-dev (1.6.0.60304-76~22.04) ...
Selecting previously unselected package rocm-hip-sdk.
Preparing to unpack .../326-rocm-hip-sdk_6.3.4.60304-76~22.04_amd64.deb ...
Unpacking rocm-hip-sdk (6.3.4.60304-76~22.04) ...
Selecting previously unselected package rocm-ml-sdk.
Preparing to unpack .../327-rocm-ml-sdk_6.3.4.60304-76~22.04_amd64.deb ...
Unpacking rocm-ml-sdk (6.3.4.60304-76~22.04) ...
Selecting previously unselected package rocm.
Preparing to unpack .../328-rocm_6.3.4.60304-76~22.04_amd64.deb ...
Unpacking rocm (6.3.4.60304-76~22.04) ...
Selecting previously unselected package i965-va-driver:amd64.
Preparing to unpack .../329-i965-va-driver_2.4.1+dfsg1-1_amd64.deb ...
Unpacking i965-va-driver:amd64 (2.4.1+dfsg1-1) ...
Selecting previously unselected package va-driver-all:amd64.
Preparing to unpack .../330-va-driver-all_2.14.0-1_amd64.deb ...
Unpacking va-driver-all:amd64 (2.14.0-1) ...
Selecting previously unselected package vdpau-driver-all:amd64.
Preparing to unpack .../331-vdpau-driver-all_1.4-3build2_amd64.deb ...
Unpacking vdpau-driver-all:amd64 (1.4-3build2) ...
Selecting previously unselected package pocketsphinx-en-us.
Preparing to unpack .../332-pocketsphinx-en-us_0.8.0+real5prealpha+1-14ubuntu1_all.deb ...
Unpacking pocketsphinx-en-us (0.8.0+real5prealpha+1-14ubuntu1) ...
Setting up libgme0:amd64 (0.6.3-2) ...
Setting up libssh-gcrypt-4:amd64 (0.9.6-2ubuntu0.22.04.3) ...
Setting up javascript-common (11+nmu1) ...
Setting up libsrt1.4-gnutls:amd64 (1.4.4-4) ...
Setting up libudfread0:amd64 (1.1.2-1) ...
Setting up gcc-11-base:amd64 (11.4.0-1ubuntu1~22.04) ...
Setting up libaom3:amd64 (3.3.0-1ubuntu0.1) ...
Setting up manpages-dev (5.10-1ubuntu1) ...
Setting up libfile-which-perl (1.23-1) ...
Setting up librabbitmq4:amd64 (0.10.0-1ubuntu2) ...
Setting up libraw1394-11:amd64 (2.1.2-2build2) ...
Setting up lto-disabled-list (24) ...
Setting up libcodec2-1.0:amd64 (1.0.1-3) ...
Setting up libmpg123-0:amd64 (1.29.3-1ubuntu0.1) ...
Setting up libpciaccess-dev:amd64 (0.16-3) ...
Setting up libogg0:amd64 (1.3.5-0ubuntu3) ...
Setting up libspeex1:amd64 (1.2~rc1.2-1.1ubuntu3) ...
Setting up libshine3:amd64 (3.1.1-2) ...
Setting up libcaca0:amd64 (0.99.beta19-2.2ubuntu4) ...
Setting up libxpm4:amd64 (1:3.5.12-1ubuntu0.22.04.2) ...
Setting up libtwolame0:amd64 (0.4.0-2build2) ...
Setting up libdebuginfod-common (0.186-1ubuntu0.1) ...

Creating config file /etc/profile.d/debuginfod.sh with new version

Creating config file /etc/profile.d/debuginfod.csh with new version
Setting up libgsm1:amd64 (1.0.19-1) ...
Setting up libfile-fcntllock-perl (0.22-3build7) ...
Setting up libalgorithm-diff-perl (1.201-1) ...
Setting up libpgm-5.3-0:amd64 (5.3.128~dfsg-2) ...
Setting up libdebuginfod1:amd64 (0.186-1ubuntu0.1) ...
Setting up libnorm1:amd64 (1.5.9+dfsg-2) ...
Setting up libmysofa1:amd64 (1.2.1~dfsg0-1) ...
Setting up libxcb-shape0:amd64 (1.14-3ubuntu3) ...
Setting up linux-libc-dev:amd64 (5.15.0-140.150) ...
Setting up libmetis5:amd64 (5.1.0.dfsg-7build2) ...
Setting up libigdgmm12:amd64 (22.1.2+ds1-1) ...
Setting up libgomp1:amd64 (12.3.0-1ubuntu1~22.04) ...
Setting up libcdio19:amd64 (2.1.0-3ubuntu0.2) ...
Setting up libxvidcore4:amd64 (2:1.3.7-1) ...
Setting up bzip2 (1.0.8-5build1) ...
Setting up libpthread-stubs0-dev:amd64 (0.4-1build2) ...
Setting up python3-wheel (0.37.1-2ubuntu0.22.04.1) ...
Setting up libsource-highlight-common (3.1.9-4.1build2) ...
Setting up libfakeroot:amd64 (1.28-1ubuntu1) ...
Setting up libasan6:amd64 (11.4.0-1ubuntu1~22.04) ...
Setting up libsnappy1v5:amd64 (1.1.8-1build3) ...
Setting up libflac8:amd64 (1.3.3-2ubuntu0.2) ...
Setting up libc6-dbg:amd64 (2.35-0ubuntu3.9) ...
Setting up rocm-core (6.3.4.60304-76~22.04) ...
update-alternatives: using /opt/rocm-6.3.4 to provide /opt/rocm (rocm) in auto mode
Setting up libc6-x32 (2.35-0ubuntu3.9) ...
Setting up libfile-copy-recursive-perl (0.45-1) ...
Setting up fakeroot (1.28-1ubuntu1) ...
update-alternatives: using /usr/bin/fakeroot-sysv to provide /usr/bin/fakeroot (fakeroot) in auto mode
Setting up rocm-device-libs (1.0.0.60304-76~22.04) ...
Setting up libasound2-data (1.2.6.1-1ubuntu1) ...
Setting up xtrans-dev (1.4.0-1) ...
Setting up libblas3:amd64 (3.10.0-2ubuntu1) ...
update-alternatives: using /usr/lib/x86_64-linux-gnu/blas/libblas.so.3 to provide /usr/lib/x86_64-linux-gnu/libblas.so.3 (libblas.so.3-x86_64-linux-gnu) in auto mode
Setting up libtirpc-dev:amd64 (1.3.2-2ubuntu0.1) ...
Setting up rpcsvc-proto (1.4.2-0ubuntu6) ...
Setting up libass9:amd64 (1:0.15.2-1) ...
Setting up libva2:amd64 (2.14.0-1) ...
Setting up make (4.3-4.1build1) ...
Setting up libx264-163:amd64 (2:0.163.3060+git5db6aa6-2build1) ...
Setting up libboost-regex1.74.0:amd64 (1.74.0-14ubuntu3) ...
Setting up libopus0:amd64 (1.3.1-0.1build2) ...
Setting up libquadmath0:amd64 (12.3.0-1ubuntu1~22.04) ...
Setting up libgd3:amd64 (2.3.0-2ubuntu2.3) ...
Setting up libdc1394-25:amd64 (2.2.6-4) ...
Setting up intel-media-va-driver:amd64 (22.3.1+dfsg1-1ubuntu2) ...
Setting up libxv1:amd64 (2:1.0.11-1build2) ...
Setting up libmpc3:amd64 (1.2.1-2build1) ...
Setting up libatomic1:amd64 (12.3.0-1ubuntu1~22.04) ...
Setting up libvorbis0a:amd64 (1.3.7-1build2) ...
Setting up rocfft (1.0.31.60304-76~22.04) ...
Setting up icu-devtools (70.1-2) ...
Setting up libipt2 (2.0.5-1) ...
Setting up libaacs0:amd64 (0.11.1-1) ...
Setting up python3-pip (22.0.2+dfsg-1ubuntu0.5) ...
Setting up pocketsphinx-en-us (0.8.0+real5prealpha+1-14ubuntu1) ...
Setting up libx32gomp1 (12.3.0-1ubuntu1~22.04) ...
Setting up hipblas-common-dev (1.0.0.60304-76~22.04) ...
Setting up amdgpu-core (1:6.3.60304-2125197.22.04) ...
Setting up rocprofiler-register (0.4.0.60304-76~22.04) ...
Setting up libbabeltrace1:amd64 (1.5.8-2build1) ...
Setting up libdpkg-perl (1.21.1ubuntu2.3) ...
Setting up libgfortran5:amd64 (12.3.0-1ubuntu1~22.04) ...
Setting up libx265-199:amd64 (3.5-2) ...
Setting up rocwmma-dev (1.6.0.60304-76~22.04) ...
Setting up libtimedate-perl (2.3300-2) ...
Setting up libubsan1:amd64 (12.3.0-1ubuntu1~22.04) ...
Setting up libbdplus0:amd64 (0.2.0-1) ...
Setting up libvidstab1.1:amd64 (1.1.0-2) ...
Setting up alsa-topology-conf (1.2.5.1-2) ...
Setting up libva-drm2:amd64 (2.14.0-1) ...
Setting up libnsl-dev:amd64 (1.3.0-2build2) ...
Setting up ocl-icd-libopencl1:amd64 (2.2.14-3) ...
Setting up libasyncns0:amd64 (0.8-6build2) ...
Setting up libvdpau1:amd64 (1.4-3build2) ...
Setting up libcrypt-dev:amd64 (1:4.4.27-1) ...
Setting up libbs2b0:amd64 (3.1.0+dfsg-2.2build1) ...
Setting up hipify-clang (18.0.0.60304-76~22.04) ...
Setting up libdrm-amdgpu-common (1.0.0.60304-2125197.22.04) ...
Setting up libtheora0:amd64 (1.1.1+dfsg.1-15ubuntu4) ...
Setting up libasound2:amd64 (1.2.6.1-1ubuntu1) ...
Setting up libdecor-0-0:amd64 (0.1.0-3build1) ...
Setting up libc6-i386 (2.35-0ubuntu3.9) ...
Setting up libzimg2:amd64 (3.0.3+ds1-1) ...
Setting up libopenjp2-7:amd64 (2.4.0-6ubuntu0.3) ...
Setting up libopenal-data (1:1.19.1-2build3) ...
Setting up libx32quadmath0 (12.3.0-1ubuntu1~22.04) ...
Setting up xorg-sgml-doctools (1:1.11-1.1) ...
Setting up libvpx7:amd64 (1.11.0-2ubuntu2.3) ...
Setting up rocm-smi-lib (7.4.0.60304-76~22.04) ...
Removed /etc/systemd/system/timers.target.wants/logrotate.timer.
Created symlink /etc/systemd/system/timers.target.wants/logrotate.timer → /lib/systemd/system/logrotate.timer.
Setting up libxss1:amd64 (1:1.2.3-1build2) ...
Setting up mesa-va-drivers:amd64 (23.2.1-1ubuntu3.1~22.04.3) ...
Setting up libjs-jquery (3.6.0+dfsg+~3.5.13-1) ...
Setting up libdav1d5:amd64 (0.9.2-1) ...
Setting up libmfx1:amd64 (22.3.0-1) ...
Setting up libisl23:amd64 (0.24-2build1) ...
Setting up libbluray2:amd64 (1:1.3.1-1) ...
Setting up libc-dev-bin (2.35-0ubuntu3.9) ...
Setting up python3-argcomplete (1.8.1-1.5) ...
Setting up valgrind (1:3.18.1-1ubuntu2) ...
Setting up libsamplerate0:amd64 (0.2.2-1build1) ...
Setting up libva-x11-2:amd64 (2.14.0-1) ...
Setting up libwebpmux3:amd64 (1.2.2-2ubuntu0.22.04.2) ...
Setting up libalgorithm-diff-xs-perl (0.04-6build3) ...
Setting up lib32atomic1 (12.3.0-1ubuntu1~22.04) ...
Setting up libsuitesparseconfig5:amd64 (1:5.10.1+dfsg-4build1) ...
Setting up libcc1-0:amd64 (12.3.0-1ubuntu1~22.04) ...
Setting up liburi-perl (5.10-1) ...
Setting up hsa-runtime-rocr4wsl-amdgpu:amd64 (24.30-2127960.22.04) ...
Setting up libzvbi-common (0.2.35-19) ...
Setting up liblsan0:amd64 (12.3.0-1ubuntu1~22.04) ...
Setting up libmp3lame0:amd64 (3.100-3build2) ...
Setting up libitm1:amd64 (12.3.0-1ubuntu1~22.04) ...
Setting up i965-va-driver:amd64 (2.4.1+dfsg1-1) ...
Setting up libsource-highlight4v5 (3.1.9-4.1build2) ...
Setting up libvorbisenc2:amd64 (1.3.7-1build2) ...
Setting up libc-devtools (2.35-0ubuntu3.9) ...
Setting up libjs-underscore (1.13.2~dfsg-2) ...
Setting up libalgorithm-merge-perl (0.08-3) ...
Setting up libiec61883-0:amd64 (1.2.0-4build3) ...
Setting up libserd-0-0:amd64 (0.30.10-2) ...
Setting up rocrand (3.2.0.60304-76~22.04) ...
Setting up libtsan0:amd64 (11.4.0-1ubuntu1~22.04) ...
Setting up libx32atomic1 (12.3.0-1ubuntu1~22.04) ...
Setting up x11proto-dev (2021.5-1) ...
Setting up libavc1394-0:amd64 (0.5.4-5build2) ...
Setting up cpp-11 (11.4.0-1ubuntu1~22.04) ...
Setting up mesa-vdpau-drivers:amd64 (23.2.1-1ubuntu3.1~22.04.3) ...
Setting up libzvbi0:amd64 (0.2.35-19) ...
Setting up libamd2:amd64 (1:5.10.1+dfsg-4build1) ...
Setting up hiprand (2.11.1.60304-76~22.04) ...
Setting up libhttp-date-perl (6.05-1) ...
Setting up liblapack3:amd64 (3.10.0-2ubuntu1) ...
update-alternatives: using /usr/lib/x86_64-linux-gnu/lapack/liblapack.so.3 to provide /usr/lib/x86_64-linux-gnu/liblapack.so.3 (liblapack.so.3-x86_64-linux-gnu) in auto mode
Setting up libdrm-dev:amd64 (2.4.113-2~ubuntu0.22.04.1) ...
Setting up roctracer (4.1.60304.60304-76~22.04) ...
Setting up lib32itm1 (12.3.0-1ubuntu1~22.04) ...
Setting up libfile-listing-perl (6.14-1) ...
Setting up libzmq5:amd64 (4.3.4-2) ...
Setting up libxau-dev:amd64 (1:1.0.9-1build5) ...
Setting up libcolamd2:amd64 (1:5.10.1+dfsg-4build1) ...
Setting up rocrand-dev (3.2.0.60304-76~22.04) ...
Setting up composablekernel-dev (1.1.0.60304-76~22.04) ...
Setting up amd-smi-lib (25.1.0.60304-76~22.04) ...
Using pyproject.toml for installation due to setuptools version 59.6.0
WARNING: Running pip as the 'root' user can result in broken permissions and conflicting behaviour with the system package manager. It is recommended to use a virtual environment instead: https://pip.pypa.io/warnings/venv
Installing bash completion script /etc/bash_completion.d/python-argcomplete.sh
Removed /etc/systemd/system/timers.target.wants/logrotate.timer.
Created symlink /etc/systemd/system/timers.target.wants/logrotate.timer → /lib/systemd/system/logrotate.timer.
Setting up alsa-ucm-conf (1.2.6.3-1ubuntu1.12) ...
Setting up libsoxr0:amd64 (0.1.3-4build2) ...
Setting up libcdio-cdda2:amd64 (10.2+2.0.0-1build3) ...
Setting up rocm-cmake (0.14.0.60304-76~22.04) ...
Setting up rocprofiler-sdk-roctx (0.5.0-76~22.04) ...
Setting up libx32gcc-s1 (12.3.0-1ubuntu1~22.04) ...
Setting up libcdio-paranoia2:amd64 (10.2+2.0.0-1build3) ...
Setting up rocminfo (1.0.0.60304-76~22.04) ...
Setting up hsa-amd-aqlprofile (1.0.0.60304-76~22.04) ...
Setting up hipfft (1.0.17.60304-76~22.04) ...
Setting up libx32itm1 (12.3.0-1ubuntu1~22.04) ...
Setting up hipfft-dev (1.0.17.60304-76~22.04) ...
Setting up hiptensor (1.4.0.60304-76~22.04) ...
Setting up half (1.12.0.60304-76~22.04) ...
Setting up hipblaslt (0.10.0.60304-76~22.04) ...
Setting up hsa-rocr-dev (1.14.0.60304-76~22.04) ...
Setting up openmp-extras-runtime (18.63.0.60304-76~22.04) ...
Setting up libavutil56:amd64 (7:4.4.2-0ubuntu0.22.04.1) ...
Setting up dpkg-dev (1.21.1ubuntu2.3) ...
Setting up rocm-utils (6.3.4.60304-76~22.04) ...
Setting up libcamd2:amd64 (1:5.10.1+dfsg-4build1) ...
Setting up libvorbisfile3:amd64 (1.3.7-1build2) ...
Setting up libxdmcp-dev:amd64 (1:1.1.3-0ubuntu5) ...
Setting up gdb (12.1-0ubuntu1~22.04.2) ...
Setting up lib32gomp1 (12.3.0-1ubuntu1~22.04) ...
Setting up hiptensor-dev (1.4.0.60304-76~22.04) ...
Setting up libdrm2-amdgpu:amd64 (1:2.4.123.60304-2125197.22.04) ...
Setting up libx32asan6 (11.4.0-1ubuntu1~22.04) ...
Setting up lib32gcc-s1 (12.3.0-1ubuntu1~22.04) ...
Setting up lib32stdc++6 (12.3.0-1ubuntu1~22.04) ...
Setting up va-driver-all:amd64 (2.14.0-1) ...
Setting up rocfft-dev (1.0.31.60304-76~22.04) ...
Setting up hiprand-dev (2.11.1.60304-76~22.04) ...
Setting up libdecor-0-plugin-1-cairo:amd64 (0.1.0-3build1) ...
Setting up libpostproc55:amd64 (7:4.4.2-0ubuntu0.22.04.1) ...
Setting up rocprofiler-sdk (0.5.0-76~22.04) ...
Setting up libjs-sphinxdoc (4.3.2-1) ...
Setting up lib32asan6 (11.4.0-1ubuntu1~22.04) ...
Setting up librubberband2:amd64 (2.0.0-2) ...
Setting up libsndio7.0:amd64 (1.8.1-1.1) ...
Setting up libjack-jackd2-0:amd64 (1.9.20~dfsg-1) ...
Setting up libgcc-11-dev:amd64 (11.4.0-1ubuntu1~22.04) ...
Setting up vdpau-driver-all:amd64 (1.4-3build2) ...
Setting up libflite1:amd64 (2.2-3) ...
Setting up gcc-11 (11.4.0-1ubuntu1~22.04) ...
Setting up libsord-0-0:amd64 (0.16.8-2) ...
Setting up libccolamd2:amd64 (1:5.10.1+dfsg-4build1) ...
Setting up cpp (4:11.2.0-1ubuntu1) ...
Setting up libsratom-0-0:amd64 (0.6.8-1) ...
Setting up hipblaslt-dev (0.10.0.60304-76~22.04) ...
Setting up lib32quadmath0 (12.3.0-1ubuntu1~22.04) ...
Setting up libc6-dev:amd64 (2.35-0ubuntu3.9) ...
Setting up libswscale5:amd64 (7:4.4.2-0ubuntu0.22.04.1) ...
Setting up libsndfile1:amd64 (1.0.31-2ubuntu0.2) ...
Setting up liblilv-0-0:amd64 (0.24.12-2) ...
Setting up libicu-dev:amd64 (70.1-2) ...
Setting up libopenmpt0:amd64 (0.6.1-1) ...
Setting up libcholmod3:amd64 (1:5.10.1+dfsg-4build1) ...
Setting up libx32stdc++6 (12.3.0-1ubuntu1~22.04) ...
Setting up libavutil-dev:amd64 (7:4.4.2-0ubuntu0.22.04.1) ...
Setting up libncurses-dev:amd64 (6.3-2ubuntu0.1) ...
Setting up libc6-dev-i386 (2.35-0ubuntu3.9) ...
Setting up libxcb1-dev:amd64 (1.14-3ubuntu3) ...
Setting up libdrm-amdgpu-radeon1:amd64 (1:2.4.123.60304-2125197.22.04) ...
Setting up libx32ubsan1 (12.3.0-1ubuntu1~22.04) ...
Setting up libpulse0:amd64 (1:15.99.1+dfsg1-1ubuntu2.2) ...
Setting up libdrm-amdgpu-amdgpu1:amd64 (1:2.4.123.60304-2125197.22.04) ...
Setting up roctracer-dev (4.1.60304.60304-76~22.04) ...
Setting up libx11-dev:amd64 (2:1.7.5-1ubuntu0.3) ...
Setting up libopenal1:amd64 (1:1.19.1-2build3) ...
Setting up libswresample3:amd64 (7:4.4.2-0ubuntu0.22.04.1) ...
Setting up lib32ubsan1 (12.3.0-1ubuntu1~22.04) ...
Setting up gcc (4:11.2.0-1ubuntu1) ...
Setting up libnuma-dev:amd64 (2.0.14-3ubuntu2) ...
Setting up libc6-dev-x32 (2.35-0ubuntu3.9) ...
Setting up libxml2-dev:amd64 (2.9.13+dfsg-1ubuntu0.7) ...
Setting up libx32gcc-11-dev (11.4.0-1ubuntu1~22.04) ...
Setting up libexpat1-dev:amd64 (2.4.7-1ubuntu0.6) ...
Setting up libdrm-amdgpu-dev:amd64 (1:2.4.123.60304-2125197.22.04) ...
Setting up libswscale-dev:amd64 (7:4.4.2-0ubuntu0.22.04.1) ...
Setting up libstdc++-11-dev:amd64 (11.4.0-1ubuntu1~22.04) ...
Setting up zlib1g-dev:amd64 (1:1.2.11.dfsg-2ubuntu9.2) ...
Setting up libglx-dev:amd64 (1.4.0-1) ...
Setting up libavcodec58:amd64 (7:4.4.2-0ubuntu0.22.04.1) ...
Setting up libsdl2-2.0-0:amd64 (2.0.20+dfsg-2ubuntu1.22.04.1) ...
Setting up lib32gcc-11-dev (11.4.0-1ubuntu1~22.04) ...
Setting up libgl-dev:amd64 (1.4.0-1) ...
Setting up libchromaprint1:amd64 (1.5.1-2) ...
Setting up lib32stdc++-11-dev (11.4.0-1ubuntu1~22.04) ...
Setting up rocm-llvm (18.0.0.25012.60304-76~22.04) ...
Setting up libtinfo-dev:amd64 (6.3-2ubuntu0.1) ...
Setting up comgr (2.8.0.60304-76~22.04) ...
Setting up libsphinxbase3:amd64 (0.8+5prealpha+1-13build1) ...
Setting up g++-11 (11.4.0-1ubuntu1~22.04) ...
Setting up libswresample-dev:amd64 (7:4.4.2-0ubuntu0.22.04.1) ...
Setting up libavformat58:amd64 (7:4.4.2-0ubuntu0.22.04.1) ...
Setting up libavcodec-dev:amd64 (7:4.4.2-0ubuntu0.22.04.1) ...
Setting up openmp-extras-dev (18.63.0.60304-76~22.04) ...
Setting up libavformat-dev:amd64 (7:4.4.2-0ubuntu0.22.04.1) ...
Setting up rocm-language-runtime (6.3.4.60304-76~22.04) ...
Setting up hip-runtime-amd (6.3.42134.60304-76~22.04) ...
Setting up libpocketsphinx3:amd64 (0.8.0+real5prealpha+1-14ubuntu1) ...
Setting up rocprim-dev (3.3.0.60304-76~22.04) ...
Setting up rocm-dbgapi (0.77.0.60304-76~22.04) ...
Setting up hipcub-dev (3.3.0.60304-76~22.04) ...
Setting up libx32stdc++-11-dev (11.4.0-1ubuntu1~22.04) ...
Setting up libelf-dev:amd64 (0.186-1ubuntu0.1) ...
Setting up rocm-opencl (2.0.0.60304-76~22.04) ...
Setting up libpython3.10-dev:amd64 (3.10.12-1~22.04.9) ...
Setting up rocm-hip-runtime (6.3.4.60304-76~22.04) ...
update-alternatives: using /opt/rocm-6.3.4/bin/rocm_agent_enumerator to provide /usr/bin/rocm_agent_enumerator (rocm_agent_enumerator) in auto mode
update-alternatives: using /opt/rocm-6.3.4/bin/rocminfo to provide /usr/bin/rocminfo (rocminfo) in auto mode
Setting up python3.10-dev (3.10.12-1~22.04.9) ...
Setting up g++ (4:11.2.0-1ubuntu1) ...
update-alternatives: using /usr/bin/g++ to provide /usr/bin/c++ (c++) in auto mode
Setting up libavfilter7:amd64 (7:4.4.2-0ubuntu0.22.04.1) ...
Setting up rocm-openmp-sdk (6.3.4.60304-76~22.04) ...
Setting up hipfort-dev (0.5.1.60304-76~22.04) ...
Setting up build-essential (12.9ubuntu3) ...
Setting up mesa-common-dev:amd64 (23.2.1-1ubuntu3.1~22.04.3) ...
Setting up rocblas (4.3.0.60304-76~22.04) ...
Setting up gcc-11-multilib (11.4.0-1ubuntu1~22.04) ...
Setting up rccl (2.21.5.60304-76~22.04) ...
Setting up gcc-multilib (4:11.2.0-1ubuntu1) ...
Setting up hip-dev (6.3.42134.60304-76~22.04) ...
Setting up libpython3-dev:amd64 (3.10.6-1~22.04.1) ...
Setting up rpp (1.9.1.60304-76~22.04) ...
Setting up rocprofiler (2.0.60304.60304-76~22.04) ...
Setting up rocm-opencl-runtime (6.3.4.60304-76~22.04) ...
update-alternatives: using /opt/rocm-6.3.4/bin/clinfo to provide /usr/bin/clinfo (clinfo) in auto mode
Setting up rocsparse (3.3.0.60304-76~22.04) ...
Setting up g++-11-multilib (11.4.0-1ubuntu1~22.04) ...
Setting up miopen-hip (3.3.0.60304-76~22.04) ...
Setting up rocm-opencl-dev (2.0.0.60304-76~22.04) ...
Setting up rocprofiler-dev (2.0.60304.60304-76~22.04) ...
Setting up rocthrust-dev (3.3.0.60304-76~22.04) ...
Setting up g++-multilib (4:11.2.0-1ubuntu1) ...
Setting up migraphx (2.11.0.60304-76~22.04) ...
Setting up hipsparse (3.1.2.60304-76~22.04) ...
Setting up rocprofiler-plugins (2.0.60304.60304-76~22.04) ...
Setting up rocm-debug-agent (2.0.3.60304-76~22.04) ...
Setting up rocsolver (3.27.0.60304-76~22.04) ...
Setting up miopen-hip-dev (3.3.0.60304-76~22.04) ...
Setting up mivisionx (3.1.0.60304-76~22.04) ...
Setting up python3-dev (3.10.6-1~22.04.1) ...
Setting up rocm-opencl-sdk (6.3.4.60304-76~22.04) ...
Setting up hipblas (2.3.0.60304-76~22.04) ...
Setting up libavdevice58:amd64 (7:4.4.2-0ubuntu0.22.04.1) ...
Setting up rccl-dev (2.21.5.60304-76~22.04) ...
Setting up rocblas-dev (4.3.0.60304-76~22.04) ...
Setting up hipcc (1.1.1.60304-76~22.04) ...
Setting up ffmpeg (7:4.4.2-0ubuntu0.22.04.1) ...
Setting up rocsparse-dev (3.3.0.60304-76~22.04) ...
Setting up hip-doc (6.3.42134.60304-76~22.04) ...
Setting up hipsparse-dev (3.1.2.60304-76~22.04) ...
Setting up hipblas-dev (2.3.0.60304-76~22.04) ...
Setting up rocsolver-dev (3.27.0.60304-76~22.04) ...
Setting up rocalution (3.2.1.60304-76~22.04) ...
Setting up migraphx-dev (2.11.0.60304-76~22.04) ...
Setting up hipsolver (2.3.0.60304-76~22.04) ...
Setting up hip-samples (6.3.42134.60304-76~22.04) ...
Setting up hipsolver-dev (2.3.0.60304-76~22.04) ...
Setting up hipsparselt (0.2.2.60304-76~22.04) ...
Setting up rocm-hip-runtime-dev (6.3.4.60304-76~22.04) ...
update-alternatives: using /opt/rocm-6.3.4/bin/roc-obj to provide /usr/bin/roc-obj (roc-obj) in auto mode
update-alternatives: using /opt/rocm-6.3.4/bin/roc-obj-extract to provide /usr/bin/roc-obj-extract (roc-obj-extract) in auto mode
update-alternatives: using /opt/rocm-6.3.4/bin/roc-obj-ls to provide /usr/bin/roc-obj-ls (roc-obj-ls) in auto mode
update-alternatives: using /opt/rocm-6.3.4/bin/hipcc to provide /usr/bin/hipcc (hipcc) in auto mode
update-alternatives: using /opt/rocm-6.3.4/bin/hipcc.pl to provide /usr/bin/hipcc.pl (hipcc.pl) in auto mode
/opt/rocm-6.3.4/bin/hipcc.bin not found, but that is OK
update-alternatives: using /opt/rocm-6.3.4/bin/hipcc_cmake_linker_helper to provide /usr/bin/hipcc_cmake_linker_helper (hipcc_cmake_linker_helper) in auto mode
update-alternatives: using /opt/rocm-6.3.4/bin/hipconfig to provide /usr/bin/hipconfig (hipconfig) in auto mode
update-alternatives: using /opt/rocm-6.3.4/bin/hipconfig.pl to provide /usr/bin/hipconfig.pl (hipconfig.pl) in auto mode
/opt/rocm-6.3.4/bin/hipconfig.bin not found, but that is OK
update-alternatives: using /opt/rocm-6.3.4/bin/hipconvertinplace-perl.sh to provide /usr/bin/hipconvertinplace-perl.sh (hipconvertinplace-perl.sh) in auto mode
update-alternatives: using /opt/rocm-6.3.4/bin/hipconvertinplace.sh to provide /usr/bin/hipconvertinplace.sh (hipconvertinplace.sh) in auto mode
update-alternatives: using /opt/rocm-6.3.4/bin/hipdemangleatp to provide /usr/bin/hipdemangleatp (hipdemangleatp) in auto mode
update-alternatives: using /opt/rocm-6.3.4/bin/hipexamine-perl.sh to provide /usr/bin/hipexamine-perl.sh (hipexamine-perl.sh) in auto mode
update-alternatives: using /opt/rocm-6.3.4/bin/hipexamine.sh to provide /usr/bin/hipexamine.sh (hipexamine.sh) in auto mode
update-alternatives: using /opt/rocm-6.3.4/bin/hipify-perl to provide /usr/bin/hipify-perl (hipify-perl) in auto mode
update-alternatives: using /opt/rocm-6.3.4/bin/hipify-clang to provide /usr/bin/hipify-clang (hipify-clang) in auto mode
update-alternatives: using /opt/rocm-6.3.4/bin/amdclang to provide /usr/bin/amdclang (amdclang) in auto mode
update-alternatives: using /opt/rocm-6.3.4/bin/amdclang++ to provide /usr/bin/amdclang++ (amdclang++) in auto mode
update-alternatives: using /opt/rocm-6.3.4/bin/amdflang to provide /usr/bin/amdflang (amdflang) in auto mode
update-alternatives: using /opt/rocm-6.3.4/bin/amdlld to provide /usr/bin/amdlld (amdlld) in auto mode
Setting up rocm-hip-libraries (6.3.4.60304-76~22.04) ...
Setting up rocm-gdb (15.2.60304-76~22.04) ...
Running post-installation script...
Installing rocm-gdb with [/lib/python3.10/config-3.10-x86_64-linux-gnu/libpython3.10.so].
post-installation done.
Setting up rocm-developer-tools (6.3.4.60304-76~22.04) ...
update-alternatives: using /opt/rocm-6.3.4/bin/amd-smi to provide /usr/bin/amd-smi (amd-smi) in auto mode
update-alternatives: using /opt/rocm-6.3.4/bin/rocgdb to provide /usr/bin/rocgdb (rocgdb) in auto mode
update-alternatives: using /opt/rocm-6.3.4/bin/rocm-smi to provide /usr/bin/rocm-smi (rocm-smi) in auto mode
update-alternatives: using /opt/rocm-6.3.4/bin/rocprof to provide /usr/bin/rocprof (rocprof) in auto mode
update-alternatives: using /opt/rocm-6.3.4/bin/rocsys to provide /usr/bin/rocsys (rocsys) in auto mode
update-alternatives: using /opt/rocm-6.3.4/bin/rocprofv2 to provide /usr/bin/rocprofv2 (rocprofv2) in auto mode
update-alternatives: using /opt/rocm-6.3.4/bin/roccoremerge to provide /usr/bin/roccoremerge (roccoremerge) in auto mode
update-alternatives: using /opt/rocm-6.3.4/bin/rocprofv3 to provide /usr/bin/rocprofv3 (rocprofv3) in auto mode
Setting up rocalution-dev (3.2.1.60304-76~22.04) ...
Setting up rpp-dev (1.9.1.60304-76~22.04) ...
Setting up hipsparselt-dev (0.2.2.60304-76~22.04) ...
Setting up rocm-ml-libraries (6.3.4.60304-76~22.04) ...
Setting up rocm-hip-sdk (6.3.4.60304-76~22.04) ...
update-alternatives: using /opt/rocm-6.3.4/bin/hipfc to provide /usr/bin/hipfc (hipfc) in auto mode
Setting up mivisionx-dev (3.1.0.60304-76~22.04) ...
Setting up rocm-ml-sdk (6.3.4.60304-76~22.04) ...
Setting up rocm (6.3.4.60304-76~22.04) ...
update-alternatives: using /opt/rocm-6.3.4/bin/runvx to provide /usr/bin/runvx (runvx) in auto mode
Processing triggers for man-db (2.10.2-1) ...
Processing triggers for libc-bin (2.35-0ubuntu3.9) ...
```

</details>

### Verify ROCm

If this has worked, now ```rocminfo``` should detect the GPU as ROCm agent.


```
rocminfo
```

<details>
<summary>Verify ROCm logs</summary>

```
meridia@TowerOfBabel:~$ rocminfo
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
      Size:                    32779776(0x1f42e00) KB
      Allocatable:             TRUE
      Alloc Granule:           4KB
      Alloc Recommended Granule:4KB
      Alloc Alignment:         4KB
      Accessible by all:       TRUE
    Pool 2
      Segment:                 GLOBAL; FLAGS: EXTENDED FINE GRAINED
      Size:                    32779776(0x1f42e00) KB
      Allocatable:             TRUE
      Alloc Granule:           4KB
      Alloc Recommended Granule:4KB
      Alloc Alignment:         4KB
      Accessible by all:       TRUE
    Pool 3
      Segment:                 GLOBAL; FLAGS: KERNARG, FINE GRAINED
      Size:                    32779776(0x1f42e00) KB
      Allocatable:             TRUE
      Alloc Granule:           4KB
      Alloc Recommended Granule:4KB
      Alloc Alignment:         4KB
      Accessible by all:       TRUE
    Pool 4
      Segment:                 GLOBAL; FLAGS: COARSE GRAINED
      Size:                    32779776(0x1f42e00) KB
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
  Max Clock Freq. (MHz):   2482000
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
  Packet Processor uCode:: 542
  SDMA engine uCode::      24
  IOMMU Support::          None
  Pool Info:
    Pool 1
      Segment:                 GLOBAL; FLAGS: COARSE GRAINED
      Size:                    25101988(0x17f06a4) KB
      Allocatable:             TRUE
      Alloc Granule:           4KB
      Alloc Recommended Granule:2048KB
      Alloc Alignment:         4KB
      Accessible by all:       FALSE
    Pool 2
      Segment:                 GLOBAL; FLAGS: EXTENDED FINE GRAINED
      Size:                    25101988(0x17f06a4) KB
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
```

</details>



## STEP 3 - Python with portable environment

Now, It's time to setup python.

Versions:
- 3.10: I tested it in my previous WSL attempt
- 3.12: online people say it works
- 3.13: online people say it causes issues with ComfyUI

I'm going with python 3.12. Good luck me.

Virtual Environment:
- virtualenv: I tested it successfully in other project
- uv: I was advised to try uv

[UV installation guide](https://docs.astral.sh/uv/getting-started/installation/)

```
curl -LsSf https://astral.sh/uv/install.sh | sh

sudo reboot

uv

uv python install 3.12
```

<details>
<summary>UV installation</summary>

```
meridia@TowerOfBabel:~$ curl -LsSf https://astral.sh/uv/install.sh | sh
downloading uv 0.7.4 x86_64-unknown-linux-gnu
no checksums to verify
installing to /home/meridia/.local/bin
  uv
  uvx
everything's installed!

To add $HOME/.local/bin to your PATH, either restart your shell or run:

    source $HOME/.local/bin/env (sh, bash, zsh)
    source $HOME/.local/bin/env.fish (fish)

meridia@TowerOfBabel:~/ComfyUI$ sudo reboot

Session terminated, killing shell... ...killed.
                                               Terminated
root@TowerOfBabel:/mnt/c/Users/FatherOfMachines#
C:\Users\FatherOfMachines>wsl
root@TowerOfBabel:/mnt/c/Users/FatherOfMachines# su meridia
meridia@TowerOfBabel:/mnt/c/Users/FatherOfMachines$ cd

meridia@TowerOfBabel:~$ uv
An extremely fast Python package manager.

Usage: uv [OPTIONS] <COMMAND>

Commands:
  run      Run a command or script
  init     Create a new project
  add      Add dependencies to the project
  remove   Remove dependencies from the project
  sync     Update the project's environment
  lock     Update the project's lockfile
  export   Export the project's lockfile to an alternate format
  tree     Display the project's dependency tree
  tool     Run and install commands provided by Python packages
  python   Manage Python versions and installations
  pip      Manage Python packages with a pip-compatible interface
  venv     Create a virtual environment
  build    Build Python packages into source distributions and wheels
  publish  Upload distributions to an index
  cache    Manage uv's cache
  self     Manage the uv executable
  version  Read or update the project's version
  help     Display documentation for a command

Cache options:
  -n, --no-cache               Avoid reading from or writing to the cache, instead using a temporary directory for
                               the duration of the operation [env: UV_NO_CACHE=]
      --cache-dir <CACHE_DIR>  Path to the cache directory [env: UV_CACHE_DIR=]

Python options:
      --managed-python       Require use of uv-managed Python versions [env: UV_MANAGED_PYTHON=]
      --no-managed-python    Disable use of uv-managed Python versions [env: UV_NO_MANAGED_PYTHON=]
      --no-python-downloads  Disable automatic downloads of Python. [env: "UV_PYTHON_DOWNLOADS=never"]

Global options:
  -q, --quiet...
          Use quiet output
  -v, --verbose...
          Use verbose output
      --color <COLOR_CHOICE>
          Control the use of color in output [possible values: auto, always, never]
      --native-tls
          Whether to load TLS certificates from the platform's native certificate store [env: UV_NATIVE_TLS=]
      --offline
          Disable network access [env: UV_OFFLINE=]
      --allow-insecure-host <ALLOW_INSECURE_HOST>
          Allow insecure connections to a host [env: UV_INSECURE_HOST=]
      --no-progress
          Hide all progress outputs [env: UV_NO_PROGRESS=]
      --directory <DIRECTORY>
          Change to the given directory prior to running the command
      --project <PROJECT>
          Run the command within the given project directory [env: UV_PROJECT=]
      --config-file <CONFIG_FILE>
          The path to a `uv.toml` file to use for configuration [env: UV_CONFIG_FILE=]
      --no-config
          Avoid discovering configuration files (`pyproject.toml`, `uv.toml`) [env: UV_NO_CONFIG=]
  -h, --help
          Display the concise help for this command
  -V, --version
          Display the uv version

Use `uv help` for more details.

meridia@TowerOfBabel:~$ uv python install 3.12
Installed Python 3.12.10 in 3.30s
 + cpython-3.12.10-linux-x86_64-gnu

```

</details><br>

Now that UV is installed, it's time to create the virtual environment where Comfy UI will rest


```
mkdir ComfyUI
cd ComfyUI
uv 
uv venv Dreamy --python 3.12
source Dreamy/bin/activate
sudo shutdown now
```

<details>
<summary>ComfyUI UV venv creation</summary>

```
meridia@TowerOfBabel:~$ cd ComfyUI/
meridia@TowerOfBabel:~/ComfyUI$ uv Dreamy --python 3.12
error: unrecognized subcommand 'Dreamy'

Usage: uv [OPTIONS] <COMMAND>

For more information, try '--help'.
meridia@TowerOfBabel:~/ComfyUI$ uv venv Dreamy --python 3.12
Using CPython 3.12.10
Creating virtual environment at: Dreamy
Activate with: source Dreamy/bin/activate
meridia@TowerOfBabel:~/ComfyUI$ source Dreamy/bin/activate
(Dreamy) meridia@TowerOfBabel:~/ComfyUI$
```

</details>

## STEP 4 - Pytorch ROCm Binaries

With the UV virtual environment setup, it's not time to install the pytorch binaries.

NOTE: I changed the official commands to use UV. I have no idea if this works, good luck me!

This sets of command enter the repo, activate UV, and start upgrading wheels, which is a way to download pre built distribution faster.

```
wsl
su meridia
cd

uv python 

cd ComfyUI 

uv python list

source Dreamy/bin/activate

uv pip install --upgrade pip wheel
```

<details>
<summary>Prepare UV and wheels logs</summary>

```
root@TowerOfBabel:/mnt/c/Users/FatherOfMachines# su meridia
meridia@TowerOfBabel:/mnt/c/Users/FatherOfMachines$ cd
meridia@TowerOfBabel:~$ uv python list
cpython-3.14.0a6-linux-x86_64-gnu                 <download available>
cpython-3.14.0a6+freethreaded-linux-x86_64-gnu    <download available>
cpython-3.13.3-linux-x86_64-gnu                   <download available>
cpython-3.13.3+freethreaded-linux-x86_64-gnu      <download available>
cpython-3.12.10-linux-x86_64-gnu                  .local/share/uv/python/cpython-3.12.10-linux-x86_64-gnu/bin/python3.12
cpython-3.11.12-linux-x86_64-gnu                  <download available>
cpython-3.10.17-linux-x86_64-gnu                  <download available>
cpython-3.10.12-linux-x86_64-gnu                  /usr/bin/python3.10
cpython-3.10.12-linux-x86_64-gnu                  /usr/bin/python3 -> python3.10
cpython-3.9.22-linux-x86_64-gnu                   <download available>
cpython-3.8.20-linux-x86_64-gnu                   <download available>
pypy-3.11.11-linux-x86_64-gnu                     <download available>
pypy-3.10.16-linux-x86_64-gnu                     <download available>
pypy-3.9.19-linux-x86_64-gnu                      <download available>
pypy-3.8.16-linux-x86_64-gnu                      <download available>
graalpy-3.11.0-linux-x86_64-gnu                   <download available>
graalpy-3.10.0-linux-x86_64-gnu                   <download available>
graalpy-3.8.5-linux-x86_64-gnu                    <download available>
meridia@TowerOfBabel:~$ cd ComfyUI/
meridia@TowerOfBabel:~/ComfyUI$ source Dreamy/bin/activate
(Dreamy) meridia@TowerOfBabel:~/ComfyUI$ uv pip install --upgrade pip wheel
Using Python 3.12.10 environment at: Dreamy
Resolved 2 packages in 281ms
Prepared 2 packages in 247ms
Installed 2 packages in 27ms
 + pip==25.1.1
 + wheel==0.45.1

```

</details><br>

The next sets  of command are to download the wheels for the ROCm pytorch binaries, and once downloaded, install them. This is a huge step that takes lots of time and tens of GB to download.

Here I am doing a bit of an hack, because AMD lists Python 3.10 for Ubuntu 22, and Python 3.12 for Ubuntu 24, here I'm setting Python 3.12 on Ubuntu 22.


Download Python 3.12 wheels, and use UV to install inside the portable environments the wheels.

```
wget https://repo.radeon.com/rocm/manylinux/rocm-rel-6.3.4/torch-2.4.0%2Brocm6.3.4.git7cecbf6d-cp312-cp312-linux_x86_64.whl

wget https://repo.radeon.com/rocm/manylinux/rocm-rel-6.3.4/torchvision-0.19.0%2Brocm6.3.4.gitfab84886-cp312-cp312-linux_x86_64.whl

wget https://repo.radeon.com/rocm/manylinux/rocm-rel-6.3.4/pytorch_triton_rocm-3.0.0%2Brocm6.3.4.git75cc27c2-cp312-cp312-linux_x86_64.whl

wget https://repo.radeon.com/rocm/manylinux/rocm-rel-6.3.4/torchaudio-2.4.0%2Brocm6.3.4.git69d40773-cp312-cp312-linux_x86_64.whl

uv pip uninstall torch torchvision pytorch-triton-rocm

uv pip install torch-2.4.0+rocm6.3.4.git7cecbf6d-cp312-cp312-linux_x86_64.whl torchvision-0.19.0+rocm6.3.4.gitfab84886-cp312-cp312-linux_x86_64.whl torchaudio-2.4.0+rocm6.3.4.git69d40773-cp312-cp312-linux_x86_64.whl pytorch_triton_rocm-3.0.0+rocm6.3.4.git75cc27c2-cp312-cp312-linux_x86_64.whl

location=$(pip show torch | grep Location | awk -F ": " '{print $2}')

cd ${location}/torch/lib/

rm libhsa-runtime64.so*

cd

cd ComfyUI

```

<details>
<summary>Install pytorch ROCm binaries from wheel in the UV venv</summary>

```
(Dreamy) meridia@TowerOfBabel:~/ComfyUI$ wget https://repo.radeon.com/rocm/manylinux/rocm-rel-6.3.4/torch-2.4.0%2Brocm6.3.4.git7cecbf6d-cp312-cp312-linux_x86_64.whl

wget https://repo.radeon.com/rocm/manylinux/rocm-rel-6.3.4/torchvision-0.19.0%2Brocm6.3.4.gitfab84886-cp312-cp312-linux_x86_64.whl

wget https://repo.radeon.com/rocm/manylinux/rocm-rel-6.3.4/pytorch_triton_rocm-3.0.0%2Brocm6.3.4.git75cc27c2-cp312-cp312-linux_x86_64.whl

wget https://repo.radeon.com/rocm/manylinux/rocm-rel-6.3.4/torchaudio-2.4.0%2Brocm6.3.4.git69d40773-cp312-cp312-linux_x86_64.whl
--2025-05-16 11:25:57--  https://repo.radeon.com/rocm/manylinux/rocm-rel-6.3.4/torch-2.4.0%2Brocm6.3.4.git7cecbf6d-cp312-cp312-linux_x86_64.whl
Resolving repo.radeon.com (repo.radeon.com)... 23.53.42.179, 23.53.42.139, 2a02:26f0:7100::687e:25f2, ...
Connecting to repo.radeon.com (repo.radeon.com)|23.53.42.179|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 5074215944 (4.7G) [application/octet-stream]
Saving to: ‘torch-2.4.0+rocm6.3.4.git7cecbf6d-cp312-cp312-linux_x86_64.whl’

torch-2.4.0+rocm6.3.4.git7ce 100%[==============================================>]   4.73G  8.37MB/s    in 10m 7s

2025-05-16 11:36:24 (7.97 MB/s) - ‘torch-2.4.0+rocm6.3.4.git7cecbf6d-cp312-cp312-linux_x86_64.whl’ saved [5074215944/5074215944]

--2025-05-16 11:36:24--  https://repo.radeon.com/rocm/manylinux/rocm-rel-6.3.4/torchvision-0.19.0%2Brocm6.3.4.gitfab84886-cp312-cp312-linux_x86_64.whl
Resolving repo.radeon.com (repo.radeon.com)... 23.53.42.179, 23.53.42.162, 2a02:26f0:7100::213:c6f2, ...
Connecting to repo.radeon.com (repo.radeon.com)|23.53.42.179|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 69965270 (67M) [application/octet-stream]
Saving to: ‘torchvision-0.19.0+rocm6.3.4.gitfab84886-cp312-cp312-linux_x86_64.whl’

torchvision-0.19.0+rocm6.3.4 100%[==============================================>]  66.72M  7.91MB/s    in 8.4s

2025-05-16 11:36:34 (7.91 MB/s) - ‘torchvision-0.19.0+rocm6.3.4.gitfab84886-cp312-cp312-linux_x86_64.whl’ saved [69965270/69965270]

--2025-05-16 11:36:34--  https://repo.radeon.com/rocm/manylinux/rocm-rel-6.3.4/pytorch_triton_rocm-3.0.0%2Brocm6.3.4.git75cc27c2-cp312-cp312-linux_x86_64.whl
Resolving repo.radeon.com (repo.radeon.com)... 23.53.42.179, 23.53.42.162, 2a02:26f0:7100::213:c6f2, ...
Connecting to repo.radeon.com (repo.radeon.com)|23.53.42.179|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 235407299 (225M) [application/octet-stream]
Saving to: ‘pytorch_triton_rocm-3.0.0+rocm6.3.4.git75cc27c2-cp312-cp312-linux_x86_64.whl’

pytorch_triton_rocm-3.0.0+ro 100%[==============================================>] 224.50M  8.01MB/s    in 28s

2025-05-16 11:37:03 (8.16 MB/s) - ‘pytorch_triton_rocm-3.0.0+rocm6.3.4.git75cc27c2-cp312-cp312-linux_x86_64.whl’ saved [235407299/235407299]

--2025-05-16 11:37:03--  https://repo.radeon.com/rocm/manylinux/rocm-rel-6.3.4/torchaudio-2.4.0%2Brocm6.3.4.git69d40773-cp312-cp312-linux_x86_64.whl
Resolving repo.radeon.com (repo.radeon.com)... 23.53.42.179, 23.53.42.162, 2a02:26f0:7100::687e:25f2, ...
Connecting to repo.radeon.com (repo.radeon.com)|23.53.42.179|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 1734328 (1.7M) [application/octet-stream]
Saving to: ‘torchaudio-2.4.0+rocm6.3.4.git69d40773-cp312-cp312-linux_x86_64.whl’

torchaudio-2.4.0+rocm6.3.4.g 100%[==============================================>]   1.65M  6.45MB/s    in 0.3s

2025-05-16 11:37:04 (6.45 MB/s) - ‘torchaudio-2.4.0+rocm6.3.4.git69d40773-cp312-cp312-linux_x86_64.whl’ saved [1734328/1734328]

(Dreamy) meridia@TowerOfBabel:~/ComfyUI$ uv pip install torch-2.4.0+rocm6.3.4.git7cecbf6d-cp312-cp312-linux_x86_64.whl torchvision-0.19.0+rocm6.3.4.gitfab84886-cp312-cp312-linux_x86_64.whl torchaudio-2.4.0+rocm6.3.4.git69d40773-cp312-cp312-linux_x86_64.whl pytorch_triton_rocm-3.0.0+rocm6.3.4.git75cc27c2-cp312-cp312-linux_x86_64.whl
Using Python 3.12.10 environment at: Dreamy
Resolved 15 packages in 550ms
Prepared 15 packages in 9.45s
Installed 15 packages in 109ms
 + filelock==3.18.0
 + fsspec==2025.3.2
 + jinja2==3.1.6
 + markupsafe==3.0.2
 + mpmath==1.3.0
 + networkx==3.4.2
 + numpy==1.26.4
 + pillow==11.2.1
 + pytorch-triton-rocm==3.0.0+rocm6.3.4.git75cc27c2 (from file:///home/meridia/ComfyUI/pytorch_triton_rocm-3.0.0+rocm6.3.4.git75cc27c2-cp312-cp312-linux_x86_64.whl)
 + setuptools==80.7.1
 + sympy==1.12.1
 + torch==2.4.0+rocm6.3.4.git7cecbf6d (from file:///home/meridia/ComfyUI/torch-2.4.0+rocm6.3.4.git7cecbf6d-cp312-cp312-linux_x86_64.whl)
 + torchaudio==2.4.0+rocm6.3.4.git69d40773 (from file:///home/meridia/ComfyUI/torchaudio-2.4.0+rocm6.3.4.git69d40773-cp312-cp312-linux_x86_64.whl)
 + torchvision==0.19.0+rocm6.3.4.gitfab84886 (from file:///home/meridia/ComfyUI/torchvision-0.19.0+rocm6.3.4.gitfab84886-cp312-cp312-linux_x86_64.whl)
 + typing-extensions==4.13.2

(Dreamy) meridia@TowerOfBabel:~/ComfyUI$ location=$(pip show torch | grep Location | awk -F ": " '{print $2}')
cd ${location}/torch/lib/
rm libhsa-runtime64.so
(Dreamy) meridia@TowerOfBabel:~/ComfyUI/Dreamy/lib/python3.12/site-packages/torch/lib$ python -c 'import torch; print(torch.cuda.is_available())'
True
(Dreamy) meridia@TowerOfBabel:~/ComfyUI/Dreamy/lib/python3.12/site-packages/torch/lib$ python -c "import torch; print(f'device name [0]:', torch.cuda.get_device_name(0))"
device name [0]: AMD Radeon RX 7900 XTX
(Dreamy) meridia@TowerOfBabel:~/ComfyUI/Dreamy/lib/python3.12/site-packages/torch/lib$ python -m torch.utils.collect_env
<frozen runpy>:128: RuntimeWarning: 'torch.utils.collect_env' found in sys.modules after import of package 'torch.utils', but prior to execution of 'torch.utils.collect_env'; this may result in unpredictable behaviour
Collecting environment information...
PyTorch version: 2.4.0+rocm6.3.4.git7cecbf6d
Is debug build: False
CUDA used to build PyTorch: N/A
ROCM used to build PyTorch: 6.3.42134-a9a80e791

OS: Ubuntu 22.04.5 LTS (x86_64)
GCC version: (Ubuntu 11.4.0-1ubuntu1~22.04) 11.4.0
Clang version: Could not collect
CMake version: Could not collect
Libc version: glibc-2.35

Python version: 3.12.10 (main, Apr  9 2025, 04:03:51) [Clang 20.1.0 ] (64-bit runtime)
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
[pip3] numpy==1.26.4
[pip3] pytorch-triton-rocm==3.0.0+rocm6.3.4.git75cc27c2
[pip3] torch==2.4.0+rocm6.3.4.git7cecbf6d
[pip3] torchaudio==2.4.0+rocm6.3.4.git69d40773
[pip3] torchvision==0.19.0+rocm6.3.4.gitfab84886
[conda] Could not collect

```
</details><br>

Now run a series of tests to verify that ROCm acceleration is running

```
python -c 'import torch' 2> /dev/null && echo 'Success' || echo 'Failure'
python -c 'import torch; print(torch.cuda.is_available())'
python -c "import torch; print(f'device name [0]:', torch.cuda.get_device_name(0))"
python -m torch.utils.collect_env
sudo chmod +x test_pytorch_rocm.sh
./test_pytorch_rocm.sh
```

<details>
<summary>Test ROCm working</summary>

```
(Dreamy) meridia@TowerOfBabel:~/ComfyUI$ ls
Dreamy  test_pytorch_rocm.sh
(Dreamy) meridia@TowerOfBabel:~/ComfyUI$ sudo chmod +x test_pytorch_rocm.sh
[sudo] password for meridia:
(Dreamy) meridia@TowerOfBabel:~/ComfyUI$ ./test_pytorch_rocm.sh
Success
True
device name [0]: AMD Radeon RX 7900 XTX
<frozen runpy>:128: RuntimeWarning: 'torch.utils.collect_env' found in sys.modules after import of package 'torch.utils', but prior to execution of 'torch.utils.collect_env'; this may result in unpredictable behaviour
Collecting environment information...
PyTorch version: 2.4.0+rocm6.3.4.git7cecbf6d
Is debug build: False
CUDA used to build PyTorch: N/A
ROCM used to build PyTorch: 6.3.42134-a9a80e791

OS: Ubuntu 22.04.5 LTS (x86_64)
GCC version: (Ubuntu 11.4.0-1ubuntu1~22.04) 11.4.0
Clang version: Could not collect
CMake version: Could not collect
Libc version: glibc-2.35

Python version: 3.12.10 (main, Apr  9 2025, 04:03:51) [Clang 20.1.0 ] (64-bit runtime)
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
[pip3] numpy==1.26.4
[pip3] pytorch-triton-rocm==3.0.0+rocm6.3.4.git75cc27c2
[pip3] torch==2.4.0+rocm6.3.4.git7cecbf6d
[pip3] torchaudio==2.4.0+rocm6.3.4.git69d40773
[pip3] torchvision==0.19.0+rocm6.3.4.gitfab84886
[conda] Could not collect
```

</details><br>

At this point the local environment is set with ROCm acceleration and detects the GPU and pytorch.

## STEP 5 - Comfy UI

It's time to install ComfyUI. [Comfy UI repository](https://github.com/comfyanonymous/ComfyUI)

Since I setup the UV VENV first, I now I need to link git to ComfyUI repo and clone it.

```
cd
cd ComfyUI
git init
git remote add origin https://github.com/comfyanonymous/ComfyUI.git
git fetch origin
git reset --hard origin/master
```

Now I need to install the dependencies for ComfyUI

```
cat requirements.txt
uv pip install -r requirements.txt
```

Now I need to install ComfyUI manager extension, that will let you install more extensions from inside ComfyUI GUI

```
cd custom_nodes/
git clone https://github.com/ltdrdata/ComfyUI-Manager comfyui-manager
cd ..
```

Finally I can run ComfyUI, click on the URL to open the GUI on the browser of the host machine

```
python main.py
```

<details>
<summary>ComfyUI git clone logs</summary>

```
(Dreamy) meridia@TowerOfBabel:~$ cd ComfyUI/
(Dreamy) meridia@TowerOfBabel:~/ComfyUI$ git init
hint: Using 'master' as the name for the initial branch. This default branch name
hint: is subject to change. To configure the initial branch name to use in all
hint: of your new repositories, which will suppress this warning, call:
hint:
hint:   git config --global init.defaultBranch <name>
hint:
hint: Names commonly chosen instead of 'master' are 'main', 'trunk' and
hint: 'development'. The just-created branch can be renamed via this command:
hint:
hint:   git branch -m <name>
Initialized empty Git repository in /home/meridia/ComfyUI/.git/
(Dreamy) meridia@TowerOfBabel:~/ComfyUI$ git remote add origin https://github.com/comfyanonymous/ComfyUI.git
(Dreamy) meridia@TowerOfBabel:~/ComfyUI$ git fetch origin
remote: Enumerating objects: 20376, done.
remote: Counting objects: 100% (79/79), done.
remote: Compressing objects: 100% (62/62), done.
remote: Total 20376 (delta 48), reused 18 (delta 17), pack-reused 20297 (from 3)
Receiving objects: 100% (20376/20376), 68.78 MiB | 7.75 MiB/s, done.
Resolving deltas: 100% (13687/13687), done.
From https://github.com/comfyanonymous/ComfyUI
 * [new branch]        annoate_get_input_info      -> origin/annoate_get_input_info
 * [new branch]        api-nodes                   -> origin/api-nodes
 * [new branch]        base-path-env-var           -> origin/base-path-env-var
 * [new branch]        desktop-release-apr222025   -> origin/desktop-release-apr222025
 * [new branch]        desktop-release-apr242025   -> origin/desktop-release-apr242025
 * [new branch]        desktop-release-april172025 -> origin/desktop-release-april172025
 * [new branch]        desktop-release-may062025   -> origin/desktop-release-may062025
 * [new branch]        huchenlei-patch-1           -> origin/huchenlei-patch-1
 * [new branch]        master                      -> origin/master
 * [new branch]        model-paths-helper          -> origin/model-paths-helper
 * [new branch]        model_management            -> origin/model_management
 * [new branch]        model_manager               -> origin/model_manager
 * [new branch]        not_required_typing         -> origin/not_required_typing
 * [new branch]        prep-branch                 -> origin/prep-branch
 * [new branch]        required_frontend_ver       -> origin/required_frontend_ver
 * [new branch]        rh-uvtest                   -> origin/rh-uvtest
 * [new branch]        robinjhuang-patch-1         -> origin/robinjhuang-patch-1
 * [new branch]        socketless                  -> origin/socketless
 * [new branch]        video_output                -> origin/video_output
 * [new branch]        webfiltered-patch-1         -> origin/webfiltered-patch-1
 * [new branch]        weight-zipper               -> origin/weight-zipper
 * [new branch]        worksplit-multigpu          -> origin/worksplit-multigpu
 * [new branch]        worksplit-multigpu-loaders  -> origin/worksplit-multigpu-loaders
 * [new branch]        yo-add-precommit            -> origin/yo-add-precommit
 * [new branch]        yo-lora-trainer             -> origin/yo-lora-trainer
 * [new branch]        yoland68-more-owner-updates -> origin/yoland68-more-owner-updates
 * [new branch]        yoland68-patch-1            -> origin/yoland68-patch-1
 * [new branch]        yoland68-patch-2            -> origin/yoland68-patch-2
 * [new branch]        yoland68-patch-3            -> origin/yoland68-patch-3
 * [new tag]           latest                      -> latest
 * [new tag]           v0.0.1                      -> v0.0.1
 * [new tag]           v0.0.2                      -> v0.0.2
 * [new tag]           v0.0.4                      -> v0.0.4
 * [new tag]           v0.0.5                      -> v0.0.5
 * [new tag]           v0.0.6                      -> v0.0.6
 * [new tag]           v0.0.7                      -> v0.0.7
 * [new tag]           v0.0.8                      -> v0.0.8
 * [new tag]           v0.1.0                      -> v0.1.0
 * [new tag]           v0.1.1                      -> v0.1.1
 * [new tag]           v0.1.2                      -> v0.1.2
 * [new tag]           v0.1.3                      -> v0.1.3
 * [new tag]           v0.2.0                      -> v0.2.0
 * [new tag]           v0.2.1                      -> v0.2.1
 * [new tag]           v0.2.2                      -> v0.2.2
 * [new tag]           v0.2.3                      -> v0.2.3
 * [new tag]           v0.2.4                      -> v0.2.4
 * [new tag]           v0.2.5                      -> v0.2.5
 * [new tag]           v0.2.6                      -> v0.2.6
 * [new tag]           v0.2.7                      -> v0.2.7
 * [new tag]           v0.3.0                      -> v0.3.0
 * [new tag]           v0.3.1                      -> v0.3.1
 * [new tag]           v0.3.10                     -> v0.3.10
 * [new tag]           v0.3.11                     -> v0.3.11
 * [new tag]           v0.3.12                     -> v0.3.12
 * [new tag]           v0.3.13                     -> v0.3.13
 * [new tag]           v0.3.14                     -> v0.3.14
 * [new tag]           v0.3.15                     -> v0.3.15
 * [new tag]           v0.3.16                     -> v0.3.16
 * [new tag]           v0.3.17                     -> v0.3.17
 * [new tag]           v0.3.18                     -> v0.3.18
 * [new tag]           v0.3.19                     -> v0.3.19
 * [new tag]           v0.3.2                      -> v0.3.2
 * [new tag]           v0.3.20                     -> v0.3.20
 * [new tag]           v0.3.21                     -> v0.3.21
 * [new tag]           v0.3.22                     -> v0.3.22
 * [new tag]           v0.3.23                     -> v0.3.23
 * [new tag]           v0.3.24                     -> v0.3.24
 * [new tag]           v0.3.25                     -> v0.3.25
 * [new tag]           v0.3.26                     -> v0.3.26
 * [new tag]           v0.3.27                     -> v0.3.27
 * [new tag]           v0.3.28                     -> v0.3.28
 * [new tag]           v0.3.29                     -> v0.3.29
 * [new tag]           v0.3.3                      -> v0.3.3
 * [new tag]           v0.3.30                     -> v0.3.30
 * [new tag]           v0.3.31                     -> v0.3.31
 * [new tag]           v0.3.32                     -> v0.3.32
 * [new tag]           v0.3.33                     -> v0.3.33
 * [new tag]           v0.3.34                     -> v0.3.34
 * [new tag]           v0.3.4                      -> v0.3.4
 * [new tag]           v0.3.5                      -> v0.3.5
 * [new tag]           v0.3.6                      -> v0.3.6
 * [new tag]           v0.3.7                      -> v0.3.7
 * [new tag]           v0.3.8                      -> v0.3.8
 * [new tag]           v0.3.9                      -> v0.3.9
(Dreamy) meridia@TowerOfBabel:~/ComfyUI$ ls
Dreamy  test_pytorch_rocm.sh

(Dreamy) meridia@TowerOfBabel:~/ComfyUI$ git reset --hard origin/master
HEAD is now at 1c2d45d2 Fix typo in last PR. (#8144)
(Dreamy) meridia@TowerOfBabel:~/ComfyUI$ ls
CODEOWNERS       comfy_api           extra_model_paths.yaml.example  node_helpers.py   server.py
CONTRIBUTING.md  comfy_api_nodes     folder_paths.py                 nodes.py          test_pytorch_rocm.sh
Dreamy           comfy_execution     hook_breaker_ac10a0.py          notebooks         tests
LICENSE          comfy_extras        input                           output            tests-unit
README.md        comfyui_version.py  latent_preview.py               pyproject.toml    utils
api_server       cuda_malloc.py      main.py                         pytest.ini
app              custom_nodes        models                          requirements.txt
comfy            execution.py        new_updater.py                  script_examples
(Dreamy) meridia@TowerOfBabel:~/ComfyUI$ cat requirements.txt
comfyui-frontend-package==1.19.9
comfyui-workflow-templates==0.1.14
torch
torchsde
torchvision
torchaudio
numpy>=1.25.0
einops
transformers>=4.28.1
tokenizers>=0.13.3
sentencepiece
safetensors>=0.4.2
aiohttp>=3.11.8
yarl>=1.18.0
pyyaml
Pillow
scipy
tqdm
psutil

#non essential dependencies:
kornia>=0.7.1
spandrel
soundfile
av>=14.2.0
pydantic~=2.0
(Dreamy) meridia@TowerOfBabel:~/ComfyUI$ uv pip install -r requirements.txt
Using Python 3.12.10 environment at: Dreamy
Resolved 55 packages in 906ms
Prepared 40 packages in 26.53s
Installed 40 packages in 51ms
 + aiohappyeyeballs==2.6.1
 + aiohttp==3.11.18
 + aiosignal==1.3.2
 + annotated-types==0.7.0
 + attrs==25.3.0
 + av==14.3.0
 + certifi==2025.4.26
 + cffi==1.17.1
 + charset-normalizer==3.4.2
 + comfyui-frontend-package==1.19.9
 + comfyui-workflow-templates==0.1.14
 + einops==0.8.1
 + frozenlist==1.6.0
 + huggingface-hub==0.31.2
 + idna==3.10
 + kornia==0.8.1
 + kornia-rs==0.1.9
 + multidict==6.4.3
 + packaging==25.0
 + propcache==0.3.1
 + psutil==7.0.0
 + pycparser==2.22
 + pydantic==2.11.4
 + pydantic-core==2.33.2
 + pyyaml==6.0.2
 + regex==2024.11.6
 + requests==2.32.3
 + safetensors==0.5.3
 + scipy==1.15.3
 + sentencepiece==0.2.0
 + soundfile==0.13.1
 + spandrel==0.4.1
 + tokenizers==0.21.1
 + torchsde==0.2.6
 + tqdm==4.67.1
 + trampoline==0.1.2
 + transformers==4.51.3
 + typing-inspection==0.4.0
 + urllib3==2.4.0
 + yarl==1.20.0

(Dreamy) meridia@TowerOfBabel:~/ComfyUI$ cd custom_nodes/
(Dreamy) meridia@TowerOfBabel:~/ComfyUI/custom_nodes$ git clone https://github.com/ltdrdata/ComfyUI-Manager comfyui-manager
Cloning into 'comfyui-manager'...
remote: Enumerating objects: 19713, done.
remote: Counting objects: 100% (3782/3782), done.
remote: Compressing objects: 100% (543/543), done.
remote: Total 19713 (delta 3469), reused 3257 (delta 3239), pack-reused 15931 (from 3)
Receiving objects: 100% (19713/19713), 30.55 MiB | 7.31 MiB/s, done.
Resolving deltas: 100% (14576/14576), done.
(Dreamy) meridia@TowerOfBabel:~/ComfyUI/custom_nodes$ cd ..

```

</details><br>

At this point, the ComfyUI should be launched and open in your browser.

### Run ComfyUI

When you start the machine, the sequence of command to run comfyUI

```
wsl
su USER
cd
cd ComfyUI
source Dreamy/bin/activate
python main.py
```

To deactivate the UV venv

```
deactivate
```


### Comfy UI Folders

- ```ComfyUI\user\default\workflows``` Holds the workflows

- ```ComfyUI\models\checkpoints``` It's where SD15, SDXL, FLux checkpoints go

#### Copy workflows from host machine to WSL

When copying workflows to ```ComfyUI\user\default\workflows``` you might need to update permissions, or you get the following error:

```PermissionError: [Errno 13] Permission denied: '/home/meridia/ComfyUI/user/default/workflows/txt2img-flux.json'```


Commands

```
ls -l /home/meridia/ComfyUI/user/default/workflows/
sudo chown -R $(whoami) /home/meridia/ComfyUI/user/default/workflows/
ls -l /home/meridia/ComfyUI/user/default/workflows/
```

<details>
<summary>Workflow permission change logs</summary>

```
(Dreamy) meridia@TowerOfBabel:~/ComfyUI$ ls -l /home/meridia/ComfyUI/user/default/workflows/
total 360
-rw-rw-r-- 1 meridia meridia  4718 May 16 12:43 XXX.json
-rw-rw-r-- 1 meridia meridia  5222 May 17 10:30 ZBUG-vae-decode-adrenaline-crash.json
-rw-r--r-- 1 root    root    11226 May  7 19:48 img2img-background.json
-rw-r--r-- 1 root    root    12250 Mar 15 11:05 img2img-depth-sd15.json
-rw-r--r-- 1 root    root    20911 Mar 16 10:25 img2img-flux-caption.json
-rw-r--r-- 1 root    root    21973 Apr 20 10:58 img2img-flux-inpaint.json
-rw-r--r-- 1 root    root    18978 Mar 16 11:17 img2img-flux-tiled.json
-rw-r--r-- 1 root    root    28615 May 11 11:20 img2img-flux-v2.json
-rw-r--r-- 1 root    root    22843 May 10 13:31 img2img-hidream-dev-gguf.json
-rw-r--r-- 1 root    root     9145 Mar 15 10:11 img2img-inpaint-sd15.json
-rw-r--r-- 1 root    root    39500 Apr 13 14:58 img2img-outpaint-sd.json
-rw-r--r-- 1 root    root    14637 Apr 13 11:40 img2img-outpaint-sdxl.json
-rw-r--r-- 1 root    root    32858 May  9 14:43 img2stl-Hunyuan-background.json
-rw-r--r-- 1 root    root     3125 Apr 13 11:41 img2txt.json
-rw-r--r-- 1 root    root     5863 May  4 14:41 txt2audio-kokoro.json
-rw-r--r-- 1 root    root     4059 May  4 13:54 txt2audio-whisperspeech-clone.json
-rw-rw-r-- 1 meridia meridia  5111 May 17 11:00 txt2img-SD15-minimal.json
-rw-r--r-- 1 root    root    24325 May 11 11:11 txt2img-flux.json
-rw-r--r-- 1 root    root    24529 May 11 10:31 txt2img-gguf-flux.json
-rw-r--r-- 1 root    root    22843 May 10 13:30 txt2img-hidream-dev-gguf.json
(Dreamy) meridia@TowerOfBabel:~/ComfyUI$ sudo chown -R $(whoami) /home/meridia/ComfyUI/user/default/workflows/
[sudo] password for meridia:
(Dreamy) meridia@TowerOfBabel:~/ComfyUI$ ls -l /home/meridia/ComfyUI/user/default/workflows/
total 360
-rw-rw-r-- 1 meridia meridia  4718 May 16 12:43 XXX.json
-rw-rw-r-- 1 meridia meridia  5222 May 17 10:30 ZBUG-vae-decode-adrenaline-crash.json
-rw-r--r-- 1 meridia root    11226 May  7 19:48 img2img-background.json
-rw-r--r-- 1 meridia root    12250 Mar 15 11:05 img2img-depth-sd15.json
-rw-r--r-- 1 meridia root    20911 Mar 16 10:25 img2img-flux-caption.json
-rw-r--r-- 1 meridia root    21973 Apr 20 10:58 img2img-flux-inpaint.json
-rw-r--r-- 1 meridia root    18978 Mar 16 11:17 img2img-flux-tiled.json
-rw-r--r-- 1 meridia root    28615 May 11 11:20 img2img-flux-v2.json
-rw-r--r-- 1 meridia root    22843 May 10 13:31 img2img-hidream-dev-gguf.json
-rw-r--r-- 1 meridia root     9145 Mar 15 10:11 img2img-inpaint-sd15.json
-rw-r--r-- 1 meridia root    39500 Apr 13 14:58 img2img-outpaint-sd.json
-rw-r--r-- 1 meridia root    14637 Apr 13 11:40 img2img-outpaint-sdxl.json
-rw-r--r-- 1 meridia root    32858 May  9 14:43 img2stl-Hunyuan-background.json
-rw-r--r-- 1 meridia root     3125 Apr 13 11:41 img2txt.json
-rw-r--r-- 1 meridia root     5863 May  4 14:41 txt2audio-kokoro.json
-rw-r--r-- 1 meridia root     4059 May  4 13:54 txt2audio-whisperspeech-clone.json
-rw-rw-r-- 1 meridia meridia  5111 May 17 11:00 txt2img-SD15-minimal.json
-rw-r--r-- 1 meridia root    24325 May 11 11:11 txt2img-flux.json
-rw-r--r-- 1 meridia root    24529 May 11 10:31 txt2img-gguf-flux.json
-rw-r--r-- 1 meridia root    22843 May 10 13:30 txt2img-hidream-dev-gguf.json
```

</details><br>


### Test ComfyuUI - First Image Generation

In order to do a test, you need a model. The tutorial should open, select the basic txt2img SD1.5 workflow.

You need to copy a SD1.5 model in the checkpoint folder, and in the dropdown select the model in the model loader node. If the drop down doesn't refresh, from the model/checkpoint GUI tab you can refresh and drag from there.

Have faith, Flux took 340s before it did something on my machine. The first run is very slow, the second run should accelerate properly. SD1.5 should take only a few seconds at 512x512 resolution with 20 step euler


<details>
<summary>Run first basic workflow</summary>

```
(Dreamy) meridia@TowerOfBabel:~/ComfyUI$ python main.py
[START] Security scan
[DONE] Security scan
## ComfyUI-Manager: installing dependencies. (GitPython)
## ComfyUI-Manager: installing dependencies done.
** ComfyUI startup time: 2025-05-16 12:29:29.965
** Platform: Linux
** Python version: 3.12.10 (main, Apr  9 2025, 04:03:51) [Clang 20.1.0 ]
** Python executable: /home/meridia/ComfyUI/Dreamy/bin/python
** ComfyUI Path: /home/meridia/ComfyUI
** ComfyUI Base Folder Path: /home/meridia/ComfyUI
** User directory: /home/meridia/ComfyUI/user
** ComfyUI-Manager config path: /home/meridia/ComfyUI/user/default/ComfyUI-Manager/config.ini
** Log path: /home/meridia/ComfyUI/user/comfyui.log

Prestartup times for custom nodes:
   8.5 seconds: /home/meridia/ComfyUI/custom_nodes/comfyui-manager

Checkpoint files will always be loaded safely.
Total VRAM 24514 MB, total RAM 32012 MB
pytorch version: 2.4.0+rocm6.3.4.git7cecbf6d
AMD arch: gfx1100
Set vram state to: NORMAL_VRAM
Device: cuda:0 AMD Radeon RX 7900 XTX : native
Using sub quadratic optimization for attention, if you have memory or speed issues try using: --use-split-cross-attention
Python version: 3.12.10 (main, Apr  9 2025, 04:03:51) [Clang 20.1.0 ]
ComfyUI version: 0.3.34
ComfyUI frontend version: 1.19.9
[Prompt Server] web root: /home/meridia/ComfyUI/Dreamy/lib/python3.12/site-packages/comfyui_frontend_package/static
### Loading: ComfyUI-Manager (V3.32.2)
[ComfyUI-Manager] network_mode: public
### ComfyUI Version: v0.3.34-18-g1c2d45d2 | Released on '2025-05-15'

Import times for custom nodes:
   0.0 seconds: /home/meridia/ComfyUI/custom_nodes/websocket_image_save.py
   0.1 seconds: /home/meridia/ComfyUI/custom_nodes/comfyui-manager

Starting server

To see the GUI go to: http://127.0.0.1:8188
[ComfyUI-Manager] default cache updated: https://raw.githubusercontent.com/ltdrdata/ComfyUI-Manager/main/model-list.json
[ComfyUI-Manager] default cache updated: https://raw.githubusercontent.com/ltdrdata/ComfyUI-Manager/main/alter-list.json
[ComfyUI-Manager] default cache updated: https://raw.githubusercontent.com/ltdrdata/ComfyUI-Manager/main/github-stats.json
[ComfyUI-Manager] default cache updated: https://raw.githubusercontent.com/ltdrdata/ComfyUI-Manager/main/custom-node-list.json
[ComfyUI-Manager] default cache updated: https://raw.githubusercontent.com/ltdrdata/ComfyUI-Manager/main/extension-node-map.json
FETCH ComfyRegistry Data: 5/85
FETCH ComfyRegistry Data: 10/85
FETCH ComfyRegistry Data: 15/85
FETCH ComfyRegistry Data: 20/85
FETCH ComfyRegistry Data: 25/85
FETCH ComfyRegistry Data: 30/85
FETCH ComfyRegistry Data: 35/85
FETCH ComfyRegistry Data: 40/85
FETCH ComfyRegistry Data: 45/85
FETCH ComfyRegistry Data: 50/85
FETCH ComfyRegistry Data: 55/85
FETCH ComfyRegistry Data: 60/85
FETCH ComfyRegistry Data: 65/85
FETCH ComfyRegistry Data: 70/85
FETCH ComfyRegistry Data: 75/85
FETCH ComfyRegistry Data: 80/85
FETCH ComfyRegistry Data: 85/85
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
Requested to load SD1ClipModel
loaded completely 16404.409375 235.84423828125 True
Requested to load BaseModel
loaded completely 16033.99384765625 1639.406135559082 True
100%|███████████████████████████████████████████████████████████████████████████████| 20/20 [00:21<00:00,  1.09s/it]
Requested to load AutoencoderKL
loaded completely 12822.92626953125 319.11416244506836 True
Prompt executed in 59.39 seconds
got prompt
100%|███████████████████████████████████████████████████████████████████████████████| 20/20 [00:01<00:00, 19.78it/s]
Prompt executed in 1.18 seconds
got prompt
Requested to load BaseModel
loaded completely 14021.85810546875 1639.406135559082 True
100%|███████████████████████████████████████████████████████████████████████████████| 20/20 [00:01<00:00, 19.69it/s]
Requested to load AutoencoderKL
loaded completely 12760.21240234375 319.11416244506836 True
Prompt executed in 1.59 seconds
```

</details><br>


## STEP 6A - Move Model folder

It's a pain to move models inside the WSL EXT4 partition. Instead I set comfyUI to fetch models from the host machine.

There is a file called ```extra_model_paths.yaml.example```

Rename it to ```extra_model_paths.yaml``` to activate it

Inside, link to the folder in the host machine

#Rename this to extra_model_paths.yaml and ComfyUI will load it

```
comfyui:
     base_path: /mnt/f/comfyui-models
     # You can use is_default to mark that these folders should be listed first, and used as the default dirs for eg downloads
     #is_default: true
     checkpoints: checkpoints/
     clip: clip/
     clip_vision: clip_vision/
     text_encoders: text_encoders/
     configs: configs/
     controlnet: controlnet/
     diffusion_models: |
                  diffusion_models
                  unet
     embeddings: embeddings/
     loras: loras/
     upscale_models: upscale_models/
     vae: vae/
```

<details>
<summary>ComfyUI loading from host folder logs</summary>

```
meridia@TowerOfBabel:~/ComfyUI$ source Dreamy/bin/activate
(Dreamy) meridia@TowerOfBabel:~/ComfyUI$ python main.py
Adding extra search path checkpoints /mnt/f/comfyui-models/checkpoints
Adding extra search path clip /mnt/f/comfyui-models/clip
Adding extra search path clip_vision /mnt/f/comfyui-models/clip_vision
Adding extra search path configs /mnt/f/comfyui-models/configs
Adding extra search path controlnet /mnt/f/comfyui-models/controlnet
Adding extra search path diffusion_models /mnt/f/comfyui-models/diffusion_models
Adding extra search path diffusion_models /mnt/f/comfyui-models/unet
Adding extra search path embeddings /mnt/f/comfyui-models/embeddings
Adding extra search path loras /mnt/f/comfyui-models/loras
Adding extra search path upscale_models /mnt/f/comfyui-models/upscale_models
Adding extra search path vae /mnt/f/comfyui-models/vae
[START] Security scan
[DONE] Security scan
## ComfyUI-Manager: installing dependencies done.
** ComfyUI startup time: 2025-05-16 13:22:54.410
** Platform: Linux
** Python version: 3.12.10 (main, Apr  9 2025, 04:03:51) [Clang 20.1.0 ]
** Python executable: /home/meridia/ComfyUI/Dreamy/bin/python
** ComfyUI Path: /home/meridia/ComfyUI
** ComfyUI Base Folder Path: /home/meridia/ComfyUI
** User directory: /home/meridia/ComfyUI/user
** ComfyUI-Manager config path: /home/meridia/ComfyUI/user/default/ComfyUI-Manager/config.ini
** Log path: /home/meridia/ComfyUI/user/comfyui.log

Prestartup times for custom nodes:
   0.7 seconds: /home/meridia/ComfyUI/custom_nodes/comfyui-manager

Checkpoint files will always be loaded safely.
Total VRAM 24514 MB, total RAM 32012 MB
pytorch version: 2.4.0+rocm6.3.4.git7cecbf6d
AMD arch: gfx1100
Set vram state to: NORMAL_VRAM
Device: cuda:0 AMD Radeon RX 7900 XTX : native
Using sub quadratic optimization for attention, if you have memory or speed issues try using: --use-split-cross-attention
Python version: 3.12.10 (main, Apr  9 2025, 04:03:51) [Clang 20.1.0 ]
ComfyUI version: 0.3.34
ComfyUI frontend version: 1.19.9
[Prompt Server] web root: /home/meridia/ComfyUI/Dreamy/lib/python3.12/site-packages/comfyui_frontend_package/static
### Loading: ComfyUI-Manager (V3.32.2)
[ComfyUI-Manager] network_mode: public
### ComfyUI Version: v0.3.34-18-g1c2d45d2 | Released on '2025-05-15'

Import times for custom nodes:
   0.0 seconds: /home/meridia/ComfyUI/custom_nodes/websocket_image_save.py
   0.0 seconds: /home/meridia/ComfyUI/custom_nodes/comfyui-manager

Starting server

To see the GUI go to: http://127.0.0.1:8188
[ComfyUI-Manager] default cache updated: https://raw.githubusercontent.com/ltdrdata/ComfyUI-Manager/main/alter-list.json
[ComfyUI-Manager] default cache updated: https://raw.githubusercontent.com/ltdrdata/ComfyUI-Manager/main/model-list.json
[ComfyUI-Manager] default cache updated: https://raw.githubusercontent.com/ltdrdata/ComfyUI-Manager/main/github-stats.json
[ComfyUI-Manager] default cache updated: https://raw.githubusercontent.com/ltdrdata/ComfyUI-Manager/main/extension-node-map.json
[ComfyUI-Manager] default cache updated: https://raw.githubusercontent.com/ltdrdata/ComfyUI-Manager/main/custom-node-list.json
FETCH ComfyRegistry Data: 5/85
FETCH ComfyRegistry Data: 10/85
FETCH ComfyRegistry Data: 15/85
FETCH ComfyRegistry Data: 20/85
FETCH ComfyRegistry Data: 25/85
FETCH ComfyRegistry Data: 30/85
FETCH ComfyRegistry Data: 35/85
FETCH ComfyRegistry Data: 40/85
FETCH ComfyRegistry Data: 45/85
FETCH ComfyRegistry Data: 50/85
FETCH ComfyRegistry Data: 55/85
FETCH ComfyRegistry Data: 60/85
FETCH ComfyRegistry Data: 65/85
FETCH ComfyRegistry Data: 70/85
FETCH ComfyRegistry Data: 75/85
FETCH ComfyRegistry Data: 80/85
FETCH ComfyRegistry Data: 85/85
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
Requested to load SD1ClipModel
loaded completely 16270.0734375 235.84423828125 True
Requested to load BaseModel
loaded completely 15900.16181640625 1639.406135559082 True
100%|███████████████████████████████████████████████████████████████████████████████| 20/20 [00:01<00:00, 15.51it/s]
Requested to load AutoencoderKL
loaded completely 12717.25048828125 319.11416244506836 True
Prompt executed in 3.70 seconds
got prompt
100%|███████████████████████████████████████████████████████████████████████████████| 20/20 [00:01<00:00, 19.20it/s]
Prompt executed in 1.20 seconds
```

</details><br>


## STEP 6B - Setup Backup

The reason I went to such length to setup the portable environment as it is, is to allow backups of the FULL ComfyUI.

Zip doesn't work, it's bricked, somehow :O

Backup point

```
pip freeze > 2025-05-17b-requirements.txt
```

Restore point
NOTE: you need to rebuild the nodes, so it takes forever the first run of the nodes.

```
rm -rf Dreamy
uv venv Dreamy
source Dreamy/bin/activate
uv pip install -r 2025-05-17-requirements.txt

location=$(pip show torch | grep Location | awk -F ": " '{print $2}')
cd ${location}/torch/lib/
rm libhsa-runtime64.so*
cd
cd ComfyUI

python -c 'import torch' 2> /dev/null && echo 'Success' || echo 'Failure'
python -c 'import torch; print(torch.cuda.is_available())'
python -c "import torch; print(f'device name [0]:', torch.cuda.get_device_name(0))"
python -m torch.utils.collect_env
```

## STEP 6C - Safe Pip

Packages really want to brick ROCm with all their willpower by overwriting torch with an incompatible version that will never work. e.g. it happened with Florence2

uv let you specify a constraint file, to add some sacred blessed dependencies that takes precedence, this should stop pip from constantly bricking ROCm?



if created outside WSL needs ownership from user
```
cd 
cd ComfyUI
sudo chown -R $(whoami) constraint.txt
cat constraint.txt
```

constraint.txt
```
pytorch-triton-rocm @ file:///home/meridia/ComfyUI/pytorch_triton_rocm-3.0.0+rocm6.3.4.git75cc27c2-cp312-cp312-linux_x86_64.whl
torch @ file:///home/meridia/ComfyUI/torch-2.4.0+rocm6.3.4.git7cecbf6d-cp312-cp312-linux_x86_64.whl
torchaudio @ file:///home/meridia/ComfyUI/torchaudio-2.4.0+rocm6.3.4.git69d40773-cp312-cp312-linux_x86_64.whl
torchvision @ file:///home/meridia/ComfyUI/torchvision-0.19.0+rocm6.3.4.gitfab84886-cp312-cp312-linux_x86_64.whl
```

manually git clone the node without installing requirements.txt, and 

```
cd
cd ComfyUI
git clone https://github.com/kijai/ComfyUI-Florence2.git
cd ComfyUI-Florence2
uv pip install -r requirements.txt --constraint ~meridia/ComfyUI/constraint.txt
cd 
cd ComfyUI
```































