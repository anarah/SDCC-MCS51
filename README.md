# SDCC-MCS51

## Eclipse + SDCC 
环境安装教程https://wenku.baidu.com/view/f0e53e62a88271fe910ef12d2af90242a895abab.html

## Eclipse 配置 工程
->properties->C/C++Build->Settings->Tool Settings->SDCC Compiler->Command:  sdcc -mmcs51 -c
->properties->C/C++Build->Settings->Tool Settings->SDCC Compiler->Symbols: 全局define的内容
->properties->C/C++Build->Settings->Tool Settings->SDCC Compiler->Directories：Include paths
->properties->C/C++Build->Settings->Tool Settings->SDCC Compiler->Memory Options->Memory Model: Large 或 Huge    区别可查<SDCC Compiler User Guide>
->properties->C/C++Build->Settings->Tool Settings->SDCC Linker->Command:
sdcc --model-large "-Wl -r -b BANK1=0x4000" "-Wl -r -b BANK2=0x10000" --xram-loc 0x0000  --xram-size 0xC000如果只用BANK1，需要删除 "-Wl -r -b BANK2=0x10000"
->properties->C/C++Build->Settings->Tool Settings->SDCC Assembler->Command: sdas8051 -l

->properties->C/C++Build->Settings->Artifact name: image
->properties->C/C++Build->Settings->Artifact extension: hex

->properties->C/C++Build->Tool Chain Editor : Current toolchain: SDCC Tool Chain   Current builder: CDT Internal Builder


## <SDCC Compiler User Guide>
    P62 4.1.3 Bankswitching (a.k.a. code banking)
	
### 4.1.3.1 Hardware

+++++++++++++++++++++++++++++++++++
8000-FFFF | bank1 | bank2 | bank3 |
+++++++++++++++++++++++++++++++++++
0000-7FFF | common 
+++++++++++++++++++++++++++++++++++
     SiLabs C8051F120 example
     
Usually the hardware uses some sfr (an output port or an internal sfr) to select a bank and put it in the banked area of the memory map. The selected bank usually becomes active immediately upon assignment to this sfr and when running inside a bank it will switch out this code it is currently running. Therefor you cannot jump or call directly from one bank to another and need to use a so-called trampoline in the common area. For SDCC an example trampoline is in crtbank.asm and you may need to change it to your 8051 derivative or schematic. The presented code is written for the C8051F120.
When calling a banked function SDCC will put the LSB of the functions address in register R0, the MSB in R1 and the bank in R2 and then call this trampoline __sdcc_banked_call. The current selected bank is saved on the stack, the new bank is selected and an indirect jump is made. When the banked function returns it jumps to __sdcc_banked_ret which restores the previous bank and returns to the caller.

### 4.1.3.2 Software
When writing banked software using SDCC you need to use some special keywords and options. You also need to take over a bit of work from the linker.
To create a function that can be called from another bank it requires the keyword __banked. The caller must see this in the prototype of the callee and the callee needs it for a proper return. Called functions within the same bank as the caller do not need the __banked keyword nor do functions in the common area. Beware: SDCC
does not know or check if functions are in the same bank. This is your responsibility!
Normally all functions you write end up in the segment CSEG. If you want a function explicitly to reside in the common area put it in segment HOME. This applies for instance to interrupt service routines as they should not be banked.
Functions that need to be in a switched bank must be put in a named segment. The name can be mostly anything up to eight characters (e.g. BANK1). To do this you either use --codeseg BANK1 (See 3.3.4) on the command line when compiling or #pragma codeseg BANK1 (See 3.16) at the top of the C source file. The segment name always applies to the whole source file and generated object so functions for different banks need to be defined in different source files.
When linking your objects you need to tell the linker where to put your segments. To do this you use the following command line option to SDCC: -Wl-b BANK1=0x18000 (See 3.3.5). This sets the virtual start address of this segment. It sets the banknumber to 0x01 and maps the bank to 0x8000 and up. The linker will not check for overflows, again this is your responsibility.

★★需要放bank的.c文件，文件名需要排序在后
..\SDCC\lib\src\mcs51文件夹下的crtbank.asm拷贝至工作目录，修改该文件内容，功能类似于keil的L51_BANK.A51

要创建一个可以从另一个bank调用的函数，它需要关键字__banked，发起调用的函数也需要。（函数声明时也需要）
需要放进BANK的.c文件需要在c源文件的顶部使用#pragma codeseg BANK1。段名称总是应用于整个源文件和生成的对象，因此需要在不同的源文件中定义不同bank的函数。
	xx.c :
	#pragma codeseg BANK2
	
	#include "reg320.h"
	#include "common_types.h"
	
	uint16 adda (uint16 source) __banked
	{
	       return source + 3;
	}


	xx.h :
	
	uint16 adda(uint16 source) __banked;
