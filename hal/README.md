# STM32WL Hardware Abstraction Layer

## Usage

This crate is not yet published to crates.io because it is incomplete, and
relies upon other unpublished crates (which is not allowed on crates.io).

There are near daily breaking changes because the crate is still under heavy
development, please pin the version you use.

```toml
[dependencies.stm32wl-hal]
git = "https://github.com/newAM/stm32wl-hal.git"
rev = "" # put a specific git commit hash here
features = [
    # use exactly one of the following dependeing on your target hardware
    "stm32wl5x_cm0p",
    "stm32wl5x_cm4",
    "stm32wle5",
    # optional: use the cortex-m-rt interrupt interface
    "rt",
    # optional: use defmt
    "defmt",
]

# include cortex-m-rt directly in your crate if you need interrupts
# use the interrupt macro from the hal with `use hal::pac::interrupt;`
# DO NOT use the interrupt macro from cortex-m-rt, it will fail to compile
[dependencies]
cortex-m-rt = "0.6"
```

**Note:** To avoid version mismatches do not include `cortex-m`, `embedded-hal`,
or `stm32wl` (the PAC) directly, these are re-exported by the hal.

```rust
use stm32wl_hal as hal;

use hal::cortex_m;
use hal::cortex_m_rt; // requires "rt" feature
use hal::embedded_hal;
use hal::pac; // published as "stm32wl" on crates.io
```

## Design

### Peripheral Access
The layout of devive memory for the STM32WL is provided from the vendor in a
formatcalled system view description (SVD).
The SVD is not perfect, so there is a set of community maintained SVD
patches at [stm32-rs].
After the SVD is patched it gets run through [svd2rust] which generates
the peripheral access crate (PAC) containing read/write/modify operations for
the device registers.

### Abstraction

_The fundamental theorem of software engineering_

> We can solve any problem by introducing an extra level of abstraction except
> for the problem of too many layers of abstraction.

You will be using the registers directly in some cases.
Not everything is abstracted, and even when this crate is complete some
functionality (e.g. CRC) may not be abstracted because the register interface is
simple enough. That being said if you find somthing missing it is likely
because this crate is incomplete, and not an intentional design decision.

```rust
use stm32wl_hal as hal;
use hal::pac;

use hal::{aes::Aes, pka::Pka};
use pac::{AES, PKA};

// The first thing you do is grab the peripherals from the PAC, this is a
// singleton, the `unwrap` call will panic only if you call this twice.
// This strategy prevents you from generating multiple peripheral drivers
// without the use on `unsafe`
let mut dp: pac::Peripherals = pac::Peripherals::take().unwrap();

// The drivers provided typically work by consuming the peripheral access
// structure generated by the PAC to provide structures with higher level
// functionality
let aes: Aes = Aes::new(dp.AES, &mut dp.RCC);
let pka: Pka = Pka::new(dp.PKA, &mut dp.RCC);
```

Generally speaking the driver structures have the following methods, though
this is not consistent (see [#78])

* `new` create a driver from a PAC struct
* `free` dystroy the driver and reclaim the PAC struct
* `steal` steal the driver
* `mask_irq` mask the peripheral
* `unmask_irq` unmask the peripheral IRQ
* `pulse_reset` reset the peripheral
* `disable_clock` disable the peripheral clock (for power saving)
* `enable_clock` enable the peripheral clock

[stm32-rs]: https://github.com/stm32-rs/stm32-rs
[svd2rust]: https://github.com/rust-embedded/svd2rust
[#78]: https://github.com/newAM/stm32wl-hal/issues/78