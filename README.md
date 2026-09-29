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
