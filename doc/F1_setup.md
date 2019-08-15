# Creating a Design Checkpoint (DCP) from RTL

Generate verilog using `make verilog`
(Make sure `<TopVerilogModule>Wrapper.sv` is included in verilog sources --- this is needed to interface with the Amazon shell's [0:0] packed vectors for some SDRAM control signals)

Clone aws-fpga repo (or navigate to existing aws-fpga repo)

```bash
(on AMI)
$ git clone https://github.com/aws/aws-fpga.git
$ cd aws-fpga
```

Set up CL directory and environment variables
\***note**: After doing making these manual changes, the freshly-modified templates can be sourced as a `cl_template`
(either by modifying `$HDK_DIR/cl/developer_designs/prepare_new_cl.sh` or by creating a script that copies the files over into a new project)
without having to go through this process in the future.

```bash
(on AMI)
$ source hdk_setup.sh
$ cd hdk/cl/developer_designs
$ mkdir <configname>
$ cd !$
$ export CL_DIR=$(pwd)
$ source $HDK_DIR/cl/developer_designs/prepare_new_cl.sh
```
\***note**: Every login must source `hdk_setup.sh` and set `CL_DIR`

Copy verilog files to `design` directory (`scp` command assumes `aws-fpga` is cloned into user's home directory)

```bash
(on build machine)
$ mkdir vsources
$ cp $BUILD_DIR/verilog/<FullConfigName>/*.{v,sv} vsources
$ cp $BUILD_DIR/memgen/<FullConfigName>.rams.v vsources
$ cp $BUILD_DIR/romgen/<FullConfigName>.roms.v vsources
$ cp $BUILD_DIR/mcs/obj/ip/corePLL/corePLL.v vsources
$ cp $BUILD_DIR/mcs/obj/ip/corePLL/corePLL_clk_wiz.v vsources
$ cp $FPGA_SHELLS_DIR/xilinx/common/vsrc/PowerOnResetFPGAOnly.v vsources
$ scp vsources/*.v user@ami:aws-fpga/hdk/cl/developer_designs/<configname>/design
```

Enter `build/scripts` directory and set up scripts

```bash
(on AMI)
$ cd $CL_DIR/build/scripts
$ mv synth_hello_world.tcl synth_<TopVerilogModule>.tcl
$ ln -sf $HDK_DIR/common/shell_stable/build/scripts/aws_build_dcp_from_cl.sh
```

Modify `encrypt.tcl` to source RTL. Replace the lines that copy source files with the following snippet

```tcl
set VSOURCES [glob $CL_DIR/design/*.{v,sv,vh}]
foreach VSOURCE $VSOURCES {
  file copy -force $VSOURCE $TARGET_DIR
}
```

Replace glob pattern for encryption command to include RTL sources and remove encrypt command for .vhd sources

```tcl
glob -nocomplain -- $TARGET_DIR/*.{v,vh,sv}
```

\***note**: if synthesis and/or place and route is failing, it may be helpful to comment out the line to encrypt the user RTL.
Encryption is only needed for AFI generation

Modify `create_dcp_from_cl.tcl`, assigning the toplevel design name (`F1VU9PShell`) to `CL_Module`:

```bash
set CL_Module <TopVerilogModule>
```

Also add the following line to the command-line arguments section:

```tcl
set VDEFINES [lindex $argv 13]
```

Modify `synth_<TopVerilogModule>.tcl` to source verilog files
Replace the glob pattern for the user RTL files with

```tcl
read_verilog -sv [glob $ENC_SRC_DIR/*.{v,sv}]
```

If using the `sh_ddr` module, a significant amount of rework needs to be done on the IP/verilog includes in the AWS shell/IP sourcing section.
Replace everything from `# ----- End of section replaced by user ------` to `puts "AWS FPGA: Reading AWS constraints";` with the following `read` commands:

```tcl
#Read IP for virtual jtag / ILA/VIO
read_ip [ list \
  $HDK_SHELL_DESIGN_DIR/ip/ila_0/ila_0.xci\
  $HDK_SHELL_DESIGN_DIR/ip/cl_debug_bridge/cl_debug_bridge.xci \
  $HDK_SHELL_DESIGN_DIR/ip/ila_vio_counter/ila_vio_counter.xci \
  $HDK_SHELL_DESIGN_DIR/ip/vio_0/vio_0.xci
]

#Read DDR IP
read_ip [ list \
  $HDK_SHELL_DESIGN_DIR/ip/ddr4_core/ddr4_core.xci
]

#Read AWS Design files
read_verilog -sv [ list \
  $HDK_SHELL_DESIGN_DIR/lib/lib_pipe.sv \
  $HDK_SHELL_DESIGN_DIR/lib/bram_2rw.sv \
  $HDK_SHELL_DESIGN_DIR/lib/flop_fifo.sv \
  $HDK_SHELL_DESIGN_DIR/sh_ddr/synth/sync.v \
  $HDK_SHELL_DESIGN_DIR/sh_ddr/synth/flop_ccf.sv \
  $HDK_SHELL_DESIGN_DIR/sh_ddr/synth/ccf_ctl.v \
  $HDK_SHELL_DESIGN_DIR/sh_ddr/synth/mgt_acc_axl.sv \
  $HDK_SHELL_DESIGN_DIR/sh_ddr/synth/mgt_gen_axl.sv \
  $HDK_SHELL_DESIGN_DIR/sh_ddr/synth/sh_ddr.sv \
  $HDK_SHELL_DESIGN_DIR/interfaces/cl_ports.vh
]

#Read IP for axi register slices
read_ip [ list \
  $HDK_SHELL_DESIGN_DIR/ip/src_register_slice/src_register_slice.xci \
  $HDK_SHELL_DESIGN_DIR/ip/dest_register_slice/dest_register_slice.xci \
  $HDK_SHELL_DESIGN_DIR/ip/axi_clock_converter_0/axi_clock_converter_0.xci \
  $HDK_SHELL_DESIGN_DIR/ip/axi_register_slice/axi_register_slice.xci \
  $HDK_SHELL_DESIGN_DIR/ip/axi_register_slice_light/axi_register_slice_light.xci
]

# Additional IP's that might be needed if using the DDR
read_bd [ list \
  $HDK_SHELL_DESIGN_DIR/ip/cl_axi_interconnect/cl_axi_interconnect.bd
]
```

Finally, in `$CL_DIR/design`

```bash
$ mv cl_template_defines.vh <TopVerilogModule>_defines.vh
$ rm cl_template.sv
```

Modify `<TopVerilogModule>_defines.vh`, replacing the tick-define for `CL_NAME`:

```verilog
`define CL_NAME <TopVerilogModule>
```

Make sure that your PCI vendor/device IDs are set properly in `cl_id_defines.vh`

If you are using an FPGAShell with a PLL (the default `F1VU9PShell` has one), then you'll need to add the following line to `$CL_DIR/build/constraints/cl_pnr_user.xdc`:

```
set_property CLOCK_DEDICATED_ROUTE ANY_CMT_COLUMN [get_nets WRAPPER_INST/SH/kernel_clks_i/clkwiz_sys_clk/inst/CLK_CORE_DRP_I/clk_inst/clk_out1]
```

This is necessary to allow for cascading of the Amazon-provided shell MMCM through a BUFG to the clock generation in our own shell.
Without this line, place and route fails with a "suboptimal placement for an MMCM-BUFGCE-MMCM cascade pair".
Demoting this error to a warning tells the placer to not worry about routing delays for the clock signal.

Additionally, the `.tcl` script `create_dcp_from_cl.tcl` must be modified to prevent errors in bitstream generation.

Please add the following line before synth scripts are sourced in `create_dcp_from_cl.tcl`:

```tcl
set_param hd.clockRoutingWireReduction false

##################################################
### CL XPR OOC Synthesis
##################################################
if {${cl.synth}} {
```