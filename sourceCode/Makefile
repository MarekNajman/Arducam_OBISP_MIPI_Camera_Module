# SPDX-License-Identifier: GPL-2.0
# Out-of-tree build entry point for the Arducam OBISP MIPI camera module.

obj-m := arducam.o

KDIR ?= /lib/modules/$(shell uname -r)/build
PWD := $(shell pwd)

default modules:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

modules_install install:
	$(MAKE) -C $(KDIR) M=$(PWD) modules_install

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
