# Fiber

Source-binary patch presence test system. Given a target software binary (Android kernel image) and a security patch (in source), Fiber can check if the patch has been applied in the target binary. 

The design, implementation, and more details of fiber can be found in our research paper:

*Hang Zhang and Zhiyun Qian. "[Precise and accurate patch presence test for binaries.](https://www.usenix.org/system/files/conference/usenixsecurity18/sec18-zhang.pdf)" USENIX Security 2018.*

## Background

Surprisingly often, open source components get integrated into larger software and eventually released in binary forms only. Take Linux as an example: IoT devices, cars, voting machines, and Android smartphones all use some derived version of Linux kernel (possibly customized). Unfortunately, as end users or independent researchers, we don't get access to the source code. Yes, Android device vendors (e.g., Samsung, Xiaomi) do not necessarily release the complete kernel source history (no individual commit is given) and most OTA updates do not come with corresponding source code. This makes it hard to check if any security patch has been applied. Another example, car manufacturers often take binaries from third-parties (infotainment system) who in turn integrate other open source components. Car manufacturers may want to ensure the security of these binaries but again don't have access to source code. This is what Fiber is designed for.

## Key insight

Checking the presence of a patch in binary is inherently challenging because the patch can be small (affecting only few instructions) which can be burried/obscured by other non-security updates to the codebase afterwards. Compiler configs also drastically influence the compiled binary instructions. Our insight is that if the patch source is available (which is the case for open source software such as Linux), then we can take advantage of it to extract a proper signature based on how control and data flow are perturbed because of the patch.

## 0x0 A Simple Workflow

We will briefly explain fiber's workflow here with the examples under *examples* folder. Basically, we prepared some security patches under *examples/patches* folder, our reference kernel *examples/imgs/angler_img_20170513* adopts all these patches but *examples/imgs/angler_img_20160513* does not. We then generate binary signatures (stored in *examples/sigs*) for these patches and then use them to test the patch presence for the target kernel (*examples/imgs/image-G9300-160909*, Samsung S7 kernel released in 2016/09/09). The test result can be found in *examples/match_res_image-G9300-160909_1528162862_m1* where **P** means the related patch has been adopted and **N** otherwise.

**Step 0**  
Use the picker (section 0x2) to analyze the patches and the reference source code in order to pick out
most suitable change sites introduced by the patch. Our reference kernel source code (patched) is [kernel-msm-src](https://android.googlesource.com/kernel/msm/) with commit cedc139f61870d3f4f8a80f9030b0836b56e2204.

**Step 1**  
Translate each change site identified by the picker to a binary signature with the translator (section 0x3).

**Step 2**  
Validate the generated binary signatures by trying to match them in both reference unpatched and patched kernels.
This can be done with the matcher in mode 0,2 (section 0x4). 

**Step 3**  
Use the valid binary signatures to do patch presence test for the target kernels with the matcher's mode 1. (section 0x4)

## 0x1 Environment Setup

At first we need to install virtualenvwrapper for python, please follow the [official installation instructions](http://virtualenvwrapper.readthedocs.io/en/latest/install.html).
Before continuing, plz make sure that virtualenvwrapper is correctly installed and you can execute its commands:  
`~$ workon`  
NOTE, **plz don't use *sudo* from here on unless explicitly prompted**.  
`~$ git clone https://github.com/fiberx/fiber.git`  
`~$ cd fiber`  
Setup the angr development environment specifically crafted for fiber:  
`~/fiber$ ./setup_angr_env.sh [dir_name] [venv_name]`

- *dir_name*:
Specify a directory and we'll put angr related files there.
- *venv_name*:
We will use a virtual python environment for fiber, specify its name here.

**NOTE**, should you be prompted to enter username/password for GitHub accounts during the execution of above script,
plz simply ignore that and just type *"Enter"*.  
It's time to install some required packages in the virtual env:  
`~/fiber$ workon [venv_name]`  
`(venv_name)~/fiber$ ./install_pkgs.sh`  
Now you are ready to use fiber scripts.  
As a test, you can run below command to see whether the signature can be shown w/o issues:  
`(venv_name)~/fiber$ python test_sig.py examples/sigs/CVE-2016-3866-sig-0`

**Before running any fiber scripts, remember to switch the virtual environment at first**:  
`workon [venv_name]`  
To exit the virtual environment:  
`deactivate`  

## 0x2 Picker

`(venv_name)~/fiber$ python pick_sig.py [patch_list] [reference kernel source] [output_file] [compiled reference kernel(binary image, symbol table, vmlinux, debuginfo)]`

**Params**:  

- *patch_list*:
A file where each line specifies the path to a patch file. (eg. *examples/patch_list*)
- *reference kernel source*:
The path to the reference kernel source code root folder.
- *output_file*:
The results will be stored there. This file can then be used as "ext_list" for the translator (eg. *examples/ext_list*)
- *compiled reference kernel*:
To solve function inline probelm, we need debug information of compiled reference kernel. This directory should include binary image, symbol table, debuginfo(extracted from vmlinux)
For example,examples/Test/refkernel
- *boot*:ref_kernel_image. The reference kernel zImage from which the binary signature will be generated.
- *System.map*:ref_kernel_symbol_table
The symbol table for the reference kernel zImage (eg. examples/imgs/angler_img_20170513.sym). Since source code is available for the reference kernel, we can use "System.map" generated by the compiler.
- *vmlinux*:ref_kernel_vmlinux
The "vmlinux" (generated by the compiler) for the reference kernel. We need this because it contains fine-grained DWARF debug information that can help to map source code lines to binary instructions. (Due to size limit we haven't included the vmlinux file for reference kernel in *example* folder, you can download it from here [*angler_img_20170513_vmlinux*](https://drive.google.com/file/d/1r16Q2yy-zpALJJWZg1gZBQNAplwDwdWC/view?usp=sharing))
- *tmp_o*: debug info
The debug info file extracted from vmlinux

**Output**:ext_list which stores the change site information
For example, examples/Test/ext_list
Besides the *extlist*, if necessary, the picker will generate another file *output_file_fail* which records the patches for which the picker fails to identify any suitable change sites. The possible reasons include: (1) the picker cannot match/locate the patch in the reference kernel source (2) the function changed by the patch cannot be found in the symbol tables (in this case the function will be inlined in the binary, currently we are unable to locate an inlined function in the binary.) (3) the patch has no suitable change sites to translate (eg. only change some variable definitions.) (4) fiber's own issues when doing the signature matching.

## 0x3 Translator

`(venv_name)~/fiber$ python ext_sig.py [compiled reference kernel(binary image......)] [ext_list dir] [optional: cfgfast]

**Params**:  

- *compiled reference kernel*: the same as 0x2
- *ext_list dir*: the directory containing extlist
This file is generated by the picker. Each line specifies the information required to translate a binary signature. (eg. *examples/ext_list*)
- *output_dir*: we will store signatures ext_list dir/sigs. Thus we don't need to specify the output path

**NOTE**:  
The translator needs to use *addr2line* to read DWARF debug information, whose path is currently hardcoded in *ext_sig.py* (ADDR2LINE = '/path/to/addr2line'), plz make it right before executing this script.

## 0x4 Matcher

`(venv_name)~/fiber$ python match_sig.py [ext_list dir] [mode] [target kernel path]`  

**Params**:  

- *ext_list dir*: as show in 0x3, this directory contains the signatures

- *target kernel path*
the kernel  that needs to be tested. (eg. *examples/imgs/Test/targetkernel*)
**NOTE**:
we provide *tools/ext_sym* to extract the embedded symbol table from aarch64 linux kernel image (see the usage of this tool in section 0x5). Most Android kernel images should have such an embedded

- *mode*:
'0': match signatures with patched kernel
'1': match signatures with target kernel
'2': match signatures with unpatched kernel

'all': execute mode 0, mode 2, mode 1 sequencely

**NOTE**:
unpatched kernel is used for filtering signatures. This process will be done automatically at the end of mode 2.

**Output**:  
The matcher will output the results to both the screen and an automatically generated file *target kernel path/matchresults*.

## 0x5 Auxilary Tools

### 5.1 tools/ext_sym

To extract the embedded symbol table from a kernel zImage.  
`~/fiber$ tools/ext_sym [image] [idc](optional) > output`  
**Params**:

- *image*:
The kernel zImage
- *idc*:
If you want, you can use `./ext_sym [image] 1` to generate an ".idc" file which is an IDA Pro script that can apply the symbol names when disassembling the kernel image.
Otherwise, a normal symbol table will be generated. (The format is like *System.map* file generated by the compiler).

### 5.3 test_sig.py

To view a binary signature.  
`(venv_name)~/fiber$ python test_sig.py [bin_sig]`  
**Params**:  
- *bin_sig*:
A binary signature generated by the translator. (eg. *examples/sigs/CVE-2016-3866-sig-0*)  

**Output**:  
The strcuture, node, root instructions and formulas of the signature will be shown on the screen.
