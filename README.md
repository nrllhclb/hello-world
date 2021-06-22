# hello-world
Let's see what happens, Hello World!
# Run VPR for the 'and' design
#--write_rr_graph example_rr_graph.xml
vpr ${VPR_ARCH_FILE} ${VPR_TESTBENCH_BLIF} --clock_modeling route

# Read OpenFPGA architecture definition
read_openfpga_arch -f ${OPENFPGA_ARCH_FILE}

# Read OpenFPGA simulation settings
read_openfpga_simulation_setting -f ${OPENFPGA_SIM_SETTING_FILE}

# Annotate the OpenFPGA architecture to VPR data base
# to debug use --verbose options
link_openfpga_arch --activity_file ${ACTIVITY_FILE} --sort_gsb_chan_node_in_edges

# Check and correct any naming conflicts in the BLIF netlist
check_netlist_naming_conflict --fix --report ./netlist_renaming.xml

# Apply fix-up to clustering nets based on routing results
pb_pin_fixup --verbose

# Apply fix-up to Look-Up Table truth tables based on packing results
lut_truth_table_fixup

# Build the module graph
#  - Enabled compression on routing architecture modules
#  - Enable pin duplication on grid modules
build_fabric --compress_routing --duplicate_grid_pin #--verbose

# Write the fabric hierarchy of module graph to a file
# This is used by hierarchical PnR flows
write_fabric_hierarchy --file ./fabric_hierarchy.txt

# Repack the netlist to physical pbs
# This must be done before bitstream generator and testbench generation
# Strongly recommend it is done after all the fix-up have been applied
repack #--verbose

# Build the bitstream
#  - Output the fabric-independent bitstream to a file
build_architecture_bitstream --verbose --write_file fabric_independent_bitstream.xml

# Build fabric-dependent bitstream
build_fabric_bitstream --verbose

# Write fabric-dependent bitstream
write_fabric_bitstream --file fabric_bitstream.bit --format plain_text

# Write the Verilog netlist for FPGA fabric
#  - Enable the use of explicit port mapping in Verilog netlist
write_fabric_verilog --file ./SRC --include_timing --print_user_defined_template --default_net_type ${OPENFPGA_DEFAULT_NET_TYPE} --verbose

# Write the Verilog testbench for FPGA fabric
#  - We suggest the use of same output directory as fabric Verilog netlists
#  - Must specify the reference benchmark file if you want to output any testbenches
#  - Enable top-level testbench which is a full verification including programming circuit and core logic of FPGA
#  - Enable pre-configured top-level testbench which is a fast verification skipping programming phase
#  - Simulation ini file is optional and is needed only when you need to interface different HDL simulators using openfpga flow-run scripts
write_full_testbench --file ./SRC --reference_benchmark_file_path ${REFERENCE_VERILOG_TESTBENCH} --include_signal_init --bitstream fabric_bitstream.bit --default_net_type ${OPENFPGA_DEFAULT_NET_TYPE}
write_preconfigured_fabric_wrapper --file ./SRC --support_icarus_simulator --default_net_type ${OPENFPGA_DEFAULT_NET_TYPE}
write_preconfigured_testbench --file ./SRC --reference_benchmark_file_path ${REFERENCE_VERILOG_TESTBENCH} --support_icarus_simulator --default_net_type ${OPENFPGA_DEFAULT_NET_TYPE}                        

# Write the SDC files for PnR backend
#  - Turn on every options here
write_pnr_sdc --file ./SDC

# Write SDC to disable timing for configure ports
write_sdc_disable_timing_configure_ports --file ./SDC/disable_configure_ports.sdc

# Write the SDC to run timing analysis for a mapped FPGA fabric
write_analysis_sdc --file ./SDC_analysis

# Finish and exit OpenFPGA
exit

# Note :
# To run verification at the end of the flow maintain source in ./SRC directory
