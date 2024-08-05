# `cortex-m-quickstart fork with instructions for targetting STM32F401RE from a Linux Host`

# STM32F401RE Rust Setup

This guide provides a step-by-step process for setting up Rust to run on the STM32F401RE microcontroller, assuming a host system running Ubuntu 24.04 LTS.

## Prerequisites

1. **Embedded Rust Book:** Complete all steps up to Chapter 2.2. You can skip 2.1 QEMU if not needed. You can access the book [here](https://rust-embedded.github.io/book).

2. **Install Rust Toolchain:**
   ```bash
   curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
   rustup target add thumbv7em-none-eabihf
   ```

3. **Install `cargo-generate`:**
   ```bash
   cargo install cargo-generate
   ```

4. **Install `st-util` for flashing:**
   ```bash
   sudo apt install stlink-tools
   ```

## Setup

### 1. Configure `.cargo/config.toml`

Set the compilation target specific to your board:

```toml
[build]
target = "thumbv7em-none-eabihf" # Cortex-M4F and Cortex-M7F (with FPU)
```

### 2. Modify `memory.x`

Add `ENTRY(Reset_Handler)` to your `memory.x` file:

### 3. Add STM32F4 Crate

Add the necessary crate for the STM32F4 series:

```bash
cargo add stm32f4@=0.15.1
```

### 4. Update `Cargo.toml`

Configure your dependencies in `Cargo.toml`:

```toml
[package]
edition = "2018"
name = "rust_nucleo"
version = "0.1.0"

[dependencies]
cortex-m = "0.6.0"
cortex-m-rt = { version = "0.6.10", features = ["device"] }
cortex-m-semihosting = "0.3.3"
panic-halt = "0.2.0"
stm32f4 = { version = "0.14.0", features = ["stm32f401", "rt"] }

[[bin]]
name = "rust_nucleo"
test = false
bench = false

[profile.release]
codegen-units = 1 # better optimizations
debug = true # symbols are nice and they don't increase the size on Flash
lto = true # better optimizations
```

### 5. Edit `src/main.rs`

Include this in your `main.rs` file to control the GPIO:

```rust
#![no_std]
#![no_main]

use panic_halt as _;
use cortex_m_semihosting::hprintln;
use stm32f4::stm32f401;
use cortex_m::asm;
use cortex_m_rt::entry;

#[entry]
fn main() -> ! {
    let peripherals = stm32f401::Peripherals::take().unwrap();
    let gpioa = &peripherals.GPIOA;
    gpioa.odr.modify(|_, w| w.odr0().set_bit());

    loop {
        hprintln!("Hello, world!").unwrap();
        asm::nop();
    }
}
```

## Build and Flash

### 1. Build

Compile your project in debug mode for better debugging information:

```bash
cargo build
```

### 2. Generate `.bin` File

Convert the ELF file to a binary:

```bash
cd ./target/thumbv7em-none-eabihf/debug
sudo apt install binutils-arm-none-eabi
arm-none-eabi-objcopy -O binary rust_nucleo rust_nucleo.bin
```

### 3. Flash

Erase and flash the microcontroller:

```bash
st-flash erase
st-flash write rust_nucleo.bin 0x08000000
st-flash reset
```

### 4. Debug

Start the OpenOCD server in a second terminal:

```bash
openocd -f interface/stlink.cfg -f target/stm32f4x.cfg
```

From the first terminal, start GDB:

```bash
gdb-multiarch -tui -q ./target/thumbv7em-none-eabihf/debug/rust_nucleo
```

Enter these commands in GDB:

```
target extended-remote :3333
monitor arm semihosting enable
break main
step
next
continue
```

## Optional

**Check Binary Size:**

```bash
file rust_nucleo
arm-none-eabi-readelf -S rust_nucleo
```

## Resources

- [Rust Embedded Book](https://rust-embedded.github.io/book)
- [STM32F4 Crate](https://crates.io/crates/stm32f4)
- [OpenOCD Documentation](http://openocd.org/documentation/)
- [SVD2Rust Documentation](https://docs.rs/svd2rust/0.24.1/svd2rust/#peripheral-api)

