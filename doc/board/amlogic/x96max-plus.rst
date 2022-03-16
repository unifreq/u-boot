.. SPDX-License-Identifier: GPL-2.0+

U-Boot for X96Max Plus
=========================

X96Max plus (gbit ethernet) is a android tvbox manufactured by Shenzhen 
Amediatech Technology Co., Ltd
specifications:

 - Amlogic S905X3 ARM Cortex-A55 quad-core SoC
 - 4GB DDR4 SDRAM
 - 1Gbps Ethernet (External PHY)
 - 1 x USB 3.0 OTG
 - 1 x USB 2.0 HOST
 - eMMC
 - SDcard
 - Infrared receiver
 - SDIO WiFi Module

U-Boot compilation
------------------

.. code-block:: bash

    $ export CROSS_COMPILE=aarch64-none-elf-
    $ make x96max-plus_defconfig
    $ make

Image creation
--------------

Amlogic does not provide a source of firmware and the required tools 
to create a bootloader image, so it is necessary to extract acs.bin 
from the bootloader binary of the Android firmware released by the 
vendor, and copy the other relevant files from existing similar 
hardware (e.g. from khadas-vim3l).

This work has been done by flippy <flippy@sina.com>, see:
https://github.com/unifreq/amlogic-boot-fip/tree/master/x96max-plus

The general principle is: open the bootloader provided by the manufacturer 
with a hexadecimal editor, intercept the 4096 bytes content starting at 
offset 0xf200, and save it as acs.bin.

(Special attention is: If the manufacturer's bootloader is encrypted, then 
this method is invalid!)


.. code-block:: bash

    $ git clone https://github.com/unifreq/amlogic-boot-fip
    $ export FIPDIR=${PWD}/amlogic-boot-fip/x96max-plus

Go back to mainline U-Boot source tree then :


.. code-block:: bash

    $ mkdir fip
    $ cp $FIPDIR/* fip/
    $ cp u-boot.bin fip/bl33.bin

    $ bash fip/blx_fix.sh \
        fip/bl30.bin \
        fip/zero_tmp \
        fip/bl30_zero.bin \
        fip/bl301.bin \
        fip/bl301_zero.bin \
        fip/bl30_new.bin \
        bl30

    $ bash fip/blx_fix.sh \
        fip/bl2.bin \
        fip/zero_tmp \
        fip/bl2_zero.bin \
        fip/acs.bin \
        fip/bl21_zero.bin \
        fip/bl2_new.bin \
        bl2

    $ $FIPDIR/fip/aml_encrypt_g12a --bl30sig --input fip/bl30_new.bin \
        --output fip/bl30_new.bin.g12a.enc \
        --level v3

    $ $FIPDIR/aml_encrypt_g12a --bl3sig --input fip/bl30_new.bin.g12a.enc \
        --output fip/bl30_new.bin.enc \
        --level v3 --type bl30

    $ $FIPDIR/aml_encrypt_g12a --bl3sig --input fip/bl31.img \
        --output fip/bl31.img.enc \
        --level v3 --type bl31

    $ $FIPDIR/aml_encrypt_g12a --bl3sig --input fip/bl33.bin --compress lz4 \
        --output fip/bl33.bin.enc \
        --level v3 --type bl33

    $ $FIPDIR/aml_encrypt_g12a --bl2sig --input fip/bl2_new.bin \
        --output fip/bl2.n.bin.sig

    $ $FIPDIR/aml_encrypt_g12a --bootmk \
        --output fip/u-boot.bin \
        --bl2  fip/bl2.n.bin.sig \
        --bl30 fip/bl30_new.bin.enc \
        --bl31 fip/bl31.img.enc \
        --bl33 fip/bl33.bin.enc \
        --ddrfw1 fip/ddr4_1d.fw \
        --ddrfw2 fip/ddr4_2d.fw \
        --ddrfw3 fip/ddr3_1d.fw \
        --ddrfw4 fip/piei.fw \
        --ddrfw5 fip/lpddr4_1d.fw \
        --ddrfw6 fip/lpddr4_2d.fw \
        --ddrfw7 fip/diag_lpddr4.fw \
        --ddrfw8 fip/aml_ddr.fw \
        --ddrfw9 fip/lpddr3_1d.fw \
        --level v3

and then write the image to SD with:

.. code-block:: bash

    $ DEV=/dev/your_sd_device
    $ dd if=fip/u-boot.bin.sd.bin of=$DEV conv=fsync,notrunc bs=512 skip=1 seek=1
    $ dd if=fip/u-boot.bin.sd.bin of=$DEV conv=fsync,notrunc bs=1 count=444
