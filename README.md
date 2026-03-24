# Single-Cycle-CPU-FPGA
實作 https://safari.ethz.ch/ddca/spring2025/doku.php?id=start 的 Lab 8  
根據 ddca_ss25_lab8-1_manual.pdf:  
修改 MIPS.v  
根據 ddca_ss25_lab8-2_manual.pdf:  
修改 top.v, top.xdc, snake_patterns.asm, insmem_h.txt, datamem_h.txt 
## 1. 修改 MIPS.v
### 第 1 修改點
```verilog
InstructionMemory i_imem (
                              .A(PC[7:2]),      // 根據手冊，使用 PC 的位元 7 到 2 作為位址 [cite: 89]
                              .RD(Instr)        // 讀取出的指令存入 Instr 訊號中
                         );
```
### 第 2 修改點
對應 ALU.v 裡的宣告  
```verilog
ALU i_alu (
                // TODO Part 1
                .a(SrcA), 
                .b(SrcB), 
                .aluop(ALUControl[3:0]),
                .result(ALUResult), 
                .zero(Zero)
                );
```
### 第 3 修改點
對應 DataMemory.v 裡的宣告
```verilog
DataMemory i_dmem (
                        // TODO Part 1
                        .CLK(CLK), 
                        .WE(DataMemWrite),
                        .A(ALUResult[7:2]),
                        .WD(WriteData), 
                        .RD(ReadData)
                        );
```
### 第 4 修改點
判斷是要寫入內部 Data Memory 還是外部 I/O  
```verilog
assign DataMemWrite  = MemWrite & (~IsIO); // Is 1 when there is a SW instruction on DataMem address
assign IOWriteData = WriteData;      // This line is connected directly to WriteData
assign IOAddr      = ALUResult[3:0]; // The LSB 4 bits of the Address is assigned to IOAddr
assign IOWriteEn   = MemWrite & IsIO; // Is 1 when there is a SW instruction on IO address
```
### 第 5 修改點
對應 ControlUnit.v 裡的宣告
```verilog
ControlUnit i_cont (
                        //TODO Part 1
                        .Op(Instr[31:26]),
                        .Funct(Instr[5:0]),
                        .MemtoReg(MemtoReg),
                        .MemWrite(MemWrite),
                        .Branch(Branch),
                        .ALUSrc(ALUSrc),
                        .RegDst(RegDst),
                        .RegWrite(RegWrite),
                        .Jump(Jump),
                        .ALUControl(ALUControl)
                       );
```
## 2. 修改 top.v
### 第 1 修改點
加一個 SW
```verilog
module top(
        input             FPGACLK,
        // TODO PART II for Lab 8
        input      [1:0]  SW,      // 加switch開關
        input             RESET,
        output     [6:0]  LED,
        output reg [3:0]  AN
    );
```
### 第 2 修改點
當 IOAddr 等於 4 (對應 0x7FF4)，讀取 SW 的值，否則預設為 0 (因為開關對應的位址是 0x00007FF4，所以當位址的最後 4 個 bits (IOAddr) 等於 4 時，就把開關數值傳回去)
```verilog
assign IOReadData = (IOAddr == 4'h4) ? {30'b0, SW} : 32'h0;
```
## 3. 修改 top.xdc
兩個開關的腳位綁定
```verilog
set_property PACKAGE_PIN V16 [get_ports {SW[1]}]
set_property IOSTANDARD LVCMOS33 [get_ports {SW[1]}]

set_property PACKAGE_PIN V17 [get_ports {SW[0]}]
set_property IOSTANDARD LVCMOS33 [get_ports {SW[0]}]
```
## 4. 修改 snake_patterns.asm
改 foward 和 wait 的部分
```mips
forward:
   beq $t5,$t4, restart
   lw $t0,0($t4)
   sw  $t0, 0x7ff0($0)

   addi $t4, $t4, 4 

   lw $s0, 0x7FF4($0)
   
   addi $t7, $0, 1
   beq $s0, $0, set_done
   
   addi $t1, $0, 1
   addi $t7, $0, 2
   beq $s0, $t1, set_done
   
   addi $t1, $0, 2
   addi $t7, $0, 4
   beq $s0, $t1, set_done
   
   addi $t7, $0, 8
set_done:

   addi $t2, $0, 0

wait:
   slt $t1, $t2, $t3     
   beq $t1, $0, forward  
   add $t2, $t2, $t7     
   j wait
```
