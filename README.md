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

   lw $s0, 0x7FF4($0)  # 讀取開關數值
   
   addi $t7, $0, 1  # 設速度 = 1
   beq $s0, $0, set_done
   
   addi $t1, $0, 1
   addi $t7, $0, 2  # 若 SW==1，設速度 = 2
   beq $s0, $t1, set_done
   
   addi $t1, $0, 2
   addi $t7, $0, 4  # 設速度 = 4
   beq $s0, $t1, set_done

   # 如果上面都沒有beq，代表開關值一定是 3
   addi $t7, $0, 8  # 設速度 = 8
set_done:

   addi $t2, $0, 0  # clear $t2 counter

wait:
   slt $t1, $t2, $t3  #(If $t2 < $t3, then $t1 = 1, else $t1 = 0)
   beq $t1, $0, forward  
   add $t2, $t2, $t7     
   j wait
```
將修改完後的 snake_patterns.asm 用 MARS 組譯:  
* 在 MARS 中寫好程式後，點擊上方的 Run -> Assemble 完成組譯。  
* 點選選單列的 File -> Dump Memory...。  
* 在跳出的視窗中，Memory Segment 選擇 .text; Dump Format 選擇 Hexadecimal Text。
* 點擊 Dump To File... 並將檔案命名為 insmem_h.txt。
* 確保 insmem_h.txt 剛好有 64 行 ( 補0或刪掉多餘的 )。
* 再點選Dump Memory。
* Memory Segment 改選 .data; Dump Format 一樣選 Hexadecimal Text。
* Dump To File... 將檔案命名為 datamem_h.txt。
* 一樣確保 datamem_h.txt 剛好有 64 行。
* 將處理好的兩個 .txt 檔案覆蓋 Vivado 專案資料夾中原本提供的舊檔案。
## 5. Vivado
較新版本的 Vivado 不支援 .ngc 格式，並移除了 ngc2edif 轉換工具。新版 Vivado 合成器可把標準語法推導並合成為 FPGA 內部的記憶體。
1. 從 Vivado 的 Sources 視窗中 Remove reg_half.v 和 reg_half.ngc 這兩個原本提供的檔案。  
2. 改寫 RegisterFile.v (直接覆蓋原檔):
```verilog
`timescale 1ns / 1ps

module RegisterFile(
         input   [4:0] A1,   // selects one of 32 registers
         output [31:0] RD1,  // register corresponding to A1
         input   [4:0] A2,   // selects one of 32 registers
         output [31:0] RD2,  // register corresponding to A2
         input   [4:0] A3,   // selects the address for writeback
         input  [31:0] WD3,  // Write-back data, will be written to addess A3
         input         WE3,  // Write-enable for third port WE3=1 write WD3 to A3
         input         CLK   // System clock
    );

    // 宣告 32 個 32-bit 的暫存器陣列 (MIPS 標準架構)
    reg [31:0] rf [31:0];

    // 在 Clock 正緣時且 WE3 為 1 時寫入
    // 加上 (A3 != 5'b00000) 確保 MIPS 的 $0 暫存器永遠保持為 0，不可被覆寫
    always @(posedge CLK) begin
        if (WE3 && (A3 != 5'b00000)) begin
            rf[A3] <= WD3;
        end
    end

    // 非同步讀取。如果讀取的是暫存器 0 (4'b0000)，直接輸出 0
    assign RD1 = (A1 != 5'b00000) ? rf[A1] : 32'h00000000;
    assign RD2 = (A2 != 5'b00000) ? rf[A2] : 32'h00000000;

endmodule
```
跑 Synthesis -> Implementation -> Bitstream -> open hardware manager
