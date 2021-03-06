# AXI_DMA_FIFO
Transfer data from DDR memory to AXI4-Stream Data FIFO and back through AXI DMA
***
## Getting Start (Experiment Setup)
+ Windows 7/8/8.1/10 64-bit
+ Xilinx Vivado 2016/17/18.X (must include SDK)
+ [Board definition files](http://microzed.org/support/documentation/1519) for MicroZed 70X0
+ USB to micro USB cable x 1 
+ USB to mini USB cable x 1 (for JTAG)
+ RJ45 ethernet cable
+ [Putty.exe](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html)
## System Overview
本實驗將透過[AXI DMA (Direct Memory Access)](https://www.xilinx.com/support/documentation/ip_documentation/axi_dma/v7_1/pg021_axi_dma.pdf) 負責I/O Device與DDRX Memory之間的資料傳輸，其過程完全不需CPU(中央處理器)參與，CPU只需將傳送任務 (transfer task)和敘述位址(descriptors addresses)發送給DMA代理，本身就有更多時間將運算單元與暫存器用在對其他程序的執行上。就PS端而言，典型的的傳送作業方式有兩種：
1. SG_Intr Mode 
   + Definition: Transfer packets in Interrupt Mode when AXIDMA core is configured in Scatter Gather Mode.
   + 由CPU完成的作業步驟包括：初始化DMA並設立傳輸，啟用DMA的中斷系統，初始化旗標，發送封包，待發生中斷時執行程式碼迴圈(ex: 再次指定傳輸，旗標值改變)，全部傳送完畢後跳開迴圈並檢查DMA傳送結果，指定發送下一次封包或關閉DMA的中斷系統。
2. SG_Poll Mode
   + Definition: Transfer packets in Polling Mode when AXIDMA core is configured in Scatter Gather Mode.
   + 由CPU完成的作業步驟包括：初始化DMA並設立傳輸，發送封包，檢查DMA傳送結果(可能重複數次直到傳送結束)，指定發送下一次封包。

Block diagram (Compatible with the above two Modes)：

<img src="https://github.com/absolutezero2730/AXI_DMA_FIFO/blob/master/picture/Design%20Overview.png" width="80%" height="80%">

For high speed computing design, also for reducing the payload on CPU, the DMA may be the most efficient way of doing so. Why DMA? If we had data coming in from a very fast ASIC readout chip, or a multi-channel ADC device, and we need to store it very quickly through the FPGA to the DDR memory. We can't just rely on the processor to transfer data. This may overkill the intelligent performances of CPU and waste too much of its registers. 

In this tutorial we are using the DMA interface to build a simple data transfer through PL to the DDR memory. The AXI DMA and AXI Data FIFO are connected through the AXIS_MM2S and AXIS_S2MM buses. These two AXIS buses mainly source and sink data stream without address. The processor will communicate through the AXI-lite bus to the DMA for setting up, initiating and monitoring. The AXI_MM2S and AXI_S2MM are memory-mapped AXI buses that connect to the memory controller. 

When this experiment is complete, you will be able to: 
1. Use [AXI4-Stream Data FIFO](https://www.xilinx.com/support/documentation/ip_documentation/axis_infrastructure_ip_suite/v1_1/pg085-axi4stream-infrastructure.pdf) and [AXI DMA](https://www.xilinx.com/support/documentation/ip_documentation/axi_dma/v7_1/pg021_axi_dma.pdf)
2. Use the xaxidma driver on the Xilinx AXI DMA to transfer packets in polling mode
3. Use the xaxidma driver on the Xilinx AXI DMA to transfer packets in interrupt mode
4. Get DMA status through UART
## Lab 1: Create a New Zynq Project
1. Launch Vivado.
2. Select <b>File</b> -> <b>New Project</b> or click on <b>Create New Project</b> under Quick Start. 
3. Click the browse icon. Browse to set the Project location to your desired project location and <b>click Start</b>.
4. Set the project name to <b>Microzed_7020_AXI_DMA_test</b>. Also verify the <b>Create project subdirectory</b> check box is selected. Click <b>Next ></b>.
5. The project will be RTL based. Leave the radio button for RTL Project selected. Since this is a brand new project, check the box for <b>Do not specify sources at this time</b>. Click <b>Next ></b>.

<img src="https://github.com/absolutezero2730/AXI_DMA_FIFO/blob/master/picture/catch01.PNG" width="80%" height="80%">

6. In the Select area, select <b>Boards</b>.

<img src="https://github.com/absolutezero2730/AXI_DMA_FIFO/blob/master/picture/catch02.PNG" width="80%" height="80%">

7. Single-click the <b>MicroZed Board</b> that matches your configuration. Click <b>Next ></b>.
8. A project summary is displayed. Click <b>Finish</b>. The Vivado cockpit is now displayed. 

## Lab 2: Create and Edit a Block Design
1. The recommended way to add an embedded processor is through the Block design method via IP Integrator. Select <b>Create Block Design</b>. 
2. Give the Block Design a name (default: design_1). Click <b>OK</b>.
3. In the Diagram window, click the <b>Add IP</b> text icon. 
4. Find the [<b>ZYNQ7 Processing System</b>](https://www.xilinx.com/support/documentation/ip_documentation/processing_system7/v5_5/pg082-processing-system7.pdf) IP. Either double click this or drag and drop to the Diagram window. 

<img src="https://github.com/absolutezero2730/AXI_DMA_FIFO/blob/master/picture/catch03.PNG" width="80%" height="80%">

5. Similar to the Add IP prompt in the previous step, notice now that the Designer Assistance has provided the hint to Run Block Automation. Click the <b>Run Block Automation</b> link at the top of the window. (Notice the block automation wizard has identified two sources of I/O that need to be made external. One is obvious, the <b>DDR</b> interface. The other is labeled <b>FIXED_IO</b>. FIXED_IO is basically the MIO pin connections. They are labeled FIXED_IO because you cannot change their assignments in this window. The Apply Board Preset checkbox applies the Preset TCL that was included as part of the board definition archive. Leave this checked. The Cross Trigger options may be left Disabled. Click <b>OK</b> to connect these external signals.)
6. You will now see the Zynq block with external I/O.

<p align="center">
<img src="https://github.com/absolutezero2730/AXI_DMA_FIFO/blob/master/picture/catch04.PNG" width="60%" height="60%">
</p>

7. Double click on the <b>Zynq PS</b> and have a look on the Block Diagram. In the Application Processor Unit, find the <b>TTC</b> block and simply click it. 

<img src="https://github.com/absolutezero2730/AXI_DMA_FIFO/blob/master/picture/catch05.PNG" width="80%" height="80%">

8. MIO Configuration -> Application Processor Unit -> check the box for <b>Timer 0</b>. Disable it. 

<img src="https://github.com/absolutezero2730/AXI_DMA_FIFO/blob/master/picture/catch06.PNG" width="80%" height="80%">

9. Next have a look on the <b>Clock Configuration</b> section. Click on the <b>PL Fabric Clocks</b>. We can see that we have one of the fabric clocks that has been configured as 100 MHz. We will use that clock for all of logic IPs. 
10. The AXI DMA needs access to the DDR memory controller and a configuration interface to exchange information between itself and the CPU. Thus the processor will configure the AXI DMA through an AXI-lite interface and send back the AXI DMA an access for the DDR mapping. Refer to the Zynq Block Design again. For the DMA configuration, the processor requires a general purpose AXI master port so that it can configure the DMA. Navigate to the Programmable Logic area. Find the <b>32b GP AXI Master Ports</b> and click it. Here we can see a general purpose master AXI interface. Now check the box for <b>M AXI GP0 interface</b> and enable it. 

<img src="https://github.com/absolutezero2730/AXI_DMA_FIFO/blob/master/picture/catch07.PNG" width="80%" height="80%">

11. The other interface we need is to access the DDR controller. Go back to the Zynq Block Design section. We need to allow the DMA to read and write from the DDR. We can see from the block diagram we need to enable the <b>High Performance AXI 32b/64b Slave Ports</b>. So we click on that. Choose one of those. PS-PL Configuration -> HP Slave AXI Interface -> <b>S AXI HP0 interface</b>.

<img src="https://github.com/absolutezero2730/AXI_DMA_FIFO/blob/master/picture/catch08.PNG" width="80%" height="80%">

12. One more thing that we need to configure is the interrupts. In the SG_Intr Mode, the CPU will receive interrupts from the DMA. We need to enable the Fabric Interrupts. Refer to Interrupts -> Fabric Interrupts -> PL-PS Interrupt Ports -> <b>IRQ_F2P[15:0]</b>. Check the box and select. 

<img src="https://github.com/absolutezero2730/AXI_DMA_FIFO/blob/master/picture/catch09.PNG" width="80%" height="80%">

13. Click <b>OK</b>. Now the ZYNQ7 Processing System is properly configured. 

<p align="center">
<img src="https://github.com/absolutezero2730/AXI_DMA_FIFO/blob/master/picture/catch10.PNG" width="60%" height="60%">
</p>

14. The 100 MHz clock should feed the two AXI interfaces. Thus we wire the <b>fabric clock (FCLK_CLK0)</b> to the <b>general purpose AXI interface (M_AXI_GP0_ACLK)</b> and the <b>high performance slave interface (S_AXI_HP0_ACLK)</b>. 

<p align="center">
<img src="https://github.com/absolutezero2730/AXI_DMA_FIFO/blob/master/picture/catch11.PNG" width="65%" height="65%">
</p>

15. Click <b>+ (Add IP)</b> to add the [<b>AXI Direct Memory Access</b>](https://www.xilinx.com/support/documentation/ip_documentation/axi_dma/v7_1/pg021_axi_dma.pdf). 

<img src="https://github.com/absolutezero2730/AXI_DMA_FIFO/blob/master/picture/catch12.PNG" width="80%" height="80%">

16. We don't actually need the <b>AXI streaming status (S_AXIS_STS)</b> and the <b>AXI streaming control ports (M_AXIS_CNTRL)</b>. Double click on the <b>AXI Direct Memory Access</b> to exclude these two ports, which are used in ethernet applications. Check the box for <b>Enable Control/Status Stream</b>. Disable it. 

<img src="https://github.com/absolutezero2730/AXI_DMA_FIFO/blob/master/picture/catch13.PNG" width="80%" height="80%">

17. Click on <b>Run Connection Automation</b>. Enable the <b>All Automation</b> box. If we click it one by one, first for the <b>S_AXI_LITE</b>, we can see it wants to connect the AXI lite interface to the processor’s general purpose AXI master ports with a bridge IP named the [<b>AXI Interconnect</b>](https://www.xilinx.com/support/documentation/ip_documentation/axi_interconnect/v2_1/pg059-axi-interconnect.pdf). Second for the <b>S_AXI_HP0</b>, the high performance slave AXI interface of the PS, vivado wants to connected it to the DMA's <b>scatter-gather AXI master interface (M_AXI_SG)</b> with an [<b>AXI SmartConnect</b>](https://www.xilinx.com/support/documentation/ip_documentation/smartconnect/v1_0/pg247-smartconnect.pdf) IP. 

18. Click <b>OK</b> to run connection automation. Make sure that the general propose AXI master ports are well connected through the <b>AXI Interconnect</b> to the AXI lite interface of the DMA. This is for the DMA configuration. And also the scatter-gather AXI master interface is connected through the <b>AXI SmartConnect</b> to the high performance slave AXI interface, which is basically an access to the DDR memory. 

<p align="center">
<img src="https://github.com/absolutezero2730/AXI_DMA_FIFO/blob/master/picture/catch14.PNG" width="100%" height="100%">
</p>

19. Click on <b>Run Connection Automation</b> again to wire the last two AXI master interfaces. Enable the <b>All Automation</b> box. The AXI memory-mapped to stream Master (MM2S Memory Map Read Interface Signals), and the AXI stream to memory-mapped master (S2MM Memory Map Write Interface Signals) will thus be integrated through the <b>AXI SmartConnect</b> to the high performance slave AXI interface with the former mentioned scatter-gather AXI master interface. Click <b>OK</b> to run connections. 

<p align="center">
<img src="https://github.com/absolutezero2730/AXI_DMA_FIFO/blob/master/picture/catch15.PNG" width="100%" height="100%">
</p>

20. Click <b>+ (Add IP)</b> to add the [<b>AXI4-Stream Data FIFO</b>](https://www.xilinx.com/support/documentation/ip_documentation/axis_infrastructure_ip_suite/v1_1/pg085-axi4stream-infrastructure.pdf). Connect the <b>AXI streaming master interface (M_AXIS)</b> to the <b>AXI streaming slave interface (S_AXIS_S2MM)</b> of the DMA. Then connect the <b>AXI streaming master interface (M_AXIS_MM2S)</b> of the DMA to the <b>AXI streaming slave interface (S_AXIS)</b> of the FIFO. Then manually connect the <b>clock (s_axis_aclk)</b> and <b>reset (s_axis_aresetn)</b> to the fabric clock (FCLK_CLK0) and the <b>peripheral_aresetn[0:0]</b> of the [<b>Processor System Reset</b>](https://www.xilinx.com/support/documentation/ip_documentation/proc_sys_reset/v5_0/pg164-proc-sys-reset.pdf) IP. 

<p align="center">
<img src="https://github.com/absolutezero2730/AXI_DMA_FIFO/blob/master/picture/catch16.PNG" width="100%" height="100%">
</p>

21. Click <b>Optimize Routing</b> to make these wires look better. Last, the two interrupts of the DMA need to be connected to the processor's <b>IRQ_F2P</b>. The way to do that is to use a [<b>Concat</b>](https://www.xilinx.com/support/documentation/ip_documentation/xilinx_com_ip_xlconcat/v2_1/pb041-xilinx-com-ip-xlconcat.pdf) IP. Simply click <b>+ (Add IP)</b> to add it into our Diagram. Manually connect the <b>mm2s_introut</b> and <b>s2mm_introut</b> to the <b>In0[0:0]</b> and <b>In1[0:0]</b>. Connect the <b>dout[1:0]</b> to the <b>IRQ_F2P</b>. All things are done now.  

<p align="center">
<img src="https://github.com/absolutezero2730/AXI_DMA_FIFO/blob/master/picture/catch17.PNG" width="100%" height="100%">
</p>

22. Click the <b>Save</b> button to save the block design and click the <b>Validate Design</b> button to make sure that we haven't make any mistake or mis-connect to these IP cores. We can safely ignore the warning messages (probably due to the some versions of board definition file...) after validating. Click the <b>Validate Design</b> button to make sure twice. Click <b>OK</b> to close the message window. 

23. Save the design again. Generate the HDL Wrapper by clicking Source -> design_1 (design_1.bd) -> right click -> Create HDL Wrapper. <b>Let Vivado manage wrapper and auto-update</b>. Click <b>OK</b>. 

<img src="https://github.com/absolutezero2730/AXI_DMA_FIFO/blob/master/picture/catch18.png" width="50%" height="50%">

24. Navigate to the bottom of Flow Navigator. Click and run the <b>Generate Bitstream</b>. This may take a while. 
25. In the Vivado tool, select <b>File -> Export -> Export Hardware</b>. Check the box to Include bitstream. Click <b>OK</b> to export hardware. 


## Lab 3: Export Hardware Platform to SDK
1. Now to launch SDK, select <b>File -> Launch SDK</b>. Click <b>OK</b>.
2. Now being in SDK we want to create an application. Start by clicking <b>File -> New -> Application Project</b>. 

<img src="https://github.com/absolutezero2730/AXI_DMA_FIFO/blob/master/picture/catch19.png" width="50%" height="50%">

3. Type a name for the application. Click <b>Next</b>. 

<img src="https://github.com/absolutezero2730/AXI_DMA_FIFO/blob/master/picture/catch20.PNG" width="50%" height="50%">

4. On the templates page, select the <b>Empty Application</b> template and click <b>Finish</b>. 

<img src="https://github.com/absolutezero2730/AXI_DMA_FIFO/blob/master/picture/catch21.PNG" width="50%" height="50%">

5. From the Project Explorer, open the dma_test/src folder. Then open a File Manager (開啟檔案總管). A DMA Poll_Mode example code can be found here: <b>C:\Xilinx\SDK\2018.3\data\embeddedsw\XilinxProcessorIPLib\drivers\axidma_v9_3\examples\xaxidma_example_sg_poll.c</b>. Drag the file from the file manager to the dma/src folder directly. (直接將檔案橫拉進SDK Project Explorer中的dma_test/src裡) On the appeared window, select <b>Copy files</b> and click <b>OK</b>. The file has now been copied to the src folder. 
6. Set the Boot Mode jumpers to Cascaded JTAG Mode (MicroZed: MIO[5:2] = GND. Set JP3, JP2, and JP1 to positions 1-2). Connect the UART (micro-USB) to your PC. Connect a Platform Cable or Digilent Programming cable from your PC to the 2x7 JTAG socket. 
7. Go back to the xilinx SDK. select <b>Xilinx -> Program FPGA</b>. Make sure that the connection is good. Then click <b>Program</b>. 
8. After the FPGA programing is complete. <b>We launch Putty.exe</b>. On the session template, set the connection type to serial. Then refer to the serial template, select a serial line (ex: COM3). Note: you can open your device manager to confirm this. Set the Speed (baud rate) to 115200, Data bits 8 and stop bits 1, Parity None. Click <b>Open</b>.

<img src="https://github.com/absolutezero2730/AXI_DMA_FIFO/blob/master/picture/catch22.PNG" width="50%" height="50%">


## Lab 4: Run DMA Poll Mode Application
1. To run the test, click on dma_test application in the Project Explorer. Then click on run configurations (<b>Run -> Run Configuration</b>) from the menu bar. 
2. Double click on the <b>Xilinx C/C++ Application (System Debugger)</b>. And we can then click <b>Run</b>. It runs the software application on the processor. 

<img src="https://github.com/absolutezero2730/AXI_DMA_FIFO/blob/master/picture/catch23.png" width="80%" height="80%">

3. After the application is executed, we can look out for the results from the Putty window. 
```
--- Entering main() ---
Successfully ran AXI DMA SG Polling Example
--- Exiting main() ---
```

## Lab 5: Run DMA Interrupt Mode Application
1. Left the Putty terminal being opened, now click <b>File -> New -> Application Project</b>. 
2. Type a name for the application (dma_intr_test). Click <b>Next</b>. 
3. On the templates page, select the <b>Empty Application</b> template and click <b>Finish</b>. 
4. From the Project Explorer, open the dma_intr_test/src folder. Then open a File Manager. A DMA Interrupt_Mode example code can be found here: <b>C:\Xilinx\SDK\2018.3\data\embeddedsw\XilinxProcessorIPLib\drivers\axidma_v9_3\examples\xaxidma_example_sg_intr.c</b>. Drag the file from the file manager to the dma/src folder directly. Then, on the appeared window, again, select <b>Copy files</b> and click <b>OK</b>. The file has now been copied to the src folder. 
5. Right click the dma_intr_test application in the Project Explorer and refresh it. 
6. Click on dma_intr_test application in the Project Explorer. Then click on run configurations (<b>Run -> Run Configuration</b>) from the menu bar again. 
7. Double click on the <b>Xilinx C/C++ Application (System Debugger)</b>. And we can then click <b>Run</b>. It runs the software application on the processor.
8. After the application is executed, we can look out for the results from the Putty window. 
```
--- Entering main() ---
Successfully ran AXI DMA SG interrupt Example
--- Exiting main() ---
```
