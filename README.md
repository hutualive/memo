# freertos kernel interrupt priority on arm cortex-m3/4/7
1. priority preempt bit is vendor dependent: TI - 3 bit, NXP - 5 bit, ST - 4 bit  
2. configMAX_SYSCALL_INTERRUPT_PRIORITY: boundry of maskable interrupts by RTOS critical sections  
3. assign all priority bits as preempt bits: NVIC_PriorityGroupConfig(NVIC_PriorityGroup_4) before RTOS started   
4. configMAX_SYSCALL_INTERRUPT_PRIORITY and configKERNEL_INTERRUPT_PRIORITY(255 - 1111 1111 - lowest priority) already shifted to the most significant bits of the priority byte  
5. ISR use RTOS API must have lower logical priority(higher numeric value) than configMAX_SYSCALL_INTERRUPT_PRIORITY  
6. RTOS kernel create a critical section by writing the configMAX_SYSCALL_INTERRUPT_PRIORITY value into the ARM Cortex-M BASEPRI register(so it's value must not be set to 0)  

# general memo

2020/03/15
====================
make - automate the running of shell commands and help with repetitive tasks  --> makefile - shell commands

for larger project, use modern build system like cmake
-hundreds of target
-configure step is desired
-use by professional
-easier to debug
-cross platform

makefile order: GNUmakefile --> makefile --> Makefile

// cd to directory then invoke
make -C some/sub/directory

// 1.5 x cores
make -j 6

// clean and clear output option
--output-sync=recurse

// recursive expansion  --> expression is evaluated when the variable is used
variable = expression

// simple expansion  --> expression is expanded at the time of assignment
variable := expression

// only assign the value if it doesn't already have a value --> can override later on
variable ?= expression

// append variable
CFLAGS += -Wall

// passing extra flags from environment
CFLAGS = '-Werror=conversion -Werror=double-promotion'

// target specific variables

// implicit variables
$(CC) - the c compiler(gcc)
$(AR) - archive program(ar)
$CFLAGS) - flags for c compiler

// automatic variables
$@ - the target name
$^ - all the prerequisites
$< - 1st prerequisite

%.o: %.c
	$(CC) -c $< -o $@

// target
target: prerequisite
	recipe

target almost always name files(except for phony targets) as make use last modified time to track if a target is newer than its prerequisites and whether it needs to be rebuilt

// all target
.PHONY: all

all: foo bar.a

// the 'all' rule that build and test  --> make it list first to make it the default rule
.PHONY: all
all: build test

// make consider .PHONY targets are always out-of-date and rebuild

// triger re-compilation if header file changed  --> done with -M compiler flag
for gcc/clang  --> output a .d file then import with Make include directive

// emitting the dependency tracking file
DEPFLAGS = -MMD -MP -MF $<.d

test.o: test.c
	$(CC) $(DEPFLAGS) $< -c $@

// bring in the prerequisites by including all the .d files. prefix the line with '-' to prevent error if such file not exist
-include $(wildcard *.d)

// order-only prerequisites  --> only build if they don't exist
OUTPUT_DIR = build

// anything right of the | pipe is considered order-only  --> emitting files to output directory will not trigger a rebuild
$(OUTPUT_DIR)/test.o: test.c | $(OUTPUT_DIR)
	$(CC) -c $^ -o $@

// rule to make the directory
$(OUTPUT_DIR):
	mkdir -p $@


// if any line of recipe return a non-zero exit code, Make will terminate and print an error message  --> ignore by prefixing '-'
.PHONY: clean
clean: 
	// don't care if rm fails
	-rm -r ./build

// prefix a recipe line with @ will disable echo

// build-in functions
FILES=$(wildcard *.c)

# you can combine function calls; here we strip the suffix off of $(FILES) with
# the $(basename) function, then add the .o suffix
O_FILES=$(addsuffix .o,$(basename $(FILES)))

# note that the GNU Make Manual suggests an alternate form for this particular
# operation:
O_FILES=$(FILES:.c=.o)

// user-defined functions
# recursive wildcard (use it instead of $(shell find . -name '*.c'))
# taken from https://stackoverflow.com/a/18258352
rwildcard=$(foreach d,$(wildcard $1*),$(call rwildcard,$d/,$2) $(filter $(subst *,%,$2),$d))

C_FILES = $(call rwildcard,.,*.c)

// include directive
include sources.mk

OBJECT_FILES = $(SOURCE_FILES:.c=.o)

%.o: %.c
	$(CC) -c $^ -o $@

// sub-make
somelib.a:
	$(MAKE) -C path/to/somelib/directory

// meta-programming with eval  --> avoid using as it's confusing
# generate rules for xml->json in some weird world
FILES = $(wildcard inputfile/*.xml)

# create a user-defined function that generates rules
define GENERATE_RULE =
$(eval
# prereq rule for creating output directory
$(1)_OUT_DIR = $(dir $(1))/$(1)_out
$(1)_OUT_DIR:
	mkdir -p $@

# rule that calls a script on the input file and produces $@ target
$(1)_OUT_DIR/$(1).json: $(1) | $(1)_OUT_DIR
	./convert-xml-to-json.sh $(1) $@
)

# add the target to the all rule
all: $(1)_OUT_DIR/$(1).json
endef

# produce the rules
.PHONY: all
all:

$(foreach file,$(FILES),$(call GENERATE_RULE,$(file)))

// VPATH  --> emit derived files into ./build directory  --> avoid using but output the generated files in a build directory
# This makefile should be invoked from the temporary build directory, eg:
# $ mkdir -p build && cd ./build && make -f ../Makefile

# Derive the directory containing this Makefile
MAKEFILE_DIR = $(shell dirname $(realpath $(firstword $(MAKEFILE_LIST))))

# now inform Make we should look for prerequisites from the root directory as
# well as the cwd
VPATH += $(MAKEFILE_DIR)

SRC_FILES = $(wildcard $(MAKEFILE_DIR)/src/*.c)

# Set the obj file paths to be relative to the cwd
OBJ_FILES = $(subst $(MAKEFILE_DIR)/,,$(SRC_FILES:.c=.o))

# now we can continue as if Make was running from the root directory, and not a
# subdirectory

# $(OBJ_FILES) will be built by the pattern rule below
foo.a: $(OBJ_FILES)
	$(AR) rcs $@ $(OBJ_FILES)

# pattern rule; since we added ROOT_DIR to VPATH, Make can find prerequisites
# like `src/test.c` when running from the build directory!
%.o: %.c
	# create the directory tree for the output file 👍
	echo $@
	mkdir -p $(dir $@)
	# compile
	$(CC) -c $^ -o $@

===================
// debugging makefiles
1. make equivalent of printf --> info/warning/error functions
2.--debug option and redirecting stdout to a file
3.profiling --> https://github.com/rocky/remake
4.verbose flag  -->
ifeq ($(V),1)
Q :=
else
Q := @
endif

%.o: %.c
	# prefix the compilation command with the $(Q) variable
	# use echo to print a simple "Compiling x.c" to show progress
	@echo Compiling $(notdir @^)
	$(Q) $(CC) -c $^ -o $@

--> enable the full print out, set the V environment variable:
$ V=1 make


2020/03/13
====================
gdb debug:
1.flash the target, setup the gdb server  --> -device -if -port
2.gdb  --> build directory
3.file xxx.out/elf  --> load the symbols
4.target remote localhost:3333  --> connect to gdb server
5.set print pretty on
6.set logging on/off  --> gdb.txt  is saved on current folder
7.b main.c:175/b main.c:175 if cr == 's'/c/bt full/s/si
8.list  --> src around PC
9.info  locals/info variables/info files 

// gdb python api: released 2009 in v7.0
gdb --ex="source /path/to/custom_gdb_extensions.py"
or put under ~/.gdbinit

2020/03/10
====================
vector table index 0  --> reset value of main stack pointer
index 1-15  --> cortex-m reserved exception number 1-15

six exceptions are always supported: reset/Non Maskable Interrupt(NMI)/HardFault/SVCall(Supervisor Call)/PendSV & SysTick(system level interrupt triggered by software)

external interrupts  --> configured via NVIC(nested vectored interrupt controller)  --> exception number start from 16

registers to configure cortex-m exceptions  --> system control space(SCS)

tail-chaining: pop and push stack is skipped when a new exception is pended during exiting an ISR  --> save 18 cycles on cortex-m3 with zero wait state memory(12 cycles to start executing an ISR and 12 cycles to return from ISR, tail-chaining just take 6 cycles to exit and start another one)

exeution priority & priority boosting:
If no exception is active, the current “execution priority” of the system can be thought of as being the “highest configurable priority level” + 1 – essentially meaning if any exception is pended, the currently running code will be interrupted and the ISR will run

Priority boosting is usually controlled via three register fields:

PRIMASK - Typically configured in code using the CMSIS __disable_irq() and __enable_irq() routines or the cpsid i and cpsie i assembly instructions directly. Setting the PRIMASK to 1 disables all exceptions of configurable priority. This means, only NMI, Hardfault, & Reset exceptions can still occur.

FAULTMASK - Typically configured in code using the CMSIS __disable_fault_irq() and __enable_fault_irq() routines or the cpsid f and cpsie f assembly instructions directly. Setting the FAULTMASK disables all exceptions except the NMI exception. This register is not available for ARMv6-M devices.

BASEPRI- Typically configured using the CMSIS __set_BASEPRI() routine. The register can be used to prevent exceptions up to a certain priority from being activated. It has no effect when set to 0 and can be set anywhere from the highest priority level, N, to 1. It’s also not available for ARMv6-M based MCUs.

interruptible-continuable instructions

2020/03/09
=====================
cortex-m operation mode: 
handler mode  --> exception like isr
thread mode  --> normal

cortex-m operation level:
privileged  --> in handler mode, the core is always privileged
unprivileged  --> in thread mode, software can execute at either level  --> switching from unprivileged to previleged can only happen in handler mode

core registers
=====================
Register	Alternative Names	Role in the procedure call standard
r15			PC			The Program Counter (Current Instruction)
r14			LR			The Link Register (Return Address)
r13			SP			The Stack Pointer
r12			IP			The Intra-Procedure-call scratch register
r11			v8			Variable-register 8
r10			v7			Variable-register 7
r9			v6, SB, TR		Variable-register 6 or Platform Register
r8,r7, r6, r5, r4	v5, v4, v3, v2, v1	Variable-register 5 - Variable-register 1
r3, r2, r1, r0		a4, a3, a2, a1		Argument / scratch register 4 - 1

r12 - intra-procedure-call scratch register  --> veneer to enable branch and link (bl) instruction to jump across the entire address region

r9 - platform register  --> as static base(SB) for PIC(position independent code)
			--> as thread register for current thread local storage context

floating point registers
========================
floating point registers  --> FPv4-SP / FPv5  --> s0-s31 as single word register
					      --> d0-d15 as double word register
			  --> FPSCR: status and control
			  --> CPACR: coprocessor access control register

special registers
=======================
special registers  --> operate by system instruction  	--> MSR: move to special register
		   		       			--> MRS: move to register from special register  						 
control register  --> important  for context switching  --> SFPA: secure floating point is active or not(ARMv8-M)
							--> FPCA: floating point context is active
							--> SPSEL: which stack pointer in use
							--> nPriv: thread mode is privileged or unprivileged 		

stack pointers
=======================
stack pointer   --> msp: main stack  --> after reset / in handler mode
		--> psp: process stack  --> in thread mode  --> write to 1 of SPSEL in control register: msp -> psp
							    --> the value in EXC_RETURN upon exception return

context state stacking
======================
per AAPCS, callee function responsible for preserve/restore the original state before return to the caller function:
registers r4-r8, r9(if designated as v6), r10, r11 and SP

caller function responsible for preserve/restore:
registers r0-r3, r12 and r14(LR) + xPSR register N,Z,C,V,Q bits(bits 27-31) and the GE bits(bits 16-19)

AAPCS also impose the stack must be double word aligned(8-byte aligned)  --> bit 9 of xPSR to indicate whether 4 byte padding was added to be 8-byte aligned upon exception entry

context is pushed on the stack in use prior to serve the exception  --> if core was serving another exception and preempted by a higher priority exception  --> pushed on the msp

if floating point extension is in use and active, AAPCS impose s0-s15 + FPSCR must be preserved/restored(total 17 registers / 68 bytes)  --> FPCCR(0xe000 ef34 - floating point context control register)  --> lazy context saving     	

exception return
=======================
exception return  --> what state to restore  --> EXC_RETURN value in link register(LR)  --> typically mirror the LR value on exception entry  --> can manually change by SPSEL on control register in thread mode

exception entry  --> link register(LR) value depends on current stack frame(extended or basic) and stack pointer(msp or psp) in use

EXC_RETURN Value	Mode to Return To			Stack to use
0xFFFFFFF1		Handler Mode				MSP
0xFFFFFFF9		Thread Mode				MSP
0xFFFFFFFD		Thread Mode				PSP
0xFFFFFFE1		Handler Mode (FPU Extended Frame)	MSP
0xFFFFFFE9		Thread Mode (FPU Extended Frame)	MSP
0xFFFFFFED		Thread Mode (FPU Extended Frame)	PSP

rtos context switching
======================
SVCall / PendSVCall / SysTick  --> for task management  --> similiar to any rtos

SysTick or Yield generate PendSV exception

start schedule by triggering SVCall exception with svc instruction  --> new task is inited as context switched out by scheduler

pxPortInitialiseStack / xPortStartScheduler / vPortSVCHandler in port.c


2019/03/08
===================
linker script has two addresses, its load address (LMA) and its virtual address (VMA). 

In a firmware context, the LMA is where your JTAG loader needs to place the section and the VMA is where the section is found during execution.

You can think of the LMA as the address “at rest” and the VMA the address during execution i.e. when the device is on and the program is running.

.data :
{
    *(.data*);
} > ram AT > rom  /* "> ram" is the VMA, "> rom" is the LMA */

2019/07/28
====================
// use ctags to index source code
cd to source directory
make tags
ctrl+] to look up definitions

2019/07/23
====================
su - dp --> the "-" instruct su to start a login shell
// when logged on as normal user dp, it's a login shell read /etc/profile then .bash_profile
// non-login shell read .bashrc instead
// the process to create a clean build environment -->
su - dp  --> source .bash_profile(login shell)  --> source .bashrc(non-login shell)

====================
// manual build cross toolchain process -->
unset CFLAGS 

2019/07/19
====================
gun make wild function  -->
// get a list of all the files in a directory(hashes), replace % from hashes/%.sha1 to $(SOURCES)/%  
$(patsubst hashes/%.sha1,$(SOURCES)/%,$(wildcard hashes/gcc*))

// functions for file names
$(dir names)  --> extract directory part of each file in names
$(notdir names)  --> extract name part of each file in names
$(basename names)  --> extract name part without suffix of each file in names

// config.sub - validate and canonicalize a configuration triplet
config.sub  --> generate by autoconf, convert system aliases into full canonical names, generally you do not need care

2019/07/17
====================
git grep / git blame

port an os to a new architecture:
1. bootstraping:  --> toolchain
	1.1 gcc  --> arm-linux-musl
	1.2 binutils  --> arm-linux-musl
	1.2 libc  --> musl
2.poring:  --> compile itself and the packages
	2.1 native toolchain
	2.2 package manager
	2.3 utility like tar, patch, openssl etc.

2019/07/14
====================
random number generator package - haveged
haveged -n 0 | dd of=/dev/sdb  --> wiped the disk with random values

by default, lbu only take care /etc and its subfolders, except for /etc/init.d

lbu include / exclude / list-backup / revert
// execute a script before/after a backup  --> /etc/lbu/pre-package.d/post-package.d

=====================
apk is digitally signed tar.gz archive containing binary, config and dependency metadata.

a repo is a directory with a collection of *.apk files include a special index file named APKINDEX.tar.gz.

use "@" to specify tagged repo in /etc/apk/repositories:
@edge http://nl.alpinelinux.org/alpine/edge/main
@edgecommunity http://nl.alpinelinux.org/alpine/edge/community
@testing http://nl.alpinelinux.org/alpine/edge/testing

and pin dependencies to these tags:
apk add stableapp newapp@edge bleedingapp@testing

or just add main repo, add community package as:
apk add openntp --update-cache --repository http://dl-3.alpinelinux.org/alpine/edge/testing/ --allow-untrusted

// add local package
apk add --allow-untrusted /path/to/file.apk

// just upgrade a package
apk add --upgrade busybox

// search package
apk search -v
apk search -v 'acf*'  --> list packages as part of ACF system
apk search -v --description 'NTP'  --> list packages with description NTP

// info on package
apk info -a zlib

// check belong to which package
apk info --who-owns /sbin/lbu  --> /sbin/lbu is owned by alpine-conf-x.x-rx

// list all installed packages in alphabet order
apk -vv info | sort

// check source
apk policy vlc  -->
vlc policy:
  2.2.6-r1:
    lib/apk/db/installed
    http://dl-3.alpinelinux.org/alpine/v3.7/community
  3.0.0_rc2-r1:
    @edgecommunity http://dl-3.alpinelinux.org/alpine/edge/community

// enable local cache
setup-apkcache

// delete old packages
apk cache clean
apk -v cache clean  --> check what is deleted 

// download missing packages
apk cache download

// delete old packages and download missing packages
apk cache -v sync

// automatically cleaning cache on reboot
/etc/local.d/cache.stop
#!/bin/sh

# verify the local cache on shutdown
apk cache -v sync

# We should always return 0
return 0

--> it's better to sync the cache after add or upgrade packages.

apk upgrade --update-cache --available  --> force upgrade all

2019/07/12
====================
modloop is loopback cramfs image where kernel modules are stored for tmpfs install

====================
alpine linux architecture:

installation mode: diskless, data, sys.

for x86 use syslinux as bootloader, kernel patched with grsecurity.

base on musl libc, busybox as core utilities, openrc as init system.

setup scripts is under /sbin/setup-*, as part of alpine-conf package.

initramfs is generated by /sbin/mkinitfs, as part of mkinitfs package. config is read from /etc/mkinitfs/* and install the initscript /usr/share/mkinitfs/initramfs-init into the initramfs.

/init --> /sbin/init --> openrc.

abuild is build utility, package-building script is named APKBUILD.

package-building tree is named aports tree.

for normal usage, just install alpine-base, which include mkinitfs and apk-tools.

for development usage, install alpine-sdk, which include gcc, git, abuild etc.

2019/06/21
====================
// raspberry pi boot files
1.first stage bootloader - from soc gpu rom code, mount fat32 partition
2.second stage bootloader - from sd card bootcode.bin - init gpu
3.gpu firmware -from sd card start.elf - init cpu
4.fixup.dat - configure dram partition between cpu and gpu
5.device tree - bcm2837-rpi-3-b-plus.dtb
6.kernel - kernel8.img
7.cmdline.txt
8.config.txt

optional with fixup_cd.dat, fixup_db.dat, fixup_x.dat, start_cd.elf, start_db.elf, start_x.elf
// raspberry pi can check at boot time to load correct soc, board and kernel

2019/06/19
====================
// change default login user
/etc/default/nodm

// boot from eMMC / NAND, system on SATA / USB
nand-sata-install

// clear the boot loader signature on sd card
dd if=/dev/zero of=/dev/mmcblkN bs=1024 seek=8 count=1

// how to connect to wireless - network manager text ui (nmtui)
nmtui-connect SSID or nmtui-connect to scan SSIDs
// show network interface managed by network manager
nmcli con show
nmcli con mod

2019/06/01
====================
$ du -h --max-depth=1 | sort -hr  --> check directory size

=====================
iw - linux wireless

iw is a new nl80211 based CLI configuration utility for wireless devices. It supports all new drivers that have been added to the kernel recently. The old tool iwconfing, which uses Wireless Extensions interface, is deprecated and it’s strongly recommended to switch to iw and nl80211.

// find wireless adapter
$ iw dev

// check device status
$ ip link show wlan0

// bring up wifi interface
$ ip link set wlan0 up

// check connection status
$ iw wlan0 link

// scan wifi network
$ iw wlan0 scan

// wpa_supplicant.conf
$ wpa_passphrase dp >> /etc/wpa_supplicant.conf

// connect to wifi network
$ wpa_supplicant -B -D nl80211 -i wlan0 -c /etc/wpa_supplicant.conf

// get ip through dhcp - dhclient wlan0 ?
$ udhcpc -i wlan0

// check ip - ifconfig wlan0
$ ip addr show wlan0

=============================
// list installed images and headers
dpkg --list | grep linux-image
dpkg --list | grep linux-headers

// purge unused images/headers
sudo apt purge linux-image-4.18.0-18
