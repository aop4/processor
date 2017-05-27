Andrew Puglionesi
aop4@pitt.edu

As far as I know, all the instructions in the JrMIPS programmer's reference manual work correctly when
run in this processor. That means the following instructions, to the best of my knowledge, work correctly: and, nor, addi, addui, add, sub, div, mul, sllv, srlv, bx, bz, bp, bn, lw, sw, li, jal, jr, 
j, put, and halt.
One potential exception is the mul instruction. I've found that when dealing with negative numbers,
it sometimes has strange values for the upper 16 bits of the product. However, this is a direct 
consequence of the built-in multiplier in Logisim: the problem cases I've tested also give the same 
incorrect upper bits when carried out using an isolated multiplier. Since we are supposed to use the 
built-in multiplier, I don't see any reasonable way for me to fix this.

-------Control Signals--------
The control unit interprets the 4 bits of the opcode as well as the subop bit and produces 10 different 
control signals that influence a variety of components in the processor.

The RegWrite signal is set to 1 for instructions that require a register in the register file to be
altered. If it is 0, the register file will not write incoming input to the destination register. If
it is 1, the destination register is altered.

A 3-bit "WriteSource" control signal controls the input to the register file's primary write port, WriteData1.
7 signal values (0-6) for WriteSource are used to specify different write-port inputs. They are set as 
follows:
[signal value]  [RF input]  				[instructions]
-----------------------------------------------------------------------------
000				zero-extended immediate		li
001				ALU							AND, NOR, ADDI, ADDUI, ADD, SUB
010				Divider			 			div
011				Multiplier					mul
100				Shifter						sllv, srlv
101				Data memory					lw
110				PC+1						jal
111				unused						unused

The Mul_Div signal is set to 1 for the mul and div instructions. This causes the remainder from division
and the upper bits of a product to be stored in $r(s+1). Without going into great depth, this is 
accomplished in the register file by "and"ing the mul/div signal with a given register's previous
register's enable signal (e.g., $r0's for $r1). That way, when a mul or div instruction is carried out,
$r(s+1)'s enable bit is set to 1. (For all other instructions, the AND operation guarantees that
the following register isn't written to, since Mul_Div will be 0. The input to $r(s+1) is chosen, via the 
subop bit [see below], from the appropriate output of the divider or multiplier.)

The SetDisplay signal is set to 1 for only the put instruction. It serves as the enable bit for the
register holding a value for the hexadecimal display. Thus, the input ($rs) to this register is displayed
if and only if a put instruction is being processed. Otherwise, the value in the register, and so on the
display, are not changed.

The halt signal is set to 1 for only halt instructions. It serves as the input to the LED (which turns on
when the signal is 1). Also, its negation serves as the enable bit for the PC register.
Its negation is 0 if and only if halt is 1, and so if and only if a halt instruction has been processed.
Thus, PC can be overwritten until a halt instruction has been reached.

The ALUsrc signal determines whether the second operand to the ALU is $rt or an immediate. That
immediate may be sign-extended or zero-extended:
the subop bit is used to decide whether the immediate input to the ALU is sign-extended or zero-extended.
It's zero-extended if and only if subop = 1 (this only has an effect for addi/addui).
If ALUSrc is 1, the immediate is used as the input. This occurs for addi and addui instructions only.

The MemWrite signal serves as the store signal for the data memory. Data memory is altered if and only if
MemWrite = 1, which occurs only during a store instruction.

The branch signal serves as an input to the "comparator" circuit I built. When branch = 1, which is the 
case for the bz, bx, bp, and bn instructions, the logic in the comparator circuit outputs a 1 if the
condition for the instruction is true. This in turn causes a MUX to select the immediate from the 
instruction as the input for PC, completing the branch. Branch = 0 -> circuit outputs a 0.
This requires the comparator circuit to know which branch instruction is being executed, so it has two bits
of the opcode and the subop bit as inputs to distinguish between them. It doesn't need all the bits of the
opcode because the circuit only outputs a 1 if branch is true, which only occurs for branch instructions.
It just needs to distinguish between different branch instructions.
The comparator_control circuit technically distinguishes between the instructions if you look deeply.

The JumpImmediate signal is set to 1 when PC is to be set to an immediate during j and jal instructions.
It has the same effect as the output of the comparator circuit, causing the input to PC to be an immediate
instead of PC+1 if it equals 1.

The JumpReg signal, in contrast, takes effect only during jr instructions. When JumpReg = 1, it causes a
MUX to select the lower 8 bits of $rs to be the input to PC. (Otherwise, either the immediate or PC+1
comes through, depending on previous control signals.)

There are some nameless control signals that haven't been mentioned. The secondary outputs of the divider 
and multiplier are selected based on subop; when it is 1, since a mul instruction *may* have been 
performed, the upper bits of the product are sent to the register file's second write port. Else, the 
remainder of division is.

The subop bit also chooses whether the potential immediate input to the ALU is sign- or zero-extended.
If subop = 1, since an addui instruction *may* be being executed, the zero-extended version is used.
Else, since addi may be the instruction, the sign-extended version is used. (There is no reason, in my
implementation, to use an immediate input to the ALU for other instructions. See the ALUSrc signal
description for more insight on ALU input.)

Within the ALU, there is a control circuit. It uses the two lsb's of the opcode and the subop bit
to determine what an appropriate ALU output is. Of course, whether it's actually used depends on
the WriteSource signal.
When the opcode ends in "00," the result of and or nor is output depending on the value of the lsb
of subop. When the opcode ends in anything else, the output of the adder or subtractor is output:
when the subop is 1 and the lsb of the opcode is 0, the only ALU-requiring operation that could
be performed is sub, so the output of the subtractor is used. Else, the output the adder is used.
That covers add, sub, and, nor, addi and addui, the only instructions the ALU is responsible for.
For other instructions, it doesn't matter what its output is, as it will never be used to write a
value: in that case its output may well be from any of these 6 operations.

The shifter simply uses the subop to determine whether a left or right shift should be performed.