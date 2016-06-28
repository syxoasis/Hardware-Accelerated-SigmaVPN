# Hardware-Accelerated-SigmaVPN

This is a VPN Device design project using SigmaVPN application as the base VPN solution. The SigmaVPN is altered to construct a VPN Device which utilize a cryptogprahic hardware acceleration to execute expensive cryptographic operations in a short time, and maximize the communication bandwidth. Moreover, the device, works with two Ethernet ports: One is public and the other is private communication ports.

A Virtual Private Network (VPN) encrypts and decrypts the private traffic it tunnels over a public network. Maximizing the available bandwidth is an important requirement for network applications, but the cryptographic operations add significant computational load to VPN applications, limiting the network throughput. This work presents a coprocessor designed to offer hardware acceleration for these encryption and decryption operations. The open-source SigmaVPN application is used as the base solution, and a coprocessor is designed for the parts of Networking and Cryptography library (NaCl) which underlies the cryptographic operation of SigmaVPN. The hardware-software codesign of this work is implemented on a Xilinx Zynq-7000 SoC, showing a 93% reduction in the execution time of encrypting a 1024-byte frame, and this improved the TCP and UDP communication bandwidths by a factor of 4.36 and 5.36 respectively for a 1024-byte frame compared to pure software solution.

This work is completed as a Master Thesis project in KU Leuven - ESAT / COSIC.

## VPN device

The device is implemented on Zynq SoC. The Zynq consists of both Processing System (PS) and Programmable Logic (PL). PS refers to dual ARM Cortex-A0 cores, on which Linux and SigmaVPN run, and PL refers to FPGA on which the hardware acceleration for the cryptographic operations are implemented.

The device is configured to operate with two ports which are named as public and private ports. Private port listens every packet arriving the device i.e. all TCP, UDP, DHCP, ARP, ... All specific or broadcast messages received by the private port are encrypted and transmitted to a destination device over public port. The prive port is obtained by creating a new network interface module that promiscuously reads Ethernet frames using libpcap packet sniffing library.

In that scheme, all messages arriving to private sides of both VPN devices are forwarded to each other, and PC's or subnetworks connected to private sides of both VPN pairs communicate with each other as if they are physically connected. And the packets traveling in between two private pairs are transferred over Internet under encryption.

The hardware acceleration is provided for NaCl's CryptoBox which underlies the cryptographic operations of the SigmaVPN. With a new module introduced to the SigmaVPN, it is possible to utilize the NaCl CryptoBox coprocessor hardware implemented in the hardware through provided device driver.

## How to Use It

To use the design on a ZYBO board, you should prepare the BootFiles (or use the one provided, and jump to step 11).

A [tutorial](http://www.instructables.com/id/Embedded-Linux-Tutorial-Zybo/?ALLSTEPS) showing the steps are given provided by Digilent and can offer a great help.

Step 1: Implement the design on Vivado, and create bitstream.

Step 2: Make UImage (Use [U-boot](https://github.com/DigilentInc/u-boot-Digilent-Dev) by Digilent)

Step 3: Generate U-boot.elf

Step 4: Build the first stage boot loader from the bitstream, a more explanatory tutorial for that is [here](http://www.dbrss.org/zybo/tutorial4.html) (Replace the default `fsbl_hooks.c` file with the provided one.)

Step 5: Provide a USB-to-Ethernet Adapter for the second Ethernet port of the VPN Device. We used [Edimax EU-4306 USB 3.0 Gigabit Ethernet Adapter](http://www.edimax.com/edimax/merchandise/merchandise_detail/data/edimax/au/network_adapters_usb_adapters/eu-4306/) 

Step 6: Cross-Compile Linux Kernel (with the drivers enabled for adapter)

Step 7: Make Uramdisk

Step 8: Generate DTB file (study the provided one for accessing the DMA using its physical address mapped to the PS.
 
Step 9: Cross-Compile SigmaVPN.

Step 10: Copy the output files to MicroSD card

Step 11: Boot the ZYBO board from SD Card.

Step 12: Install the SigmaVPN on it.

Step 13: Initialize the SigmaVPN application. 

Sample initialization scripts (installing, and running the SigmaVPN) together with configuration files are provided in BootFiles directory. For example, you should initialize it with demo1 configuration on one node, and  with demo2 configuration on the other node:

```
cd mnt/sigmavpn
./demo1.sh
```

## Explanation of Directories

### BootFiles

This directory contains required files that you need to copy into a MicroSD card and run the ZYBO board with.

### HW

This folder includes the base HW design of FPGA as Vivado project. The output of the design is provided as .bit file in the BootFiles folder as well.

The folder has three designs, first two are a design for NaCl CryptoBox as custom IP's. One design impelements single instantiation of processing blocks, an the other double instantiation. The latter one is designed at a later stage, to utilize available resource of the Zynq SoC more in order to achieve better performance. 

The last design in the folder is the whole hardware implementation where NaCl CryptoBox IP and PS are instantiated, and DMA IP is provided as a communucation mechanism between them.

The hardware is designed on Vivado 2014.3.1 using VHDL.

### Linux Kernel

The device runs [this Linux kernel](https://github.com/Digilent/linux-digilent-dev ) provided by Digilent for ZYBO board.                 

It is not added as sub-repository to the project, but two modifications are included. 

The first modification is done to `.config` file to enable AX88179_178A drivers needed to enable drivers of used USB-Ethernet device. Therefore, provided `.config` file should be used instead of creating default configuration file using 'xilinx_xynq_defconfig'.

Another modification is to provide DMA driver in 'drivers/dma/xilinx/' directory. This driver is a key element to utilize the hardware acceleration withing the new cryptography module provided to the SigmaVPN. The is the driver file used to communicate with DMA and so handle communication with coprocessor to provide HW accelerators of cryptographic functions.

In the driver folder, two different drivers are provided. The first one does a context switching between user space and kernel space, and the second one mmap's kernel space memory into user space to avoid context switching. The comparison of them are discussed in the thesis text in detail. 

You can use this tutorial to learn how to compile Linux and play with ZYBO board [here](http://www.instructables.com/id/Embedded-Linux-Tutorial-Zybo/?ALLSTEPS).

### SigmaVPN

This folder has a fork of original SigmaVPN code, as a sub-repository. In this fork, the change of SigmaVPN is made to provide it two-port operation capability at first, and to make it utilize the coprocessor. For each feature, two new modules to the SigmaVPN are offered. The first one for physical communication feature instead of using virtual network interface, and the other is to utilize designed cryptographic coprocessor. 

Running the SigmaVPN requires libsodium, and libpcap libraries. The former is for NaCl and the latter is for two-port operation. 

The readme file in the directory describes how to cross-compule these libraries and SigmaVPN for the ZYBO board. 

### TestApp / TestApp_UserSpace

This folder includes an application that creates random test vector and then encrypts/decrypts them using first software only implementation of NaCl CryptoBox, and then using the designed coprocessor. The aim is to compare results, and compare execution time of both implementations.  the results are provided in the thesis text in detail. Shortly, encrypting a 1024-byte vector in hardware requires 94% less time compared to using the software implementation.

Two folders are there for two different device drivers that are provided in the LinuxKernel folder.

### VerificationApp

This folder includes a C application, and some Magma codes that are used to create test vectors, and encrypt/decrypting them. The goal is providing the intermediate results through out executing the cryptographic operations. These resutls are used while desining the hardware, and comparing the results obtained in the hardware simulation.
