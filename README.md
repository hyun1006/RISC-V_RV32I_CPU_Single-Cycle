# 🚀 SystemVerilog RISC-V RV32I Processor

<div align="center">

<img src="https://img.shields.io/badge/Architecture-RISC--V_RV32I-purple?style=for-the-badge&logo=riscv" />
<img src="https://img.shields.io/badge/Language-SystemVerilog-green?style=for-the-badge&logo=systemverilog" />
<img src="https://img.shields.io/badge/Implementation-Single_Cycle-blue?style=for-the-badge" />
<img src="https://img.shields.io/badge/Platform-Xilinx_Vivado-red?style=for-the-badge&logo=xilinx" />

**32-bit RISC-V Instruction Set Architecture (ISA) Implementation**<br>
단일 사이클(Single-Cycle) 구조의 CPU 코어와 Harvard Architecture 기반의 메모리 서브시스템 설계

</div>

---

## 📖 1. 프로젝트 개요 (Overview)

이 프로젝트는 **SystemVerilog**를 사용하여 **RISC-V RV32I (Base Integer Instruction Set)** 아키텍처를 하드웨어 레벨에서 구현한 프로세서 설계입니다.
CPU 코어(`CPU_RV32I`)는 제어 유닛(Control Unit)과 데이터 패스(DataPath)로 명확히 분리되어 있으며, 최상위 모듈인 `MCU`에서 명령어 메모리(ROM)와 데이터 메모리(RAM)를 통합하여 실제 임베디드 어플리케이션 실행이 가능한 구조를 갖추고 있습니다.

### ✨ 핵심 설계 특징 (Key Features)
* **Complete RV32I ISA:** 산술/논리(ALU), 메모리(Load/Store), 분기(Branch), 점프(Jump) 등 32비트 정수 명령어 셋을 완벽히 지원합니다.
* **Single-Cycle Microarchitecture:** 모든 명령어가 1 클럭 사이클 내에 Fetch, Decode, Execute, Memory, Writeback 단계를 완료합니다.
* **Modular Control Logic:** 명령어의 Opcode를 분석하여 ALU 제어, 레지스터 쓰기, 분기 신호 등을 생성하는 9-bit 제어 신호 벡터를 생성합니다.
* **Versatile Memory Subsystem:**
    * **ROM:** 초기화된 헥사 코드를 통한 프로그램 실행.
    * **RAM:** Byte(8-bit), Half-word(16-bit), Word(32-bit) 단위의 정밀한 읽기/쓰기 및 부호 확장(Sign Extension) 지원.

---

## 🏗️ 2. 시스템 아키텍처 (System Architecture)

### 2.1 MCU Top-Level Diagram
MCU는 **Harvard Architecture**와 유사하게 명령어 버스와 데이터 버스가 분리되어 동작합니다.

```mermaid
graph TD
    subgraph "MCU (Micro Controller Unit)"
        ROM["Instruction Memory (ROM)"] -->|instrCode| CPU
        CPU["RISC-V CPU Core (RV32I)"] -->|instrAddr| ROM
        
        CPU -->|busAddr| RAM["Data Memory (RAM)"]
        CPU -->|busWData| RAM
        CPU -->|we / strb| RAM
        RAM -->|busRData| CPU
    end
````

### 2.2 CPU Internal Microarchitecture

CPU 내부는 제어 신호를 생성하는 **Control Unit**과 실제 연산을 수행하는 **Data Path**로 구성됩니다.

```mermaid
graph LR
    Input[Instruction Code] -->|Opcode| CU[Control Unit]
    Input -->|rs1, rs2, rd, imm| DP[Data Path]
    
    subgraph "CPU Core Logic"
        CU -->|ALU Control| ALU[ALU]
        CU -->|RegFile WE| RF[Register File]
        CU -->|Branch/Jump| PC[PC Logic]
        CU -->|ImmSel| EXT[Imm Extender]
        
        RF <==>|"Operands"| ALU
        EXT -->|"Immediate"| ALU
        ALU -->|"Result / Address"| Output[Data Bus]
    end
```

-----
## 💻 3. Module Design Details

### 3.1 Control Unit Design

`ControlUnit.sv`는 입력된 명령어의 7-bit Opcode를 해독하여 시스템 전반을 제어합니다.

  * [cite_start]**Decoding Logic:** `Case` 문을 사용하여 R, I, S, L, B, LU, AU, J, JL 타입을 판별합니다 [cite: 54-57].
  * [cite_start]**Signal Generation:** `regFileWe`, `aluSrcMuxSel`, `branch`, `jal`, `jalr` 등 핵심 제어 신호를 9비트 벡터로 통합 관리합니다[cite: 51].
  * [cite_start]**ALU Control:** `funct3`와 `funct7` 필드를 조합하여 `ADD`, `SUB`, `SLL`, `SRA` 등의 구체적인 연산 코드를 ALU로 전달합니다 [cite: 57-60].

### 3.2 Data Path & ALU

`DataPath.sv`와 `alu.sv`는 실제 데이터 처리를 담당합니다.

  * [cite_start]**Program Counter (PC):** `JAL`, `JALR`, `Branch` 발생 시 다음 주소를 계산하는 MUX와 Adder 로직을 포함합니다 [cite: 636-640].
  * [cite_start]**ALU Operations:** 덧셈/뺄셈뿐만 아니라 논리 연산(AND, OR, XOR), 시프트(SLL, SRL, SRA), 비교(SLT, SLTU)를 수행합니다 [cite: 646-653].
  * [cite_start]**Immediate Extension:** 명령어 포맷에 따라 흩어져 있는 즉시값(Immediate) 비트들을 모아 32비트로 부호 확장합니다[cite: 669].

### 3.3 Memory Interface (RAM)

`RAM.sv`는 `strb` (Strobe) 신호를 통해 다양한 데이터 크기를 처리합니다.

  * [cite_start]**Store Logic:** `SB`(Byte), `SH`(Half), `SW`(Word)에 따라 메모리의 특정 바이트 레인에만 데이터를 씁니다 [cite: 677-680].
  * [cite_start]**Load Logic:** `LB`, `LH` 명령어 수행 시 MSB를 상위 비트로 복사하는 **Sign Extension**을 수행하고, `LBU`, `LHU` 시에는 0으로 채우는 **Zero Extension**을 수행합니다 [cite: 681-691].

-----

## 💻 4. 상세 기능 명세 및 동작 원리 (Detailed Specification)

각 명령어 타입별 \*\*데이터 흐름(Data Flow)\*\*과 **제어 신호(Control Signal)** 동작 방식입니다.

### 4.1 R-Type (Register-Register)

레지스터 간의 산술 및 논리 연산을 수행합니다.

  * **Instructions:** `ADD`, `SUB`, `SLL`, `SLT`, `XOR`, `SRL`, `OR`, `AND` 등.
  * **Data Flow:**
    1.  ROM에서 명령어를 인출합니다.
    2.  Register File에서 `rs1`, `rs2` 데이터를 읽어 ALU로 전달합니다.
    3.  ALU 연산 결과가 MUX(0번 입력)를 통해 다시 Register File(`rd`)에 저장됩니다.
  * **Control Signals:**
      * `reg_wr_en = 1`: 연산 결과를 저장하기 위해 활성화.
      * `aluSrcMuxSel = 0`: 두 번째 피연산자로 레지스터값(`rs2`) 선택.
      * `RegWdataSel = 0`: ALU 결과를 저장 데이터로 선택.

### 4.2 I-Type (Immediate / Load)

상수 연산 또는 메모리 로드 명령을 수행합니다.

  * **Instructions:** `ADDI`, `ANDI`, `LB`, `LW`, `JALR` 등.
  * **Data Flow (Arithmetic):**
    1.  `rs1` 값과 확장된 `imm` 값이 ALU에서 연산됩니다.
    2.  결과가 Register File에 저장됩니다.
  * **Data Flow (Load):**
    1.  ALU에서 `rs1 + imm` 주소를 계산합니다.
    2.  RAM의 해당 주소 데이터를 읽어 MUX(1번 입력)를 통해 Register File에 저장합니다.
  * **Control Signals (Load):**
      * `reg_wr_en = 1`: 데이터 저장을 위해 활성화.
      * `aluSrcMuxSel = 1`: 주소 계산을 위해 상수(`imm`) 선택.
      * `RegWdataSel = 1`: 메모리에서 읽은 데이터(`busRData`) 선택.

### 4.3 S-Type (Store)

레지스터의 값을 메모리에 저장합니다.

  * **Instructions:** `SB` (Byte), `SH` (Half), `SW` (Word).
  * **Data Flow:**
    1.  `rs1 + imm`을 통해 저장할 메모리 주소를 계산합니다.
    2.  `rs2`의 값을 RAM의 데이터 포트로 전달합니다.
  * **Control Signals:**
      * `d_wr_en = 1`: RAM 쓰기 활성화.
      * `aluSrcMuxSel = 1`: 주소 계산용 상수 선택.

### 4.4 B-Type (Branch)

조건부 분기를 수행합니다.

  * **Instructions:** `BEQ`, `BNE`, `BLT`, `BGE` 등.
  * **Data Flow:**
    1.  비교기(Comparator)가 `rs1`과 `rs2`를 비교하여 `b_taken` 신호를 생성합니다.
    2.  `b_taken`이 참이면 `PC = PC + imm`, 거짓이면 `PC = PC + 4`로 업데이트됩니다.
  * **Control Signals:**
      * `branch = 1`: 분기 명령어임을 알림.
      * `aluSrcMuxSel = 0`: 비교를 위해 레지스터값 선택.

### 4.5 U-Type (Upper Immediate)

상위 20비트 상수를 처리합니다.

  * **Instructions:** `LUI`, `AUIPC`.
  * **Data Flow:**
    1.  20비트 `imm`을 32비트로 확장(하위 12비트 0)합니다.
    2.  `LUI`: 확장된 값을 그대로 저장. `AUIPC`: `PC + imm` 값을 저장.
  * **Control Signals (LUI):**
      * `RegWdataSel = 2`: ALU를 거치지 않은 Immediate 값 선택.

### 4.6 J-Type (Jump)

무조건 점프 및 복귀 주소 저장을 수행합니다.

  * **Instructions:** `JAL`, `JALR`.
  * **Data Flow:**
    1.  점프할 주소(`PC + imm` 또는 `rs1 + imm`)를 계산하여 PC를 업데이트합니다.
    2.  복귀 주소(`PC + 4`)를 Register File에 저장합니다.
  * **Control Signals:**
      * `jal = 1`: PC 점프 활성화.
      * `RegWdataSel = 4`: `PC + 4` 값을 저장 데이터로 선택.

-----

## 📜 5. 지원 명령어 셋 (Supported ISA)

`defines.sv`에 정의된 지원 명령어 목록입니다.

| Type | Opcode | Instructions | Description |
| :---: | :---: | :--- | :--- |
| **R-Type** | `0110011` | ADD, SUB, SLL, SLT, SLTU, XOR, SRL, SRA, OR, AND | Register-Register 산술/논리 연산 |
| **I-Type** | `0010011` | ADDI, SLTI, SLTIU, XORI, ORI, ANDI, SLLI, SRLI, SRAI | Immediate 산술/논리 연산 |
| **I-Type** | `0000011` | LB, LH, LW, LBU, LHU | 메모리 로드 (Load) |
| **I-Type** | `1100111` | JALR | 레지스터 기반 점프 |
| **S-Type** | `0100011` | SB, SH, SW | 메모리 저장 (Store) |
| **B-Type** | `1100011` | BEQ, BNE, BLT, BGE, BLTU, BGEU | 조건부 분기 (Branch) |
| **U-Type** | `0110111` | LUI, AUIPC | 상위 비트 로드 |
| **J-Type** | `1101111` | JAL | 점프 및 링크 |

-----

## 📂 6. 디렉토리 구조 (Directory Structure)

```text
📦 RISCV-RV32I-Project
 ┣ 📂 src
 ┃ ┣ 📂 core
 ┃ ┃ ┣ 📜 CPU_RV32I.sv       # [Top] CPU Core Wrapper
 ┃ ┃ ┣ 📜 ControlUnit.sv     # Instruction Decoder & Control
 ┃ ┃ ┣ 📜 DataPath.sv        # Registers, ALU, MUX wiring
 ┃ ┃ ┣ 📜 alu.sv             # Arithmetic Logic Unit
 ┃ ┃ ┣ 📜 RegisterFile.sv    # 32 x 32-bit Register Bank
 ┃ ┃ ┣ 📜 immExtend.sv       # Immediate Generator
 ┃ ┃ ┗ 📜 defines.sv         # Opcode & ALU Function Definitions
 ┃ ┣ 📂 memory
 ┃ ┃ ┣ 📜 ROM.sv             # Instruction Memory (Code Storage)
 ┃ ┃ ┗ 📜 RAM.sv             # Data Memory (Stack/Heap)
 ┃ ┗ 📜 MCU.sv               # [System Top] Processor + Memory Integration
 ┣ 📂 sim
 ┃ ┗ 📜 tb_rv32i.sv          # Testbench for Full System Verification
 ┗ 📜 README.md              # Project Documentation
```

-----

## 🚀 7. 시뮬레이션 및 검증 (Simulation)

### 테스트벤치 개요 (`tb_rv32i.sv`)

테스트벤치는 `MCU` 모듈을 인스턴스화하고 클럭(`clk`)과 리셋(`reset`) 신호를 공급합니다.
[cite_start]`ROM.sv` 파일 내부에는 검증을 위한 어셈블리 코드(ADD, SUB, AND, OR, Load/Store, Jump 등)가 초기화되어 있어, 시뮬레이션 시작과 동시에 프로그램이 실행됩니다 [cite: 143-162].

-----

> *Designed with SystemVerilog for RISC-V Architecture Study*

```
```
