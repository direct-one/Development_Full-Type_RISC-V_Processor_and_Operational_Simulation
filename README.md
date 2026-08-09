# RV32I Base Integer Single-Cycle CPU Architecture

본 프로젝트는 RISC-V ISA(RV32I Base Integer Instruction Set) 표준 규격을 엄격하게 준수하여 Verilog HDL로 구현한 단일 사이클(Single-Cycle) 마이크로프로세서입니다. 하드웨어 자원의 효율성과 디버깅 용이성을 확보하기 위해 Control Unit과 Datapath의 계층적 분리(Hierarchical Decoupling) 설계를 적용했습니다.

---

## 1. Top-Level Architecture & Data Flow

단일 사이클 아키텍처의 특성상, 명령어 인출(Fetch)부터 레지스터 쓰기(Write-Back)까지의 모든 데이터 흐름이 하나의 클럭 주기(`1 CLK`) 내에 안정적으로 완료되어야 합니다. 이를 위해 각 모듈 간의 전파 지연(Propagation Delay)을 최소화하는 직관적인 멀티플렉서 트리 구조를 채택했습니다.

### 1.1 시스템 전체 블록 다이어그램 (Overall Datapath)
메모리 버스, 레지스터 제어 신호, 그리고 ALU 연산의 유기적인 동기화 상태를 나타냅니다.

![RV32I Overall Block Diagram](https://github.com/user-attachments/assets/6a1ad6c0-9e89-41d0-824e-6e8b2a00695b)

### 1.2 핵심 하드웨어 모듈 명세
| 모듈명 | 주요 기능 및 세부 설계 특징 | 버스 폭 (Bus Width) |
| :--- | :--- | :--- |
| **Instruction Memory** | Program Counter(PC)를 인덱스로 사용하여 32-bit 기계어 코드를 지연 없이 출력하는 비동기 읽기 ROM | Address: 32-bit<br>Data: 32-bit |
| **Control Unit** | `opcode(7-bit)`, `funct3`, `funct7`을 분석하여 하드웨어 제어 매트릭스(Mux Select, Write Enable) 활성화 | Input: 17-bit<br>Output: 10+ bit |
| **Register File** | 32개의 32-bit 범용 레지스터($x0 \sim x31$). Dual-Port Read / Single-Port Write 아키텍처 | Data: 32-bit<br>Addr: 5-bit |
| **ALU** | 제어 신호(`ALUControl`)에 기반한 산술/논리/시프트 연산 및 분기(Branch) 성립 여부(`Zero` 플래그) 판별 | Data: 32-bit<br>Ctrl: 4-bit |
| **Imm Extender** | 명령어 포맷(I, S, B, U, J)별 Immediate 필드 추출 및 32-bit 부호 확장(Sign-Extension) 처리 | Input: 25-bit<br>Output: 32-bit |
| **Data Memory** | S-Type/I-Type 명령어 처리를 위한 Byte-Addressable 정격 RAM. 바이트 정렬 및 마스킹 하드웨어 포함 | Address: 32-bit<br>Data: 32-bit |

---

## 2. RTL Implementation Deep Dive

단순한 논리 게이트의 조합을 넘어, 실제 실리콘(Silicon) 상에서의 동작 무결성을 보장하기 위한 하드웨어 예외 처리 및 최적화 로직이 포함되어 있습니다.

### 2.1 Instruction Execution Flow & Assembly
컴파일러를 통해 생성된 `.mem` 바이너리 파일은 하드웨어 인스턴스화 과정에서 Memory Array로 직접 로드됩니다.

![Instruction Memory Flow 1](https://github.com/user-attachments/assets/6d5ab2ff-8cb5-484b-94d7-017dbaecbc80)
![Instruction Memory Flow 2](https://github.com/user-attachments/assets/bf2a75bd-cd99-480a-9c0c-8f4a4737157b)

### 2.2 Control Unit: Two-Stage Decoding Mechanism
단일 거대 조합 회로가 아닌, 메인 디코더(Main Decoder)와 ALU 디코더(ALU Decoder)의 2단 분리 구조를 채택하여 논리 합성(Synthesis) 시 게이트 카운트(Gate Count)를 최적화했습니다.

![Control Unit Specification](https://github.com/user-attachments/assets/bf9773bd-e441-41cb-8d40-a15af42ea704)

### 2.3 Register File: Hardwired Zero ($x0) Interlock
RISC-V 아키텍처의 필수 제약인 `$x0 = 0` 원칙을 물리적 하드웨어 레벨에서 강제하기 위해 쓰기 보호 회로(Write Protection Logic)를 구현했습니다.

```verilog
// x0 레지스터 보호용 인터록 하드웨어 제어 로직
assign final_we = (!rst && rf_we && (WA != 5'd0));

always @(posedge clk) begin
    if (final_we) begin
        registers[WA] <= WD;
    end
end
```

---

## 3. Data Memory & Byte-Alignment (Critical Path)

단일 사이클 CPU에서 가장 긴 전파 지연(Critical Path)을 가지는 메모리 접근 구간입니다. 데이터 무결성을 위해 `funct3` 신호를 활용한 정밀한 비트 슬라이싱 및 부호 확장이 수행됩니다.

### 3.1 S-Type (Store Operations)
레지스터 데이터를 메모리 주소 공간에 기록합니다. (`MemWrite = 1`)
* **SW (`funct3 = 3'b010`):** 32-bit 워드 전체 매핑.
* **SH (`funct3 = 3'b001`):** `WD[15:0]` 하위 16비트 매핑 (Half-word 정렬).
* **SB (`funct3 = 3'b000`):** `WD[7:0]` 하위 8비트 매핑 (Byte 정렬).

### 3.2 I-Type (Load Operations & Sign Extension)
메모리 장치에서 버스로 데이터를 적재합니다. 하드웨어 확장기가 병렬 동작합니다.
* **LW (`funct3 = 3'b010`):** 32-bit 원본 데이터 그대로 적재.
* **LH (`funct3 = 3'b001`):** 16-bit 로드 후 최상위 부호 비트(MSB) 기반 32-bit 복제 확장.
* **LB (`funct3 = 3'b000`):** 8-bit 로드 후 최상위 부호 비트 기반 32-bit 복제 확장.

---

## 4. Constant Definitions (`defines.vh`)

제어 유닛의 하드코딩을 방지하고 가독성을 극대화하기 위해, 오프코드(Opcode) 및 연산 코드를 전역 헤더 파일로 완전 모듈화했습니다.

![Verilog Header Defines](https://github.com/user-attachments/assets/3becb2b5-1482-40ca-a7ec-c9ea110418df)

---

## 5. Verification & Simulation Environment

### 5.1 Test Scenario (C to Assembly)
반복문 루프(i loop)를 이용한 누적합(Summation) 알고리즘을 크로스 컴파일하여 하드웨어 검증용 시나리오로 채택했습니다. 분기(Branch) 명령어와 산술(Arithmetic) 명령어의 복합적인 타이밍을 검증할 수 있습니다.

![Simulation Test Scenario](https://github.com/user-attachments/assets/570b931f-a5eb-4712-bc44-72e7609a8b1b)

### 5.2 Testbench Execution Steps
1. **RTL Simulation:** Xilinx Vivado 환경에서 `tb_top.v`를 최상위 모듈로 설정합니다.
2. **Memory Initialization:** 컴파일된 `imem.mem` 파일이 Instruction Memory 경로에 정확히 마운트되었는지 확인합니다.
3. **Waveform Analysis:** 초기 2클럭 동안 `rst = 1`을 인가하여 시스템을 초기화한 후, 매 클럭마다 `PC`의 순차적 증가와 `Register File` 내부 데이터의 변화를 파형(Waveform)으로 관찰합니다.

## Simulation

### Overview
<img width="1450" height="587" alt="image" src="https://github.com/user-attachments/assets/a3d946ad-e100-4e84-9700-5aa08164ea61" />

### I-Type
<img width="1184" height="625" alt="image" src="https://github.com/user-attachments/assets/66364c3e-cca2-438a-87da-9ded04946ae1" />
<img width="1292" height="560" alt="image" src="https://github.com/user-attachments/assets/3328df76-1601-4039-b6d2-bf242d73d0bd" />

### R-Type
<img width="1301" height="576" alt="image" src="https://github.com/user-attachments/assets/f76eaa70-3a2a-46fc-9dfc-fe6e2939264d" />

### IL-Type
<img width="1306" height="637" alt="image" src="https://github.com/user-attachments/assets/a6684981-9514-499c-a7d0-38f2175a093f" />

### S-Type
<img width="1364" height="596" alt="image" src="https://github.com/user-attachments/assets/11abbed1-3393-4d6e-b1c1-d4cccfd58bb7" />

### B-Type
<img width="1336" height="674" alt="image" src="https://github.com/user-attachments/assets/c6bd9ef3-7ab7-45ad-9016-fc2cc26aa525" />
<img width="1334" height="622" alt="image" src="https://github.com/user-attachments/assets/a9832150-6ab2-4c1b-99e6-e150293e4dc8" />

### U-Type
<img width="1359" height="662" alt="image" src="https://github.com/user-attachments/assets/f2a9e57d-ecd7-4260-a746-50d63d5541c5" />

### J-type
<img width="1406" height="625" alt="image" src="https://github.com/user-attachments/assets/76dff4db-f0dc-4186-90a6-54c1135f93a6" />









