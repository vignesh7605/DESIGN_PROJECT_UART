# DESIGN_PROJECT_UART
# AIM:
To implement simulation and synthesis design for uart using cadence,genus and innovus. 

# TOOLS REQUIRED:
Functional Simulation: Incisive Simulator (ncvlog, ncelab, ncsim)
Synthesis: Genus
Floorplannig : innovus
# PROGRAM:
Step 1: Getting Started
README
Synthesis requires three files as follows,
◦ Liberty Files (.lib)
◦ Verilog/VHDL Files (.v or .vhdl or .vhd)
uart.v 
```
`timescale 1ns / 1ps

module uart (
    input reset,
    input txclk,
    input ld_tx_data,
    input [7:0] tx_data,
    input tx_enable,
    output reg tx_out,
    output reg tx_empty,
    input rxclk,
    input uld_rx_data,
    output reg [7:0] rx_data,
    input rx_enable,
    input rx_in,
    output reg rx_empty
);

// Internal Variables
reg [7:0] tx_reg;
reg tx_over_run;
reg [3:0] tx_cnt;

reg [7:0] rx_reg;
reg [3:0] rx_sample_cnt;
reg [3:0] rx_cnt;
reg rx_frame_err;
reg rx_over_run;
reg rx_d1;
reg rx_d2;
reg rx_busy;

// UART RX Logic
always @(posedge rxclk or posedge reset)
begin
    if (reset) begin
        rx_reg <= 0;
        rx_data <= 0;
        rx_sample_cnt <= 0;
        rx_cnt <= 0;
        rx_frame_err <= 0;
        rx_over_run <= 0;
        rx_empty <= 1;
        rx_d1 <= 1;
        rx_d2 <= 1;
        rx_busy <= 0;
    end 
    else begin
        rx_d1 <= rx_in;
        rx_d2 <= rx_d1;

        if (uld_rx_data) begin
            rx_data <= rx_reg;
            rx_empty <= 1;
        end

        if (rx_enable) begin
            if (!rx_busy && !rx_d2) begin
                rx_busy <= 1;
                rx_sample_cnt <= 1;
                rx_cnt <= 0;
            end

            if (rx_busy) begin
                rx_sample_cnt <= rx_sample_cnt + 1;

                if (rx_sample_cnt == 7) begin
                    if ((rx_d2 == 1) && (rx_cnt == 0)) begin 
                        rx_busy <= 0;
                    end 
                    else begin
                        rx_cnt <= rx_cnt + 1;

                        if (rx_cnt > 0 && rx_cnt < 9) begin 
                            rx_reg[rx_cnt - 1] <= rx_d2;
                        end

                        if (rx_cnt == 9) begin
                            rx_busy <= 0;
                            rx_empty <= 0;
                            rx_over_run <= (rx_empty) ? 0 : 1;
                        end
                    end
                end
            end
        end

        if (!rx_enable) begin
            rx_busy <= 0;
        end
    end
end

// UART TX Logic
always @(posedge txclk or posedge reset)
begin
    if (reset) begin
        tx_reg <= 0;
        tx_empty <= 1;
        tx_over_run <= 0;
        tx_out <= 1;
        tx_cnt <= 0;
    end 
    else begin
        if (ld_tx_data) begin
            if (!tx_empty) begin
                tx_over_run <= 1;
            end 
            else begin
                tx_reg <= tx_data;
                tx_empty <= 0;
            end
        end

        if (tx_enable && !tx_empty) begin
            tx_cnt <= tx_cnt + 1;
            if (tx_cnt == 0)
                tx_out <= 0; // Start bit
            else if (tx_cnt > 0 && tx_cnt < 9)
                tx_out <= tx_reg[tx_cnt - 1]; // Data bits
            else if (tx_cnt == 9) begin
                tx_out <= 1; // Stop bit
                tx_cnt <= 0;
                tx_empty <= 1;
            end
        end

        if (!tx_enable) begin
            tx_cnt <= 0;
        end
    end
end

endmodule
```
uart-tb.v
```
`timescale 1 ns / 1 ps
 module uart_tb; 
// Inputs
reg reset; 
reg txclk;
reg ld_tx_data; 
reg [7:0] tx_data; 
reg tx_enable; 
reg rxclk;
reg uld_rx_data; 
reg rx_enable; 
reg rx_in;
// Outputs
wire tx_out;
wire tx_empty;
wire [7:0] rx_data;
wire rx_empty;

uart uut (
.reset(reset),
.txclk(txclk),
.ld_tx_data(ld_tx_data),
.tx_data(tx_data),
.tx_enable(tx_enable),
.tx_out (tx_out),
.tx_empty(tx_empty), 
.rxclk(rxclk),
.uld_rx_data(uld_rx_data),
.rx_data(rx_data),
.rx_enable(rx_enable),
.rx_in(rx_in),
.rx_empty(rx_empty) );
//generate a master clk
reg clk;
//setup clocks
initial clk=0;
always #10 clk = ~clk; // this speed is somewhat arbitrary for the purposes of this sim..
//generate rxclk and txclk so that txclk is 16 times slower than rxclk 
reg [3:0] counter;
initial begin
rxclk=0;
txclk=0;
counter=0;
end
always @(posedge clk) begin
counter<=counter+1;
if (counter == 15) 
txclk <= ~txclk;
rxclk<= ~rxclk;
end
//setup loopback
always@ (tx_out)
 rx_in=tx_out;
initial begin
// Initialize Inputs
reset = 1;
ld_tx_data = 0;
tx_data = 0;
tx_enable = 1;
uld_rx_data = 0;
rx_enable = 1;
rx_in = 1;
// Wait 100 ns for global reset to finish
#500;
reset = 0;
// Send data using tx portion of UART and wait until data is recieved 
tx_data=8'b0111_1111;
#500;
wait (tx_empty==1); //make sure data can be sent
ld_tx_data = 1; //load data to send
wait (tx_empty==0); //wait until data loaded for send
$display("Data loaded for send");
ld_tx_data = 0;
wait (tx_empty==1); //wait for flag of data to finish sending 
$display ("Data sent");
wait (rx_empty==0); //wait for
$display("RX Byte Ready");
uld_rx_data = 1;
wait (rx_empty==1);
$display("RX Byte Unloaded: %b", rx_data);
#100;
$finish;
end
endmodule

```
Add run.tcl File
```
read_libs /cadence/install/FOUNDRY-01/digital/90nm/dig/lib/slow.lib
read_hdl uart.v
elaborate
read_sdc input_constraints.sdc 
syn_generic
report_area
syn_map
report_area
syn_opt
report_area 
report_area > uart_area.txt
report_power > uart_power.txt
write_hdl > uart_netlist.v
write_sdc > uart_output_constraints.sdc
gui_show
```
◦ SDC (Synopsis Design Constraint) File (.sdc)
• In your terminal type “gedit input_constraints.sdc” to create an SDC File if you do not have one.
• The SDC File must contain the following commands;
```
create_clock -name clk -period 2 -waveform {0 1} [get_ports "clk"]
set_clock_transition -rise 0.1 [get_clocks "clk"]
set_clock_transition -fall 0.1 [get_clocks "clk"]
set_clock_uncertainty 0.01 [get_ports "clk"]
set_input_delay -max 0.8 [get_ports "rst"] -clock [get_clocks "clk"]
set_output_delay -max 0.8 [get_ports "count"] -clock [get_clocks "clk"]
```
i→ Creates a Clock named “clk” with Time Period 2ns and On Time from t=0 to t=1.<br />
ii, iii → Sets Clock Rise and Fall time to 100ps.<br />
iv → Sets Clock Uncertainty to 10ps.<br />
v, vi → Sets the maximum limit for I/O port delay to 1ps.<br />
The Liberty files are present in the library path,<br />
• The Available technology nodes are 180nm ,90nm and 45nm.<br />
• In the terminal, initialise the tools with the following commands if a new terminal is being used.<br />
◦ csh<br />
◦ source /cadence/install/cshrc<br />
• The tool used for Synthesis is “Genus”. Hence, type “genus -gui” to open the tool.<br />
• Genus Script file with .tcl file Extension commands are executed one by one to synthesize the netlist.<br />
Step 2 : Creating an SDC File<br />
Step 3 : Performing Synthesis<br />
# SIMULATION WAVEFORM:
![Screenshot 2025-05-27 140924](https://github.com/user-attachments/assets/3b0d7545-b7ac-4b6b-bd5a-b90fa93fb12a)


# SYNTHESIS RTL SCHEMATIC: 
![Screenshot 2025-05-27 140040](https://github.com/user-attachments/assets/c633432a-67bb-4e3b-8547-4884eeeec4cb)

# AREA REPORT:
![Screenshot 2025-05-27 143037](https://github.com/user-attachments/assets/5e660b33-2813-4d3c-9665-43d0342cf42e)

# TIMING REPORT:
![image](https://github.com/user-attachments/assets/7249b990-3b73-4556-a957-7045e89226a8)


# POWER REPORT:
![Screenshot 2025-05-27 162626](https://github.com/user-attachments/assets/b49a9554-3bae-4662-80ae-92a1d0858f06)

# FLOORPLANNING:
![WhatsApp Image 2025-05-27 at 16 16 18_774fd77f](https://github.com/user-attachments/assets/71bf0e22-5802-46a8-8ce3-53f6a456f333)



![WhatsApp Image 2025-05-27 at 16 16 17_1ab58282](https://github.com/user-attachments/assets/8f557345-9b52-451b-986b-66236003f465)

# RESULT:
The functionality of a UART was successfully verified using a test bench and simulated with the nclaunch tool and genus and for innovus created the  floorplanning.








# 
