# CCB Peripheral Driver

This module provides an abstraction layer for the **coupling and chaining bridge** (`CCB`) of STM32 MCUs: the hardware that **chains the
public key accelerator (PKA) with the secure AES coprocessor (SAES) and the random number generator (RNG)** so that a private key can be
generated, wrapped, stored and used for an ECDSA signature, an ECC scalar multiplication (ECDH) or a modular exponentiation **without
the key ever being visible to the software in clear text**. It offers the two sequences of the hardware - the one-time **blob creation**
and the repeated **blob usage** - and the status of the cryptographic subsystem. It is a low-level MCAL module: it owns the CCB hardware, the
application and the other modules never touch the registers.

The public interface (`Ccb_Port.h`, `Ccb_Types.h`) is **identical for all supported families**. The implementation
for a family lives in its own branch (`Dev/<Family>` for development, `Releases/<Family>` for the released versions), so
a module or an application written against the interface does not change when the MCU family changes.

This README is the landing page and the **specification of the interface** (master). The family branches contain their
own README with everything that is specific to the family: the instances of the device, their settings and the status of
the implementation.

> **Status: design.** No implementation exists yet, the interface below is a proposal. Stories and branches follow.

---

## Intent and purpose

- A private key is the most valuable thing in the device. With the plain Pka module the software loads the key into the PKA RAM, so the key passes
  through the memory of the CPU. The CCB closes this hole: the key is **generated inside** (from the RNG, written straight to the PKA RAM), **encrypted** with a
  key that comes from the hardware secret of this very device (the DHUK in the SAES) into a **blob** that may be stored in the flash or sent out, and when it is needed
  the blob is **decrypted straight into the PKA RAM** and used for the operation. The software only handles the blob and the public results (the public key, the signature).
- Typical uses are the **device identity** (the key pair generated at the production, the public key certified, the private key never leaves the chip), the **signing of the
  telemetry or the challenges** of a server, the **ECDH key agreement** for a secure channel, the **RSA / modular exponentiation** with a protected private exponent. A blob is bound to the
  device: copying it to another chip is useless.
- The sequences are strict (the order of the chained operations and the protection of the intermediate values) and the hardware does them under the control of the CCB; the module drives
  the CCB configuration and calls the **Pka** and **SAes** modules for their parts of the operation (their protected entry points), so every peripheral is still accessed only
  through its own module.
- The module hides the differences between the families: the supported operations, the blob format and the key sizes of the family differ, the interface stays the same.

What the module is **not**: it does not store the blobs (the application / the flash module), it does not manage the certificates or the key hierarchy, and it does not replace the Pka and Saes
modules for the operations without a protected private key.

---

## Concept

| Item | Meaning |
|---|---|
| **Operation** (`ccb_Operation_t`) | `CCB_OP_ECDSA_SIGN`, `CCB_OP_ECC_SCALAR_MUL` (ECDH), `CCB_OP_MOD_EXP` (RSA): the three protected PKA operations. |
| **Blob creation** (`Ccb_Set_BlobCreate`) | The one-time sequence: either **generate** a private key inside (from the RNG) or **import** a key given by the software (wrapped), derive the public key, and produce the blob (the key encrypted with the DHUK-derived key, with the integrity tag). Outputs: the blob, the public key. |
| **Blob usage** (`Ccb_Set_BlobUse`) | The repeated sequence: load the blob, decrypt it into the PKA RAM, execute the protected operation with the inputs of the caller (the digest, the peer public key, the message), return the public result. |
| **Curve / operand** | The `pka_Curve_t` of the Pka module (or the modulus size for the exponentiation). |
| **Subsystem state** | The tamper events of the cryptographic subsystem (`Ccb_Get_TamperFlags`) kept in the CCB; the software reset. |

Rules:

1. The module is used from the secure world and only on a device in the state that allows the hardware key (`Sbs_Get_ProductState`); otherwise `CCB_REQUEST_ERROR`.
2. `Ccb_Set_BlobCreate( request )` and `Ccb_Set_BlobUse( request )` are **non-blocking requests**: the module configures the CCB, starts the chained sequence and the completion callback
   reports the result with the operation status; the same request and the same callbacks are used in the polling and the interrupt mode.
3. For the ECDSA signature the random number `k` is produced **inside** by the chain from the RNG; the caller supplies only the digest. A failure of the RNG (the health test) aborts the
   sequence with `CCB_RESULT_RNG_ERROR` and nothing is output.
4. A blob that is corrupted, truncated, or made by another device is rejected by the integrity check (`CCB_RESULT_BLOB_INVALID`); the PKA RAM is erased after every sequence.
5. Only one sequence runs at a time (they use the PKA and the SAES); a request while another runs returns an error.
6. `Ccb_Set_Reset` resets the bridge (a software reset, clears the chaining state) after an error or a tamper event; the tamper flags are read by `Ccb_Get_TamperFlags` and
   cleared after the application has handled them.
7. All parameters are checked before a register is written.

---

## Supported families

Every family gets its own story, its own tasks and its own branches `Dev/<Family>` and `Releases/<Family>` cut from this
master.

| Family | Branch | Bridge | Status |
|---|---|---|---|
| STM32H5 | `Dev/STM32H5` | `CCB` on the lines with the `PKA` and the `SAES` (STM32H553 and H5F; the STM32H5E description in the SVD also lists it - to be confirmed in the family story): the optional coupling of the PKA RAM writes to the SAES input, the read and write chaining between the PKA RAM and the SAES, the RNG chaining, the sequences for the protected modular exponentiation, scalar multiplication and ECDSA signature, the blob creation and usage, the private key generation, the encryption of the blob key with the DHUK, the tamper flags | planned (first) |
| STM32U5, STM32U3 | `Dev/STM32U5`, `Dev/STM32U3` | device dependent | planned |
| STM32F4, STM32F7, STM32G4, STM32H7, STM32L4 | `Dev/STM32F4` ... | no coupling bridge on the device - the module is a stub that returns `CCB_REQUEST_ERROR` | planned |

A function without a meaning in a family stays in the interface, returns `CCB_REQUEST_ERROR` and suppresses the unused
parameters (for example an operation the bridge of the device does not support).

---

## Public interface in short (proposal)

| Group | Functions |
|---|---|
| Module management | `Ccb_Get_ModuleVersion`, `Ccb_Init`, `Ccb_Deinit`, `Ccb_Task`, `Ccb_Get_DefaultConfig` |
| Sequences | `Ccb_Set_BlobCreate`, `Ccb_Set_BlobUse`, `Ccb_Set_Abort`, `Ccb_Get_State`, `Ccb_Get_Result` |
| Status and recovery | `Ccb_Get_TamperFlags`, `Ccb_Set_TamperFlagsClear`, `Ccb_Set_Reset` |
| Capability | `Ccb_Get_Capability`, `Ccb_Get_BlobSize` |
| Interrupts | `Ccb_Set_IrqPriority` / `Get`, the callbacks are a part of the configuration |

```c
ccb_ModuleVersion_t Ccb_Get_ModuleVersion  ( void );

ccb_RequestState_t  Ccb_Init               ( const ccb_Config_t * const ccbConfig );
ccb_RequestState_t  Ccb_Deinit             ( void );
void                Ccb_Task               ( void );
ccb_RequestState_t  Ccb_Get_DefaultConfig  ( ccb_Config_t * const ccbConfig );

ccb_RequestState_t  Ccb_Get_Capability     ( ccb_Operation_t operation, ccb_FunctionState_t * const supported );
ccb_RequestState_t  Ccb_Get_BlobSize       ( ccb_Operation_t operation, const pka_Curve_t * const curve, ccb_Size_t * const blobSize );

ccb_RequestState_t  Ccb_Set_BlobCreate     ( const ccb_BlobCreateRequest_t * const request );
ccb_RequestState_t  Ccb_Set_BlobUse        ( const ccb_BlobUseRequest_t * const request );
ccb_RequestState_t  Ccb_Set_Abort          ( void );
ccb_RequestState_t  Ccb_Get_Result         ( ccb_Result_t * const result );

ccb_RequestState_t  Ccb_Get_TamperFlags    ( ccb_TamperMask_t * const tamperFlags );
ccb_RequestState_t  Ccb_Set_Reset          ( void );
```

Every function returns `CCB_REQUEST_OK` / `CCB_REQUEST_ERROR` (the request was accepted and started) and checks its parameters and the state of the module
before it touches a register. The outcome of a sequence is `ccb_Result_t`.

| Type | Meaning |
|---|---|
| `ccb_Config_t` | Transfer mode (`CCB_XFER_MODE_POLL` / `ISR`), callbacks, interrupt priority |
| `ccb_BlobCreateRequest_t` | Operation, curve, generate / import, the imported wrapped key, the output blob buffer, the output public key |
| `ccb_BlobUseRequest_t` | Operation, curve, the blob, the inputs (digest, peer public key, message) and the output buffers (signature, shared point, result) |
| `ccb_Result_t` | `CCB_RESULT_OK`, `CCB_RESULT_BLOB_INVALID`, `CCB_RESULT_RNG_ERROR`, `CCB_RESULT_OPERATION_ERROR`, `CCB_RESULT_TAMPER` |

---

## Usage

### Production: create the device identity key

```c
#include "Ccb_Port.h"

static ccb_Data_t identityBlob[ 128u ];                         /* stored in the flash, useless on another device */
static pka_EccPoint_t identityPublicKey;                        /* certified by the production system */

const ccb_BlobCreateRequest_t create =
{
    .Operation = CCB_OP_ECDSA_SIGN, .Curve = &PKA_CURVE_NIST_P256, .Source = CCB_KEY_GENERATE,
    .Blob = identityBlob, .BlobSize = sizeof( identityBlob ), .PublicKey = &identityPublicKey,
};

(void)Ccb_Set_BlobCreate( &create );                            /* the private key never left the chip */
```

### Run time: sign a digest with the protected key

```c
const ccb_BlobUseRequest_t use =
{
    .Operation = CCB_OP_ECDSA_SIGN, .Curve = &PKA_CURVE_NIST_P256,
    .Blob = identityBlob, .BlobSize = sizeof( identityBlob ),
    .Digest = digest, .DigestSize = 32u, .Signature = &signature,
};

(void)Ccb_Set_BlobUse( &use );                                  /* the callback reports CCB_RESULT_OK and the signature is ready */
```

---

## Behaviour and limits

- **Device bound.** The blob is encrypted with a key derived from the hardware secret of the chip; there is no way to recover the key from the blob on another device or after the
  chip is lost. Keep a backup strategy at the system level (a second identity, the provisioning server).
- **Product state.** The hardware keys exist only in the product states that provision them; a device in a different state cannot create or use blobs (`Sbs_Get_ProductState`).
- **Security context.** The sequences run from the secure world; the PKA and SAES must be assigned to it by the Gtzc module.
- **Time.** A blob creation with the key generation takes tens of milliseconds, a signature a few milliseconds; the sequences run without the CPU in the interrupt mode.
- **Not part of the first version:** the key import with the transport wrapping by another party (a key ceremony), the cross-device blob migration, the key attestation.

---

## Dependencies

- **Pka** - the protected PKA operations of the sequences
- **SAes** - the key wrapping with the DHUK-derived key
- **Rng** - the private key and the nonce generation
- **Sbs** - the product state check
- **Rcc** - peripheral clock and reset
- **Nvic** - interrupt
- **RAL (Register Abstraction Layer)** - used internally to access the registers

All mandatory RCC and NVIC configurations are handled internally.

---

## Testing

- **Unit tests** (host, Unity + CMock + RegMem): the request validation, the sequence order against the mocked Pka / SAes / Rng modules, the state machine (create, use, abort, the errors), the
  mapping of the hardware status flags to `ccb_Result_t`, the tamper flags and the reset, the rejected calls in the wrong state.
- **Integration tests** (target, probe-rs): the blob creation and the signature verified by the Pka module (ECDSA verify with the public key), the blob made on one device rejected on
  another, the damaged blob, the repeated use of one blob, the behaviour after a tamper flag.

---

## Interface conventions

The module follows the common rules of the MCAL layer, so that it behaves like the other modules:

- **Lists instead of free members.** Everything that the hardware restricts (the pins, the kernel clock sources, the DMA
  channels, the triggers and the other selectable sources) is a `typedef enum` list with exactly the items that are valid
  for the device line; a pin is an item `CCB_<SIGNAL>_PIN_<INSTANCE>_P<port><pin>` and there is an `*_UNUSED` item. A choice
  that is not in the list cannot be written.
- **Own peripheral only.** The module calls only the registers / LL functions of its own peripheral. The clocks, the
  interrupts, the pins, the DMA and the EXTI lines go through the API of the Rcc, Nvic, Gpio, Gpdma and Exti modules; a
  missing function is added to that module, it is not bypassed.
- **Init verifies itself.** `Ccb_Init` checks the result at the end; when it is not as required, the module is
  deinitialized and an error is returned. The waiting for the hardware is a loop on the register state with a timeout,
  never a blind delay.
- **Transfer modes.** Where the module moves data, the POLL (served by `Ccb_Task`), ISR and DMA modes have the same
  configuration structure, the same request and the same callbacks (error, half transfer, transfer complete).
- **Flags and states are enums of the module** (`ccb_FunctionState_t`, `ccb_FlagState_t`), the limits and the constants are
  named, there are no magic numbers in the interface.
- **Unsupported functions stay.** A function without a meaning in a family keeps its signature, suppresses the unused
  parameters and returns `CCB_REQUEST_ERROR`.
- **Documentation.** Every function and type has a Doxygen description, the main header has the `\defgroup`.

---

## Repository layout and branches

The master holds this README and the license. The family branches (`Dev/<Family>`, `Releases/<Family>`) hold:

| Path | Content |
|---|---|
| `README.md` | README of the family: the instances of the device, specifics, errata, status |
| `Ccb_Port.h`, `Ccb_Types.h` | Public interface (identical in all families) |
| `Ccb*.c`, `Ccb.h` | Implementation of the family |
| `CMakeLists.txt` | Library `Ccb_Lib` of the EmBi platform |
| `Tests/UnitTests` | Unit tests (Unity + CMock + RegMem) |
| `Tests/IntegrationTests` | Integration tests on the target (probe-rs) |

A change of the public interface is made in all families.

---

## CMake integration

```cmake
target_link_libraries( User_Lib PRIVATE Ccb_Lib )
```

1. Link `Ccb_Lib` to your CMake library.
2. Include `Ccb_Port.h` in your project.
3. Configure the module as needed for your hardware.

---

## License

This project is licensed under the **Creative Commons Attribution-NonCommercial 4.0 International (CC BY-NC 4.0)**.

You are free to use, modify, and share this work for **non-commercial purposes**, provided appropriate credit is given.

See [LICENSE.md](LICENSE.md) for full terms or visit [creativecommons.org/licenses/by-nc/4.0](https://creativecommons.org/licenses/by-nc/4.0/).

---

## Authors

- **Mr.Nobody** - [embedbits.com](https://embedbits.com)

Contributions are welcome! Please open a pull request.
