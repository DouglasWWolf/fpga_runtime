#!/bin/bash
# -----------------------------------------------------------------------
#  This script converts a Xilinx .bit file to an .mcs file and burns
#  it into the flash memory of an Alveo U55C
#
#  Author: Doug Wolf
# ----------------------------------------------------------------------


# Fetch the name of the file we should burn to flash
filename=$1

# Fetch the name of this script
exe=$(realpath $0)
base_exe=$(basename $exe)


#==============================================================================
# Convert the input .bit file to an .mcs file
#==============================================================================
bit_to_mcs()
{
    # Fetch the name of the input file
    ifile=$1

    # Fetch the name of the output file
    ofile=$2

    # Ensure that the output file doesn't exist
    rm -rf $ofile

    # This is the filename of the script we're going to create
    script=/tmp/make_mcs.tcl

    #   Create the TCL script that will create our MCS file
    echo -n "write_cfgmem -force -format mcs -interface spix4 -size 128 -loadbit" >$script
    echo -n ' "'                                                                 >>$script
    echo -n "up 0x01002000 ${ifile}"                                             >>$script
    echo -n '" '                                                                 >>$script
    echo    "-file ${ofile}"                                                     >>$script
    echo    "exit"                                                               >>$script

    #  Run our script in VIVADO
    $VIVADO -mode tcl -source $script >/tmp/make_mcs.log
}
#==============================================================================


#==============================================================================
# This function burns the .mcs file to U55C flash memory
#==============================================================================
burn_flash()
{
    # This is the name of the TCL script we're going to create
    script="/tmp/flash_u55c.tcl"
    
    # The first line of the script sets our filename
    echo "set filename $1" >$script

    # This is a hack to stop the shell from transating these during the cat<<EOT
    device='$device'
    filename='$filename'

    # Create the rest of the script that will burn our .mcs into U55C flash memory
    cat<<EOT >>$script
    open_hw_manager
    connect_hw_server -allow_non_jtag

    open_hw_target
    current_hw_device  [get_hw_devices xcu280_u55c_0]
    set device [lindex [get_hw_devices xcu280_u55c_0] 0]

    refresh_hw_device -update_hw_probes false $device
    create_hw_cfgmem -hw_device $device [lindex [get_cfgmem_parts {mt25qu01g-spi-x1_x2_x4}] 0]

    set_property PROGRAM.BLANK_CHECK  0 [ get_property PROGRAM.HW_CFGMEM $device ]
    set_property PROGRAM.ERASE        1 [ get_property PROGRAM.HW_CFGMEM $device ]
    set_property PROGRAM.CFG_PROGRAM  1 [ get_property PROGRAM.HW_CFGMEM $device ]
    set_property PROGRAM.VERIFY       1 [ get_property PROGRAM.HW_CFGMEM $device ]
    set_property PROGRAM.CHECKSUM     0 [ get_property PROGRAM.HW_CFGMEM $device ]
    refresh_hw_device $device 

    set_property PROGRAM.ADDRESS_RANGE  {use_file}          [ get_property PROGRAM.HW_CFGMEM $device ]
    set_property PROGRAM.FILES  [ list $filename ]          [ get_property PROGRAM.HW_CFGMEM $device ]
    set_property PROGRAM.PRM_FILE               {}          [ get_property PROGRAM.HW_CFGMEM $device ]
    set_property PROGRAM.UNUSED_PIN_TERMINATION {pull-none} [ get_property PROGRAM.HW_CFGMEM $device ]
    set_property PROGRAM.BLANK_CHECK            0           [ get_property PROGRAM.HW_CFGMEM $device ]
    set_property PROGRAM.ERASE                  1           [ get_property PROGRAM.HW_CFGMEM $device ]
    set_property PROGRAM.CFG_PROGRAM            1           [ get_property PROGRAM.HW_CFGMEM $device ]
    set_property PROGRAM.VERIFY                 1           [ get_property PROGRAM.HW_CFGMEM $device ]
    set_property PROGRAM.CHECKSUM               0           [ get_property PROGRAM.HW_CFGMEM $device ]

    startgroup 
    create_hw_bitstream -hw_device $device [get_property PROGRAM.HW_CFGMEM_BITFILE $device ]
    program_hw_devices             $device 
    refresh_hw_device              $device 
    program_hw_cfgmem -hw_cfgmem [ get_property PROGRAM.HW_CFGMEM $device ]
    endgroup
    exit
EOT

    #  Run our script in VIVADO
    $VIVADO -mode tcl -source $script >/tmp/flash_u55c.log 
}
#==============================================================================


# Did the user supply the filename of a .bit file?
if [ -z $filename ]; then
    echo "$base_exe: no input file specified" 1>&2    
    exit 1
fi

# Make sure our input file exists
if [ ! -f $filename ]; then
    echo "$base_exe: not found - $filename" 1>&2    
    exit 1
fi

# This is the name of the .mcs file that we'll burn into flash
mcs=/tmp/design.mcs

# Create the .mcs file from the .bit file
echo -n "Creating .mcs from $filename..."
bit_to_mcs $filename $mcs
if [ -f $mcs ]; then
    echo " Done!"
else
    echo "FAILED!"
    exit 1
fi

# Now burn our MCS file to the U55C flash memory
echo -n "Burning .mcs to U55C flash..."
burn_flash $mcs
echo " Done!"

# All clear!
exit 0
