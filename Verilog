`timescale 1ns/1ps


//The master that connects all sub-modules and manages the entire display system.
module top_anpr_display (
    input  wire CLK100MHZ, //100MHz clock from NEXYS A7
    input  wire UART_RX, //Serial data from laptop
    output reg [6:0] SEG, //7-segment cathodes (CA-CG)
    output reg [7:0] AN //8 digit anodes (AN0-AN7)
);

    localparam CLK_FREQ = 100_000_000; 
    localparam BAUD_RATE = 9600;
    //These are passed to uart_rx module for baud rate calculation.

    wire [7:0] uart_byte; //Byte received from uart_rx
    wire uart_valid; //Pulse denotes that new byte is ready.

    wire [3:0] display_code; //Digit from ascii_to_display
    wire char_valid; //Checks validity of digits(0 to 9).
    wire is_reset; //Checks whether it is a reset(R) command.

    /*
    Data Flow:
    uart_rx ? uart_byte & uart_valid
          ?
    ascii_to_display ? display_code & char_valid & is_reset
          ?
    Buffer management ? buffer[5:0]
          ?
    Display multiplexer ? SEG & AN
    */
    
    reg [3:0] buffer [5:0]; //Array of 6 registers(Using 6 Anodes to display) & each digits can be represented in 4 bits.
    reg [2:0] digit_count; //Tracks how many digits received (0-6).
    
    /*
    Index:     [5]  [4]  [3]  [2]  [1]  [0]
    Position: Left ?????????????? Right
    Display:   AN5  AN4  AN3  AN2  AN1  AN0
    
    Example: "207602"
    buffer[5] = 2  ? Leftmost digit
    buffer[4] = 0
    buffer[3] = 7
    buffer[2] = 6
    buffer[1] = 0
    buffer[0] = 2  ? Rightmost digit
    */
    
    integer i;
    initial begin
        for (i = 0; i < 6; i = i + 1)
            buffer[i] = 4'hF;  // Blank, initially no digits received.
            digit_count = 3'd0;
    end

    //Parameter by name
    uart_rx #(.CLK_FREQ(CLK_FREQ), .BAUD_RATE(BAUD_RATE)) uart_inst(
        .clk(CLK100MHZ),
        .rx(UART_RX),
        .data_out(uart_byte),
        .valid(uart_valid)
    );

    // ASCII to 7-segment 
    ascii_to_display a2d(
        .ascii(uart_byte),
        .display_code(display_code),
        .valid(char_valid),
        .is_reset(is_reset)  
    );

    //Buffer management logic with RESET support
    always @(posedge CLK100MHZ) begin
        if (uart_valid) begin
            if (is_reset) begin
                //RESET COMMAND: Clear all displays
                buffer[0] <= 4'hF;
                buffer[1] <= 4'hF;
                buffer[2] <= 4'hF;
                buffer[3] <= 4'hF;
                buffer[4] <= 4'hF;
                buffer[5] <= 4'hF;
                digit_count <= 3'd0;
            end else if (char_valid) begin
                //Valid digit received
                if (digit_count >= 6) begin //Total number of buffers are 8 but it's positioning is from 0 to 7, so 'digit_count' is taken >= 6.
                    //Buffer overflow: eg., for '2076025' o/p will be '5' only & other buffers are cleared.
                    buffer[0] <= display_code;
                    buffer[1] <= 4'hF;
                    buffer[2] <= 4'hF;
                    buffer[3] <= 4'hF;
                    buffer[4] <= 4'hF;
                    buffer[5] <= 4'hF;
                    digit_count <= 3'd1;
                end else begin
                    //Normal operation: shift left and add new digit in right.
                    buffer[5] <= buffer[4]; //Non-blocking shift
                    buffer[4] <= buffer[3];
                    buffer[3] <= buffer[2];
                    buffer[2] <= buffer[1];
                    buffer[1] <= buffer[0];
                    buffer[0] <= display_code;
                    digit_count <= digit_count + 1'b1;
                end
            end
        end
    end

    //Display multiplexing
    reg [2:0] scan_idx; //Which digit/'anode' to display (0-7)
    reg [16:0] clk_div;
    
    initial begin
        scan_idx = 3'd0; //Start with digit 0
        clk_div = 17'd0; //Clock divider counter
    end

    //Refresh per digit is 1kHz, Display refreshes 100 times per second, Refresh frequency: 60Hz to 1KHz recommended
    always @(posedge CLK100MHZ) begin
        if (clk_div >= 17'd99999) begin //Reached target count = 10kHz
            clk_div <= 17'd0; //Reset counter
            scan_idx <= scan_idx + 1'b1; //Move to next digit
        end else begin
            clk_div <= clk_div + 1'b1;
        end
    end

    //Dynamic blanking to only show digits that have been received, if 'digit_count = 3' than only 3 digits will be ON.
    reg [3:0] current_num; //Which pattern to display
    reg digit_enable; //Will decide whether to show the number on the digit display or not.
    
    always @(*) begin
        digit_enable = 1'b1;
        
        case (scan_idx)
            3'd0: begin
                current_num = buffer[0];
                if (digit_count < 1)
                    digit_enable = 1'b0;  //Blank digit if no data given.
            end
            3'd1: begin
                current_num = buffer[1];
                if (digit_count < 2)
                    digit_enable = 1'b0;
            end
            3'd2: begin
                current_num = buffer[2];
                if (digit_count < 3)
                    digit_enable = 1'b0;
            end
            3'd3: begin
                current_num = buffer[3];
                if (digit_count < 4)
                    digit_enable = 1'b0;
            end
            3'd4: begin
                current_num = buffer[4];
                if (digit_count < 5)
                    digit_enable = 1'b0;
            end
            3'd5: begin
                current_num = buffer[5];
                if (digit_count < 6)
                    digit_enable = 1'b0;
            end
            3'd6: begin
                current_num = 4'hF; //Always blank
                digit_enable = 1'b0;
            end
            3'd7: begin
                current_num = 4'hF;
                digit_enable = 1'b0;
            end
            default: current_num = 4'hF;
        endcase
    end

    // Seven-segment decoder
    wire [6:0] seg_pattern;
    seven_seg_decoder ssd (
        .digit(current_num), 
        .seg(seg_pattern)
    );

    // Output registers
    always @(posedge CLK100MHZ) begin
        SEG <= seg_pattern; //Update segments
        AN <= 8'b11111111; // Initially all OFF
        
        if (digit_enable) begin
            AN[scan_idx] <= 1'b0; //Enable selected digit 
        end
    end
endmodule


//Converts ASCII characters from UART into 4-bit display codes (0-9 for 7-segment display).
module ascii_to_display(
    input [7:0] ascii, //ASCII byte from UART
    output reg [3:0] display_code, //Digits to display (0-9).
    output reg valid, //'Valid flag' will check that input is it a digit?
    output reg is_reset //'Reset flag' will check whether reset command is given or not.
);
    always @(*) begin  //Combinational, so outputs update instantly when ascii changes, No clock.
        valid = 1'b1;
        is_reset = 1'b0;
        
        case (ascii)
            //RESET CHARACTER
            8'h52, 8'h72: begin  //'R' or 'r'
                display_code = 4'hF; //Blank space, 'F'=15=1111 denotes that all all segments OFF (Active low).
                valid = 1'b0; //Not a digit
                is_reset = 1'b1; //Reset flag HIGH
            end
            
            // Numbers 0-9
            8'h30: display_code = 4'h0; //ASCII '0' ? 0
            8'h31: display_code = 4'h1;
            8'h32: display_code = 4'h2;
            8'h33: display_code = 4'h3;
            8'h34: display_code = 4'h4;
            8'h35: display_code = 4'h5;
            8'h36: display_code = 4'h6;
            8'h37: display_code = 4'h7;
            8'h38: display_code = 4'h8;
            8'h39: display_code = 4'h9;
            
            default: begin
                display_code = 4'hF;
                valid = 1'b0;
            end
        endcase
    end
endmodule


//Converts 4-bit binary digit (0-9 or blank) into 7-segment LED pattern for FPGA.
module seven_seg_decoder(
    input  [3:0] digit, //Input from ascii_to_display module.
    output reg [6:0] seg //Segment pattern, seg[6:0] = {g, f, e, d, c, b, a}, Active low.
);
    always @(*) begin //No clock
        case (digit)
            4'h0: seg = 7'b1000000; // 0, assigning values to cathode according to numbers to display.
            4'h1: seg = 7'b1111001; // 1
            4'h2: seg = 7'b0100100; // 2
            4'h3: seg = 7'b0110000; // 3
            4'h4: seg = 7'b0011001; // 4
            4'h5: seg = 7'b0010010; // 5
            4'h6: seg = 7'b0000010; // 6
            4'h7: seg = 7'b1111000; // 7
            4'h8: seg = 7'b0000000; // 8
            4'h9: seg = 7'b0010000; // 9
            default: seg = 7'b1111111; // Blank
        endcase
    end
endmodule
/*
      a
     ___
  f |   | b
    |_g_|
  e |   | c
    |___|
      d     · dp (decimal point, not used)
*/


// UART Receiver (Converts serial data from laptop into parallel 8-bit bytes that FPGA can process.)
module uart_rx(
    input  wire clk,
    input  wire rx,
    output reg [7:0] data_out, //One complete ASCII character received; ASCII = 8-bit encoding
    output reg valid //'valid' pulse tells next module that new data is ready.
);
    parameter CLK_FREQ = 100_000_000; //'100 MHz' which is clock frequency of FPGA.
    parameter BAUD_RATE = 9600; //9600 bits/second
    localparam integer BAUD_TICK = CLK_FREQ / BAUD_RATE; //10,416 (Clock cycles needed per bit), Each bit duration = 1/9600 = 104.16 µs

    localparam IDLE  = 2'b00;
    localparam START = 2'b01;
    localparam DATA  = 2'b10;
    localparam STOP  = 2'b11;
    /*
    State Flow:
    IDLE ? START ? DATA (×8 bits) ? STOP ? IDLE
    */
    
    reg [1:0] state;
    reg [15:0] baud_cnt; //Baud counter(baud_cnt): Times each bit duration, need to count up to 10,416.
    reg [3:0] bit_cnt; //Bit counter (bit_cnt): Tracks which bit (0-7), need to count 8 data bits.
    reg [7:0] shift_reg; //Shift register: Builds byte bit-by-bit.
    reg rx_sync1, rx_sync2, rx_sync3;

    initial begin
        state = IDLE;
        baud_cnt = 16'd0;
        bit_cnt = 4'd0;
        shift_reg = 8'd0;
        valid = 1'b0;
        data_out = 8'd0;
        rx_sync1 = 1'b1; //'1' because UART 'Idle' State is HIGH.
        rx_sync2 = 1'b1;
        rx_sync3 = 1'b1;
    end

     //3-Stage synchronizer because signal from laptop & FPGA are not synchronized, so to synchronize them here signal passes through 3 flip-flops & rx_sync3 is synchronized signal.
    always @(posedge clk) begin
        rx_sync1 <= rx;
        rx_sync2 <= rx_sync1;
        rx_sync3 <= rx_sync2;
    end

    always @(posedge clk) begin
        valid <= 1'b0;
        
        case (state)
            IDLE: begin
                baud_cnt <= 16'd0;
                bit_cnt <= 4'd0;
                if (rx_sync3 == 1'b0) begin // Detects START bit (falling edge)
                    state <= START;
                    baud_cnt <= 16'd1;
                end
            end
            
            START: begin
                if (baud_cnt == (BAUD_TICK / 2)) begin // Sample at mid-bit, for accurate result.
                    if (rx_sync3 == 1'b0) begin // Confirm still LOW
                        state <= DATA; //'baud_cnt' counter must start fresh in each state to measure these durations accurately.
                        baud_cnt <= 16'd0;
                        bit_cnt <= 4'd0;
                    end else begin
                        state <= IDLE;
                    end
                end else begin
                    baud_cnt <= baud_cnt + 1'b1;
                end
            end
            
            DATA: begin
                if (baud_cnt == BAUD_TICK - 1) begin // One full bit time
                    baud_cnt <= 16'd0; //RESETS after each bit
                    shift_reg <= {rx_sync3, shift_reg[7:1]}; //Shift right (LSB first)
                    bit_cnt <= bit_cnt + 1'b1; 
                    
                    if (bit_cnt == 4'd7) begin
                        state <= STOP;  // 'baud_cnt' already at 0
                    end
                end else begin
                    baud_cnt <= baud_cnt + 1'b1;
                end
            end
            
            STOP: begin
                if (baud_cnt == BAUD_TICK - 1) begin
                    if (rx_sync3 == 1'b1) begin  //Checks 'STOP' bit is correct.
                        data_out <= shift_reg; // Output received byte
                        valid <= 1'b1; //Signal correct data is ready.
                    end
                    state <= IDLE;
                end else begin
                    baud_cnt <= baud_cnt + 1'b1;
                end
            end
            
            default: state <= IDLE;
        endcase
    end
endmodule
