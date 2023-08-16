`include "asynch_fifo.v"

module fifo_tb; 

reg [7:0] wdata;
reg wclk,rclk,winc,rinc,wrst_n,rrst_n;
wire wfull,rempty;

wire [7:0] rdata;

fifo fifo1_t (rdata,wfull,rempty,wdata,winc, wclk, wrst_n,rinc, rclk, rrst_n);

initial
	begin
		$dumpfile("test.vcd");
		$dumpvars(0, fifo_tb);
		wclk = 0;
		rclk = 0;
		winc = 0;
		rinc = 0;
		wrst_n = 1;
		rrst_n = 1;
		#5 wrst_n = 0; rrst_n = 0;
		#5 wrst_n = 1; rrst_n = 1;
		
	
		#5 winc <= 1;
		#10 rinc <= 1; 
	 
		#5 wdata <= 1;
		#40 wdata <= 2;
		#40 wdata <= 3;
		#40 wdata <= 4;
		#40 wdata <= 5;
		
		#40 winc <= 0;
		#10000 rinc <= 0; 
		
		#100000 $finish;
		
	end
	

always #20 wclk = ~wclk;
always #100 rclk = ~rclk;

endmodule
