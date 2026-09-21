# riscv-linux-from-scratch

A minimal RV32 linux system built from source for a custom RISC-V SoC: OpenSBI as the M mode firmware, a 32-bit linux kernel and a BusyBox based initramfs linked into the kernel image. Everything is compiled as bare images (no U-Boot, no disk, no root filesystem on external media (e.g. SD Card)) and loaded straight into memory, so it also works on a small FPGA or simulated SoC that has nothing but a UART, a timer and some DRAM.

It is used with [secure-soc](https://github.com/celuk/secure-soc), which is a CVA6 based SoC with secure boot and on-the-fly memory encryption-decryption. The old and messier version of this work is in [riscv-linux-boot](https://github.com/celuk/riscv-linux-boot).

## Requirements

A 32-bit riscv linux toolchain, built from [riscv-gnu-toolchain](https://github.com/riscv-collab/riscv-gnu-toolchain) with `--with-arch=rv32imac --with-abi=ilp32`:

```bash
export CROSS_COMPILE=<your-toolchain-path>/bin/riscv32-unknown-linux-gnu-
```

A 64-bit distro toolchain also works for OpenSBI if you pass `PLATFORM_RISCV_XLEN=32`, but the kernel and BusyBox want the rv32 one.

Also needed: `dtc` for the device tree, `python3` with `pyserial` for programming over UART, and the scripts in [secure-soc/tools](https://github.com/celuk/secure-soc/tree/main/tools) ([`bin2hex.py`](https://github.com/celuk/secure-soc/tree/main/tools/bin2hex.py), [`uart_send_data_to_dram.py`](https://github.com/celuk/secure-soc/tree/main/tools/uart_send_data_to_dram.py), [`vmem_to_ddr3_init.py`](https://github.com/celuk/secure-soc/tree/main/tools/vmem_to_ddr3_init.py)).

Every submodule has its own `compile.sh` with the exact commands. The toolchain and tool paths in those scripts are absolute local paths, so fix them for your machine before running them.

## Submodules

| Submodule | Version | Description |
| --- | --- | --- |
| [riscv-opensbi-port](https://github.com/celuk/riscv-opensbi-port) | OpenSBI v1.7 | [`platform/template`](riscv-opensbi-port/platform/template) port for the SoC, custom UART driver for the SBI console, [`custom.dts`](riscv-opensbi-port/platform/template/custom.dts) device tree |
| [riscv-linux-ue](https://github.com/celuk/riscv-linux-ue) | Linux v4.20 | RV32 config [`arch/riscv/configs/32-bit.config`](riscv-linux-ue/arch/riscv/configs/32-bit.config) for the SoC |
| [riscv-busybox-port](https://github.com/celuk/riscv-busybox-port) | BusyBox 1.36.1 | static build config and the init [`script`](riscv-busybox-port/compile.sh) that builds the initramfs that provides basic linux tools e.g. bash console |

## Compilation

```bash
git clone https://github.com/celuk/riscv-linux-from-scratch
```

```bash
cd riscv-linux-from-scratch
```

```bash
git submodule update --init --remote --recursive
```

```bash
cd riscv-opensbi-port

./compile.sh

cd ..

cd riscv-busybox-port

./compile.sh

cd ..

cd ../riscv-linux-ue

./compile.sh

cd ..
```

## Running

Generated hex files to program:

```bash
riscv-opensbi-port/platform/template/custom.dtb.hex
```

```bash
riscv-linux-ue/arch/riscv/boot/Image.hex
```

```bash
riscv-opensbi-port/build/platform/template/firmware/fw_dynamic.hex
```

--> You can program these hex codes seperately to their addresses that is defined in SoC bootrom as it is done in [`Makefile`](https://github.com/celuk/secure-soc/tree/main/Makefile) `program_linux` make command to run on a soft-core SoC running on an FPGA.

## Boot Process

A zero stage bootloader in the SoC bootrom runs in M mode, hands control to OpenSBI in DRAM, OpenSBI does the M mode setup and drops to S mode at the linux entry point. The kernel takes the device tree pointer it was given, unpacks the initramfs that is linked into its own image and runs BusyBox `/init` in U mode. No second stage bootloader and no block device are involved.

The default DRAM base of the SoC is `0x80000000` and the three images are placed like this:

| Image | Offset from DRAM base | Absolute address |
| --- | --- | --- |
| OpenSBI `fw_dynamic.bin` | `0x00000000` | `0x80000000` |
| Linux `Image` (kernel + initramfs) | `0x00400000` | `0x80400000` |
| Device tree `custom.dtb` | `0x01400000` | `0x81400000` |

These offsets are the ones in [`bootloader_dram.c`](https://github.com/celuk/secure-soc/blob/main/tests/bootloader_dram/bootloader_dram.c) and in [`vmem_to_ddr3_init.py`](https://github.com/celuk/secure-soc/blob/main/tools/vmem_to_ddr3_init.py). If you change one, change the other two as well.

## FW_DYNAMIC bootloader

OpenSBI can be built three ways. `FW_JUMP` has the next stage address compiled in, `FW_PAYLOAD` embeds the kernel inside the firmware binary itself, and `FW_DYNAMIC` is told at runtime where to go next. This repo uses `FW_DYNAMIC`, so the kernel and the device tree stay separate files that can be loaded, replaced or encrypted on their own without rebuilding OpenSBI.

The price is that `FW_DYNAMIC` will not boot on its own. The previous stage has to fill a `fw_dynamic_info` struct and enter OpenSBI with `a0 = hartid`, `a1 = dtb address` and `a2 = pointer to that struct`. A bootloader that does not do this will hand OpenSBI garbage in `a2` and the boot dies before any console output.

The bootrom bootloader of secure-soc supports this. [`bootloader_dram.c`](https://github.com/celuk/secure-soc/blob/main/tests/bootloader_dram/bootloader_dram.c) is the example to copy:

```c
struct fw_dynamic_info {
    unsigned int magic;
    unsigned int version;
    unsigned int next_addr;
    unsigned int next_mode;
    unsigned int options;
    unsigned int boot_hart;
};
```

```c
dynamic_info.magic     = 0x4942534f; // magic word "OSBI"
dynamic_info.version   = 0x2;
dynamic_info.next_addr = 0x80000000 + 0x00400000; // linux Image pointer
dynamic_info.next_mode = 0x1; // S mode
dynamic_info.options   = 0x0;
dynamic_info.boot_hart = 0x0;
```

```c
__asm__ volatile (
    "mv a0, %[hart_id]\n"
    "mv a1, %[dtb_addr]\n" // 0x80000000 + 0x01400000
    "mv a2, %[info_addr]\n" // &dynamic_info
    "li t0, %[entry]\n" // 0x80000000, OpenSBI
    "jalr x0, t0, 0\n"
);
```

Note that `next_mode` is `1` (S mode), not `3`. If you set it to M mode the kernel starts with the wrong privilege level and traps as soon as it touches an S mode CSR.

If your SoC instead has a bootloader that only jumps to a fixed address, build with `FW_JUMP=y FW_JUMP_ADDR=0x80400000 FW_JUMP_FDT_ADDR=0x81400000`, or use `FW_PAYLOAD=y FW_PAYLOAD_OFFSET=0x00400000 FW_PAYLOAD_PATH=../riscv-linux-ue/arch/riscv/boot/Image` to get a single blob.
