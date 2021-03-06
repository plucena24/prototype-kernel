#
# Makefile for out-of-tree building eBPF programs
#  similar to kernel/samples/bpf/
#
# Still depend on a kernel source tree.
#
TARGETS := xdp_ddos01_blacklist
TARGETS += xdp_ttl

TARGETS += xdp_bench01_mem_access_cost
TARGETS += xdp_bench02_drop_pattern

CMDLINE_TOOLS := xdp_ddos01_blacklist_cmdline
COMMON_H      =  ${CMDLINE_TOOLS:_cmdline=_common.h}

# Files under kernel/samples/bpf/ have a name-scheme:
# ---------------------------------------------------
# The eBPF program is called xxx_kern.c. This is the restricted-C
# code, that need to be compiled with LLVM/clang, to generate an ELF
# binary containing the eBPF instructions.
#
# The userspace program called xxx_user.c, is a regular C-code
# program.  It need two external components from kernel tree, from
# samples/bpf/ and tools/lib/bpf/.
#
# 1) When loading the ELF eBPF binary is uses the API load_bpf_file()
#    via "bpf_load.h" (compiles against a modified local copy of
#    kernels samples/bpf/bpf_load.c).
#
# 2) The API for interacting with eBPF comes from tools/lib/bpf/bpf.h.
#    For now, tools/lib/bpf/bpf.c is compiled directly, and executable
#    is statically linked with object file.
#
#    This is likely improper use of tools/lib/bpf/, that can generate
#    shared library code.  Hopefully someone will cleanup this
#    Makefile and correct this usage.


# Generate file name-scheme based on TARGETS
KERN_SOURCES = ${TARGETS:=_kern.c}
USER_SOURCES = ${TARGETS:=_user.c}
KERN_OBJECTS = ${KERN_SOURCES:.c=.o}
USER_OBJECTS = ${USER_SOURCES:.c=.o}

# Notice: the kbuilddir can be redefined on make cmdline
kbuilddir ?= /lib/modules/$(shell uname -r)/build/
KERNEL=$(kbuilddir)

CFLAGS := -O2 -Wall

# Local copy of kernel/tools/lib/
CFLAGS += -I./tools/lib
#CFLAGS += -I$(KERNEL)/tools/lib
#
# Local copy of uapi/linux/bpf.h kept under ./tools/include
# needed due to enum dependency in bpf_helpers.h
CFLAGS += -I./tools/include
CFLAGS += -I$(KERNEL)/tools/include
#CFLAGS += -I$(KERNEL)/tools/perf
CFLAGS += -I$(KERNEL)/usr/include
# Strange dependency to "selftests" due to "bpf_util.h"
#CFLAGS += -I$(KERNEL)/tools/testing/selftests/bpf/

LDFLAGS= -lelf

# Objects that xxx_user program is linked with:
OBJECT_LIBBPF  = tools/lib/bpf/bpf.o
OBJECT_LOADBPF = bpf_load.o
OBJECTS = $(OBJECT_LIBBPF) $(OBJECT_LOADBPF)

# Allows pointing LLC/CLANG to another LLVM backend, redefine on cmdline:
#  make LLC=~/git/llvm/build/bin/llc CLANG=~/git/llvm/build/bin/clang
LLC ?= llc
CLANG ?= clang

CC = gcc

NOSTDINC_FLAGS := -nostdinc -isystem $(shell $(CC) -print-file-name=include)

# Copy of uapi/linux/bpf.h stored here:
LINUXINCLUDE := -I./tools/include/

LINUXINCLUDE += -I$(KERNEL)/arch/x86/include
LINUXINCLUDE += -I$(KERNEL)/arch/x86/include/generated/uapi
LINUXINCLUDE += -I$(KERNEL)/arch/x86/include/generated
LINUXINCLUDE += -I$(KERNEL)/include
LINUXINCLUDE += -I$(KERNEL)/arch/x86/include/uapi
LINUXINCLUDE += -I$(KERNEL)/include/uapi
LINUXINCLUDE += -I$(KERNEL)/include/generated/uapi
LINUXINCLUDE += -include $(KERNEL)/include/linux/kconfig.h
LINUXINCLUDE += -I$(KERNEL)/tools/lib
EXTRA_CFLAGS=-Werror

all: dependencies $(TARGETS) $(KERN_OBJECTS) $(CMDLINE_TOOLS)

.PHONY: dependencies clean verify_cmds verify_llvm_target_bpf $(CLANG) $(LLC)

clean:
	@find . -type f \
		\( -name '*~' \
		-o -name '*.ll' \
		-o -name '*.bc' \
		-o -name 'core' \) \
		-exec rm -vf '{}' \;
	rm -f $(OBJECTS)
	rm -f $(TARGETS)
	rm -f $(KERN_OBJECTS)
	rm -f $(USER_OBJECTS)

dependencies: verify_llvm_target_bpf linux-src-devel-headers

linux-src:
	@if ! test -d $(KERNEL)/; then \
		echo "ERROR: Need kernel source code to compile against" ;\
		exit 1; \
	else true; fi

linux-src-libbpf: linux-src
	@if ! test -d $(KERNEL)/tools/lib/bpf/; then \
		echo "ERROR: Need kernel source code to compile against" ;\
		echo "       and specifically tools/lib/bpf/ "; \
		exit 1; \
	else true; fi

linux-src-devel-headers: linux-src-libbpf
	@if ! test -d $(KERNEL)/usr/include/ ; then \
		echo -n "WARNING: Need kernel source devel headers"; \
		echo    " likely need to run:"; \
		echo "       (in kernel source dir: $(KERNEL))"; \
		echo -e "\n  make headers_install\n"; \
		true ; \
	else true; fi

verify_cmds: $(CLANG) $(LLC)
	@for TOOL in $^ ; do \
		if ! (which -- "$${TOOL}" > /dev/null 2>&1); then \
			echo "*** ERROR: Cannot find LLVM tool $${TOOL}" ;\
			exit 1; \
		else true; fi; \
	done

verify_llvm_target_bpf: verify_cmds
	@if ! (${LLC} -march=bpf -mattr=help > /dev/null 2>&1); then \
		echo "*** ERROR: LLVM (${LLC}) does not support 'bpf' target" ;\
		echo "   NOTICE: LLVM version >= 3.7.1 required" ;\
		exit 2; \
	else true; fi

# The fake-minimal-libbpf
$(OBJECT_LIBBPF): %.o: %.c
	$(CC) $(CFLAGS) -o $@ -c $<

$(OBJECT_LOADBPF): bpf_load.c
	$(CC) $(CFLAGS) -o $@ -c $<

# Compiling of eBPF restricted-C code with LLVM
#  clang option -S generated output file with suffix .ll
#   which is the non-binary LLVM assembly language format
#   (normally LLVM bitcode format .bc is generated)
#
$(KERN_OBJECTS): %.o: %.c
	$(CLANG) -S $(NOSTDINC_FLAGS) $(LINUXINCLUDE) $(EXTRA_CFLAGS) \
	    -D__KERNEL__ -D__ASM_SYSREG_H -Wno-unused-value -Wno-pointer-sign \
	    -Wno-compare-distinct-pointer-types \
	    -Wno-gnu-variable-sized-type-not-at-end \
	    -Wno-tautological-compare \
	    -O2 -emit-llvm -c $<
	$(LLC) -march=bpf -filetype=obj -o $@ ${@:.o=.ll}

$(TARGETS): %: %_user.c $(OBJECTS) Makefile
	$(CC) $(CFLAGS) $(OBJECTS) $(LDFLAGS) -o $@ $<

$(CMDLINE_TOOLS): %: %.c $(OBJECTS) Makefile $(COMMON_H)
	$(CC) -g $(CFLAGS) $(OBJECTS) $(LDFLAGS) -o $@ $<
