# 📻 Bare-Metal Audio Encryption System (RISC-V)


A bare-metal, real-time secure audio recording, encryption, and playback station engineered in C for a soft-core **RISC-V processor** on the **Intel Cyclone V SoC (DE1-SoC)**. 

The system implements low-level register MMIO drivers to interface directly with the onboard Wolfson WM8731 audio codec, PS/2 keyboard controller, VGA character/pixel buffers, and pushbuttons via Machine-Mode hardware interrupts. Encrypted audio is scrambled using a custom **AES-128 Counter (CTR) Mode** stream cipher keyed via PS/2 password authentication.


---

## 🎥 Hardware Demonstration

* **Real Hardware Execution:** [Download & View Demo Video (project_demo_video.mp4)](./project_demo_video.mp4)

---

## 🏛️ System Architecture

The firmware runs bare-metal without an operating system, executing machine instructions directly against memory-mapped hardware peripheral controllers mapped across the 32-bit address space.

![System Architecture](./system_architecture.png)

### Key Architectural Modules
* **Machine-Mode CSR Interrupt Handler:** Vector-intercepted hardware interrupts servicing edge-triggered pushbutton inputs via `mtvec`, `mcause`, `mie`, and `mstatus`.
* **Voice-Activated Audio Driver:** FIFO-managed DMA-like polling loops that monitor acoustic threshold levels to trigger autonomous 10-second stereo audio captures.
* **AES-128 CTR Stream Cipher Engine:** Real-time 128-bit block transformation applied as an additive XOR stream cipher over stereo audio memory buffers.
* **Dual-Buffer VGA Driver:** Direct VRAM frame-buffer rendering (320×240 RGB565 graphics) layered with an 80×60 memory-mapped ASCII character generator.
* **10-Slot Bank Storage Manager:** Multi-channel memory banking allowing slot-to-slot audio transfer and duplication through hardware switches `SW[9:0]`.

---

## 🔐 Cryptographic Engine: AES-128 CTR

The firmware implements the complete NIST Advanced Encryption Standard specification operating in **Counter (CTR) Mode**:

### 1. Zero-Latency Stream Transformation
Standard ECB/CBC block modes require block padding and introduce latency unsuitable for continuous streaming audio. CTR mode transforms AES into an **additive stream cipher**:
$$C_i = P_i \oplus \text{AES}_{K}(\text{Nonce} \parallel \text{Counter}_i)$$
* Encryption and decryption routines are mathematically identical: applying the same passphrase against scrambled audio runs the identical CTR keystream generation, XOR-restoring the original PCM waveform.
* Processes $80{,}000$ stereo audio samples per slot across 20,000 counter block increments without memory leaks or buffer overflows.

### 2. Key Derivation & Expansion
* **Passphrase Ingestion:** Passwords up to 10 characters are entered via a PS/2 keyboard interface and scrambled using an **FNV-1a 32-bit hash** with non-linear bitwise spreading across 16 bytes.
* **10-Round Key Expansion:** Expands the 16-byte master key into 176 bytes of round keys utilizing Rijndael S-box byte substitution, cyclic word rotations, and round constants ($R_{\text{con}}$).
* **Galois Field Arithmetic:** Column diffusion is evaluated over $GF(2^8)$ utilizing an optimized `xtime()` bitwise multiplier polynomial ($x^8 + x^4 + x^3 + x + 1 \equiv \text{0x11B}$).

---

## ⚡ Hardware Architecture & MMIO Map

The system bypasses operating system abstractions, reading and writing to physical registers via volatile pointers:

| Subsystem / Peripheral | Base Address | Register Offset / Width | Functionality |
| :--- | :--- | :--- | :--- |
| **Audio Core FIFO** | `0xFF203040` | `+0x0` Control, `+0x4` Fifospace, `+0x8` Left, `+0xC` Right | Dual-channel 16-bit PCM read/write FIFO queue |
| **Pushbuttons (KEYs)** | `0xFF200050` | `+0x0` Data, `+0x8` Interruptmask, `+0xC` Edgecapture | Hardware button interrupts and debouncing |
| **Slider Switches (SW)** | `0xFF200040` | 32-bit register (`0x3FF` mask) | Live active memory bank slot selection (0–9) |
| **Red LEDs** | `0xFF200000` | 32-bit register (`0x3FF` mask) | Passkey character count and recording timer countdown |
| **PS/2 Controller** | `0xFF200100` | `+0x0` Data / RVALID bit (`0x8000`) | Make/break scancode stream from keyboard |
| **VGA Pixel Buffer** | `0x08000000` | Address: `Base + (y << 10) + (x << 1)` | 320×240 16-bit RGB565 framebuffer graphics |
| **VGA Character Buffer** | `0x09000000` | Address: `Base + (y << 7) + x` | 80×60 ASCII character text generation overlay |

---

## 🎛️ Bare-Metal Interrupt Control (RISC-V Machine Mode)

Hardware responsiveness is driven through the RISC-V Machine-Mode Control and Status Registers (CSRs):

