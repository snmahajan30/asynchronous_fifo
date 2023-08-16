module fifo #(parameter data_size = 8,
		parameter addr_size = 4)
	(output [data_size-1:0] rdata,
	output wfull,
	output rempty,
	input [data_size-1:0] wdata,
	input winc, wclk, wrst_n,
	input rinc, rclk, rrst_n);
	
	wire [addr_size-1:0] waddr, raddr;
	wire [addr_size:0] wptr, rptr, wq2_rptr, rq2_wptr;
	
	sync_r2w sync_r2w1 (wq2_rptr, rptr, wclk, wrst_n);
	
	sync_w2r sync_w2r1 (rq2_wptr, wptr, rclk, rrst_n);
	
	fifomem fifomem1 (rdata, wdata, waddr, raddr, winc, wfull, wclk);
	
	rptr_empty rptr_empty1 (rempty, raddr, rptr, rq2_wptr, rinc, rclk, rrst_n);
	
	wptr_full wptr_full1 (wfull, waddr, wptr, wq2_rptr, winc, wclk, wrst_n);
	
endmodule

module fifomem #(parameter data_size = 8,
		parameter addr_size = 4)
		(output [data_size-1:0] rdata,
		input [data_size-1:0] wdata,
		input [addr_size-1:0] waddr, raddr,
		input wclken, wfull, wclk);
	
	//memory
	localparam depth = 1<<addr_size;
	reg [data_size-1:0] mem [0:depth-1];
	
	assign rdata = mem[raddr];
	
	always @(posedge wclk)
	begin
		if (wclken && !wfull) mem[waddr] <= wdata;
	end	
endmodule

module sync_r2w #(parameter addr_size = 4)
		(output reg [addr_size:0] wq2_rptr,
		input [addr_size:0] rptr,
		input wclk, wrst_n);
		
		reg [addr_size:0] wq1_rptr;
		
		always @(posedge wclk or negedge wrst_n)
		begin
			if(! wrst_n) begin
				{wq2_rptr, wq1_rptr} <= 0;
			end
			else begin
				wq1_rptr <= rptr;
				wq2_rptr <= wq1_rptr;
			end
		end
endmodule

module sync_w2r #(parameter addr_size = 4)
		(output reg [addr_size:0] rq2_wptr,
		input [addr_size:0] wptr,
		input rclk, rrst_n);
		
		reg [addr_size:0] rq1_wptr;
		
		always @(posedge rclk or negedge rrst_n)
		begin
			if(! rrst_n) begin
				{rq2_wptr, rq1_wptr} <= 0;
			end
			else begin
				rq1_wptr <= wptr;
				rq2_wptr <= rq1_wptr;
			end
		end
endmodule

module rptr_empty #(parameter addr_size = 4)
		(output reg rempty,
		output [addr_size-1:0] raddr,
		output reg [addr_size:0] rptr,
		input [addr_size:0] rq2_wptr,
		input rinc, rclk, rrst_n);
	
	reg [addr_size:0] rbin;
	wire [addr_size:0] rbinnext, rgraynext;
	
	always @(posedge rclk or negedge rrst_n) begin
		if(! rrst_n) {rbin, rptr} <= 0;
		else {rbin, rptr} <= {rbinnext, rgraynext}; 
	end	
	
	//memory read_address pointer
	assign raddr = rbin[addr_size-1:0];
	
	assign rbinnext = rbin + (rinc & ~rempty);
	assign rgraynext = (rbinnext >> 1) ^ rbinnext;
	
	//FIFO empty condition
	wire rempty_val;
	assign rempty_val = (rgraynext == rq2_wptr);
	
	always @(posedge rclk or negedge rrst_n) begin
		if(! rrst_n) rempty <= 1'b1;
		else rempty <= rempty_val;
	end 
endmodule

module wptr_full #(parameter addr_size = 4)
		(output reg wfull,
		output [addr_size-1:0] waddr,
		output reg [addr_size:0] wptr,
		input [addr_size:0] wq2_rptr,
		input winc, wclk, wrst_n);
	
	reg [addr_size:0] wbin;
	wire [addr_size:0] wbinnext, wgraynext;
	
	always @(posedge wclk or negedge wrst_n) begin
		if(! wrst_n) {wbin, wptr} <= 0;
		else {wbin, wptr} <= {wbinnext, wgraynext}; 
	end	
	
	//memory read_address pointer
	assign waddr = wbin[addr_size-1:0];
	
	assign wbinnext = wbin + (winc & ~wfull);
	assign wgraynext = (wbinnext >> 1) ^ wbinnext;
	
	//FIFO empty condition
	wire wfull_val;
	assign wfull_val = (wgraynext == {~wq2_rptr[addr_size:addr_size-1], wq2_rptr[addr_size-2:0]});
	
	always @(posedge wclk or negedge wrst_n) begin
		if(! wrst_n) wfull <= 1'b0;
		else wfull <= wfull_val;
	end 
endmodule
