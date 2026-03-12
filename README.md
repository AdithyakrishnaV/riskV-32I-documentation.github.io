# RISC-V RV32I (32-bit address space)

## 🏗️ The 5 Layers (bottom to top)
```bash
┌─────────────────────────────┐
│  5. PROGRAMS (ELF binaries) │  ← what you run
├─────────────────────────────┤
│  4. SYSCALLS (ECALL)        │  ← how programs talk to outside world
├─────────────────────────────┤
│  3. INSTRUCTIONS (execute)  │  ← what the CPU does
├─────────────────────────────┤
│  2. DECODE                  │  ← figuring out what instruction it is
├─────────────────────────────┤
│  1. MEMORY + REGISTERS      │  ← the raw hardware state
└─────────────────────────────┘
You build bottom up. Layer 1 first, layer 5 last.
```
## 📋 Every function you will write — in order

### LAYER 1 — Foundation
```bash
RISCVState struct
│
├── memory[]     → the RAM
├── regs[32]     → register file  
└── pc           → program counter

init()           → zero everything, set pc to start address

mem_read8()      → read 1 byte  from memory[addr]
mem_read16()     → read 2 bytes from memory[addr]
mem_read32()     → read 4 bytes from memory[addr]  ← you wrote this
mem_write8()     → write 1 byte  to memory[addr]
mem_write16()    → write 2 bytes to memory[addr]
mem_write32()    → write 4 bytes to memory[addr]   

reg_read()       → return regs[i]  (x0 always returns 0)
reg_write()      → regs[i] = val   (x0 write is ignored)
```

### LAYER 2 — Decode
```bash
cpu_step()       → one full clock cycle
│
├── FETCH:   instr = mem_read32(pc)
│
├── DECODE:  extract fields from the 32-bit instruction word
│            opcode = instr & 0x7F
│            rd     = (instr >> 7)  & 0x1F
│            rs1    = (instr >> 15) & 0x1F
│            rs2    = (instr >> 20) & 0x1F
│            funct3 = (instr >> 12) & 0x7
│            funct7 = (instr >> 25) & 0x7F
│
└── EXECUTE: switch(opcode) → call the right handler
```

### LAYER 3 — Execute (one function per opcode group)
```bash
exec_rtype()     → ADD SUB AND OR XOR SLL SRL SRA SLT SLTU
│                  opcode 0x33
│                  uses: rd, rs1, rs2, funct3, funct7
│
exec_itype()     → ADDI ANDI ORI XORI SLTI SLTIU SLLI SRLI SRAI
│                  opcode 0x13
│                  uses: rd, rs1, imm12, funct3
│
exec_load()      → LB LH LW LBU LHU
│                  opcode 0x03
│                  does: rd = memory[rs1 + imm]
│
exec_store()     → SB SH SW
│                  opcode 0x23
│                  does: memory[rs1 + imm] = rs2
│
exec_branch()    → BEQ BNE BLT BGE BLTU BGEU
│                  opcode 0x63
│                  does: if condition → pc += offset (does NOT pc+4)
│
exec_jal()       → JAL
│                  opcode 0x6F
│                  does: rd = pc+4, pc += offset
│
exec_jalr()      → JALR
│                  opcode 0x67
│                  does: rd = pc+4, pc = rs1 + imm
│
exec_lui()       → LUI
│                  opcode 0x37
│                  does: rd = imm << 12
│
exec_auipc()     → AUIPC
│                  opcode 0x17
│                  does: rd = pc + (imm << 12)
```

### LAYER 4 — Syscalls
```bash
exec_ecall()     → opcode 0x73
│
├── reads a0 (x10) to know which syscall
│
├── case 1  → write()  → print string to terminal
├── case 10 → exit()   → stop the emulator
└── case 93 → exit()   → Linux exit syscall
```

### LAYER 5 — Program Loader
```bash
load_elf()       → reads a compiled .elf binary file
│
├── open the file
├── read ELF header → verify it's RISC-V 32-bit
├── read program headers → find loadable segments
├── copy each segment → memcpy into memory[] at correct address
└── set pc = entry point from ELF header
```


## 🔄 The main loop — how it all connects
```bash
main()
│
├── init()              → zero the CPU
├── load_elf(filename)  → load program into memory
│
└── loop forever:
    │
    ├── cpu_step()
    │   ├── fetch
    │   ├── decode
    │   └── execute → exec_rtype / exec_load / exec_branch / etc
    │
    └── stop when ECALL exit is hit

```

## 📐 Rule for every exec function
Every exec function follows the same pattern:
1. extract the immediate (if needed)
2. read source registers (rs1, rs2)
3. compute the result
4. write to destination register (rd)
5. branches/jumps: set pc directly and RETURN (skip pc+4)
   everything else: do nothing, cpu_step adds pc+4
