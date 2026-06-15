# RV32I -TYPE

## Instuction_memory

: Memory that composes and executes scenarios, or a storage space where instructions of the program to be executed are stored. 

1. Operate through ASM (Assembly Code) via Memory_file.

<img width="571" height="378" alt="image 1" src="https://github.com/user-attachments/assets/6d5ab2ff-8cb5-484b-94d7-017dbaecbc80" />


1.Implementation using a Memory_file to execute Assembly Code logic.

<img width="852" height="345" alt="image 2" src="https://github.com/user-attachments/assets/bf2a75bd-cd99-480a-9c0c-8f4a4737157b" />


## RV32I_cpu

－ A structure composed of a Control Unit and a Datapath.

- Control_unit 
- Responsible for defining control signals for each instruction Type.

<img width="317" height="602" alt="image 3" src="https://github.com/user-attachments/assets/bf9773bd-e441-41cb-8d40-a15af42ea704" />


1.  DataPath 
- Receives signals from the Control Unit as inputs and executes specific operations based on those signals.

## RV32I_Datapath

— Composed of the Register File, PC (Program Counter), Imm Extender, and ALU.

- Register_file 
    - input : RA1, RA2, RD, WD(wirte data), Wdata, rf_we(register file wirte enable) 
    - output: RD1, RD2
    - Saves values based on rf_we and WA (Write Address) signals.
    Condition:  (!rst & rf_we & WA ≠ 5’d0)

- ALU: A module that performs operations automatically. 
    -  Executes designated instructions for R-Type and B-Type by checking the 
    - R-Type
    : ADD, SUB,SLL,SLT, etc 
    - B-Type
    :BEQ, BNE, BLT, etc

- PC(Program Counter)
: Indicates the current instruction address and handles target addresses for B-Type or J-Type jumps.

    - Includes an adder for PC+4 calculations.
    - Includes a Register to temporarily store the PC value.
    -  Uses a 2x1 Mux to select between rd1 and PC values.
    - Uses a 2x1 Mux to select between the PC+4 value and the jump/branch target.

- Imm extender
: Used to handle constant values (Immediates) for specific instruction types (I, B, U, J).

## Data_Memory

: Byte-addressable storage structure — Used for S-Type and I-Type (Load).

- S-Type

    : funct3 == 101(SW)
    - 32bit store
    
    : funct3 == 001(SH)
    - 16bit store
    - Check if a number is odd or even 
    
    : funct3 == 000(SB)
    - 8bit store
    
- IL-Type(Load)
    
    
    : funct3 == 000(LB)
    - 32bit store
    
    : funct3 == 001(LH)
    - 16bit store
    - Check if a number is odd or even 
    
    : funct3 == 010(LW)
    - 8bit store
    

## Define(Verilog Header)

Defines Opcodes, R-Type instructions, and B-Type instructions for each instruction format.

<img width="543" height="489" alt="image 4" src="https://github.com/user-attachments/assets/3becb2b5-1482-40ca-a7ec-c9ea110418df" />


## Memory_File

: A C-code program using an i loop to output a sum value, converted into ASM (Assembly) code for execution.

<img width="543" height="489" alt="image 4" src="https://github.com/user-attachments/assets/570b931f-a5eb-4712-bc44-72e7609a8b1b" />

## Overview (Blcock Diagram)
<img width="1278" height="1094" alt="image" src="https://github.com/user-attachments/assets/6a1ad6c0-9e89-41d0-824e-6e8b2a00695b" />
   

