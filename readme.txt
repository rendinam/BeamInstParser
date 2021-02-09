
"Beam Instrumentation Parser" Documentation

================================================================================
The Beam Instrumentation Parser is a C source code generation utility that was 
designed for the purpose of standardizing the generation of communications data 
structures that facilitate data transfers between Cornell Beam Instrumentation 
(CBI) devices (BPMs, BSMs, FLMs) and acquisition applications running on one or 
more control machines.

In order for communications to proceed smoothly between a CBI device and a
control application, the memory structures that are used to store configuration
information and acquired/processed data must match identically in both the DSP 
software and the control application.  The parser ensures this by accepting one 
or more memory structure definition files that closely resemble C-langage 
definitions with some added special characters that the parser itself 
understands, and then generating matched header files and the necessary C 
subroutines to perform data type conversions between the DSP environment and the 
environment within which the control application is running.

From those definition files, a series of several output files are created.  
Those output files can then be included in DSP code within the CBI devices and
any control code that will be communicating with those devices to guarantee
accurate data transfers without the added overhead of manually synchronizing all
the header files and data-format conversion routines every time the need arises
to change the number of data structures present or the elements within them on
either side.

Setup and Output:
------------------
The beam instrumentation parser, hereafter known as BIParser, depends on Python.
It was built using Python 2.3.4.  Newer versions of the interpreter should 
function just as well.

Before runing the parser, you must first create the input definition files
necessary for the CBI device's hardware/software combination.  The base name of
this file should reflect that hardware/software combination (for clarity), and 
have '.def' as its file extension.  All input files should be in the same
directory.  The files should contain typedefs for structs specific to your
hardware/software combination (or for the global structs if re-making the global
auto-generated files), and may contain #define and #include lines, if the 
typedefs within require values that are defined elsewhere.

The parser creates a collection of files containing valid C source code from 
each input file.  

These files are:

o  cbi_<basename>_types_a.h
o  cbi_<basename>_config_a.h
o  cbi_<basename>_tags_a.h
o  cbi_<basename>_init_a.c
o (cbi_dsp_<struct_name>_<basename>_convert_a.c) x (# of derived types)
o  cbi_<basename>_conv_proto_a.h


Where <basename> is the filename of the input definition file minus the 'cbi_'
and the '.def'.

In the case of generating GLOBAL structures, i.e. those that are destined to
reside in the BeamInstSupport library and be shared between ALL of the various 
instrumentation devices that will employ the control and communication scheme 
defined therein, the input definition file should have a basename of "global".

The '_a' indicates that the file was automatically generated.


--The 'types' file will contain the typedef structs from your input file and
  an additional 'DATA' typedef struct that declares structs of those types.  
  This file is needed to build both the DSP executable and control applications.

--The 'config' file initializes the communication structs of type 
  COMM_KEY_CONFIG corresponding to your input typedef structs.
  This file is needed to build both the DSP executable and control applications.

--The 'tags' file contains the enumeration defining the dsp tags corresponding
  to your input typedef structs.  These tags are necessary for proper use of the
  'packet start address table' and 'packet size table' found in the CBI 
  instrumentation units.  See instrumentation unit hardware reference document
  for TigerSHARC-based modules.
  This file is needed to build both the DSP executable and control applications.

--The 'init' file initializes the pointer key to the structs declared in the
  types file.  This code is for control applications only.
  This file is needed to build control applications only.

--One or more 'convert' files that define one function each to convert the 
  elements of each derived typedef  See the $$ section below dealing with the 
  definition of derived types.

--The 'proto' file contains function prototypes for each of the datatype 
  conversion functions as mentioned in the above bullet point.

================================================================================

Input file syntax:
-------------------

#define and #include can be used as normal and will attempt to be evaluated. 
Any lines typed into the file other than !!- or $$-demarcated lines will be
copied into the ...types_a.h file.


'!!' Flags:
----------

The !! should be the first characters in your input file.  Only one set of !!s
is necessary regardless of how many flags you want to set.  This flag sets 
parameters for the entire parsing and code-generation session.

Generally, you should always include the 'files','inc_dir' and 'src_dir' flags
in your input definition file.  Any time !! appears in the file, the same line
can include the following flags:

FLAG           WHAT IT DOES                 POSSIBLE VALUES
________________________________________________________________________________
hardware:  | changes the current         |  user-specified character string
           | hardware type               |
--------------------------------------------------------------------------------
software:  | changes the current         |  user-specified character string
           | software type               |
--------------------------------------------------------------------------------
<x>_segments: | Specifies DSP memory     | Any name may precede '_segments:'
              | segments where structure | Each segment then follows on a new 
              | definitions may take     | line with a comma (,) suffix after
              | place.                   | each item that is not the last item.
-------------------------------------------------------------------------------- 
inc_dir:   | set directory in which to   |  any directory [not implemented]
           | put auto-generated includes |
--------------------------------------------------------------------------------
src_dir:   | set directory in which to   |  any directory [not implemented]
           | put auto-generated source   |
--------------------------------------------------------------------------------

'hardware' is a string reflecting the processor type that will be running the
DSP code that will use these data structures.
    Example     : TSHARC -- Analog Devices TigerSHARC DSP


'software' is a string reflecting the particular data collection and processing 
application.  This may be generalized further to accomodate specific 
subcategories of executables: BPM-Orbit/Phase and BPM-FFT(Bunch-by-bunch tune),
for instance.
    Examples are: BPM -- DSP code functions as a Beam Position Monitor
                  BSM -- DSP code functions as a Beam Size Monitor

Notes on !! flags: 
  --For global types (e.g. DSP_COMMAND) set hardware:GLOBAL and software to 
    the desired type (in this case software:DSP_COMMAND).??
  --By default, the base filename of the definition file (which should always
    end in .def) is used to name the automatically generated files.
  --Keep the name of the software down to 3 letters, to keep the size of 
    filenames and structure names down.
  --'inc_dir' and 'src_dir' default to the current working directory.


================================================================================

'$$' Flags:
----------

Any typedef to be shared for communications purposes between a DSP executable
and a control application and begining with $$ will have a corresponding 
communication structure configuration structure generated in the config file.  
Each $$ flag may contain additional parameters that instruct the parser to 
perform specific actions when processing the struct definition in question.

Some structures are defined to be components of other structures.  It is thus 
likely that direct communication operations on those substructures will not be
needed and their presence only serves as a building block for a more complex
structure that communication operations will be performed upon.  In these 
cases, the substructure would not require a $$ flag before its typedef
stanza, the complex one would require the flag.  The complex structures are
called derived types in this context.

The syntax is:

$$ flag:value
typedef struct { 
  ...
  ...
} <STRUCT_NAME>;


If no $$ is present before the typedef, the struct will just be copied over to
the ...types_a.h file.  No corresponding structure will appear in the 
...config_a.h file.

Available flags and their values can be found in the table below.  If a flag is
not included, the default behavior is used.


  FLAG          WHAT IT SETS            POSSIBLE VALUES       DEFAULT VALUE
________________________________________________________________________________
check_ptr: |Name of a custom          |                     |
           |check/copy function       |                     |      
           |pointer.  This ptr would  |any character string |       NULL
   [X]     |reference a function      |                     |
           |responsible for doing data|                     | 
           |integrity and consistency |                     |
           |tests on the structure's  |                     |
           |contents.                 |                     |
--------------------------------------------------------------------------------
io_ptr:    |value of the stream       |any character string |   cbi_struct_io
           |I/O pointer.  This ptr    |                     |
           |would reference a func.   |                     | 
   [X]     |responsible for printing  |                     |
           |the given structure's     |                     |
           |contents in a sensible way|                     |
           |to the console.           |                     |
--------------------------------------------------------------------------------
exe:       |DSP executable type for   |any character string |hardware_software
           |which struct is to be made|                     |(current values)
   [X]     |available.                |                     |
           | (See below for more      |                     | 
           |  information on EXE      |                     |
           |  types.)                 |                     |
--------------------------------------------------------------------------------
multi:     |number of instances in    |     integer         |        1
           |memory to define          |                     |
           | A uniqe name will be     |                     |
           | assigned to each struct. |                     |
           | by appending an integer  |                     |
           | index to the name        |                     |
           | starting at 0.           |                     |
           | (e.g. the 4 PROC_BUFS)   |                     |
--------------------------------------------------------------------------------
rec_len:[X]|value of record length    |      var/fix        |       fix
--------------------------------------------------------------------------------
protection:|The protection value      |    read/write/rw    |       rw
           |determines what priveleges|                     |
           |the control application   |                     |
           |has to access the contents|                     |
           |of the populated data     |                     |
           |structure in memory.      |                     |
--------------------------------------------------------------------------------
mp_space:  |This instructs the parser |                     |
           |to either use a DSP       |                     |
           |'multiprocessor space'    |                     |
           |memory offset when loading|                     |
           |structure addresses into  |    YES  /  NO       |       YES
           |the digital board FPGA's  |                     |
           |packet address table.     |                     |
           |                          |                     |
           |All structures that live  |                     |
           |in DSP internal memory or |                     |
           |in memory on something    |                     |
           |like the AFE4 front-end   |                     |
           |board need mp_space       |                     |
           |offsets.  Those that live |                     |
           |in digital board SRAM do  |                     |
           |NOT.                      |                     | 
--------------------------------------------------------------------------------
array:     |array size in             |integer              |        1
    [X]    |DSP_DATA struct           | (need not evaluate) | 
--------------------------------------------------------------------------------
clones     |clones in DSP_DATA struct |character strings    |
           |(e.g.delay_table,         |separated by '/' or  |     no clones           
    [X]    | inj_delay_table and      |an integer telling   | 
           | std_delay_table)         |how many clones      |
--------------------------------------------------------------------------------
num_pkts   |number of packets         |integer              | calculated value
    [X]    |required                  | (need not evaluate) | 
--------------------------------------------------------------------------------
(X = not yet implemented)

Notes: -- A representative set of DSP executable types is:
           1) GENERIC_EXE
           2) FFT_EXE

          An enumeration definining all possible EXE types is kept in a central
          header file as part of the BeamInstSupport library area.  All code
          depends on making the distinction between EXE types must include that
          file.

          The default exe type for a _global_ struct is GENERIC_EXE
      **  --> The BIParser currently hardcodes this value to 1 in all code
              generation.
         
         o Parsing a GLOBAL definitions file will produce data structure
           definitions that are common to ALL beam instrumentation software.
           This typically includes basic and universal functionality that is
           shared by a control application and any of the DSP-based
           instrumentation units such as a command/handshaking structure, DSP 
           status structure, and a DSP debug structure.


================================================================================

BIParser Command Syntax:
-------------------------

Move to the directory in which your input definition files are stored. 

Type: 
BIParser [-c<dir>] [-h<dir>] [.def-filename]

Options:
-c  -- Place auto-generated .c files in this directory.
-h  -- Place auto-generated .h files in this directory.

================================================================================

Simple example of a correctly constructed global parser input file:
--------------------------------------------------------------------

!! hardware:GLOBAL inc_dir:../comm_include src_dir:../comm_code

#include "cbi_dsp_params.h"

   $$
   typedef struct {
         int cmd;
         int cmd_status;
         int error[MX_ERROR_WORDS];
         int handshake;
   } CBI_DSP_CMD;

   $$
   typedef struct {
         int state;
         int status;
         int num_levels;
         int trace[MX_TRACE_LEVELS];
   } CBI_DSP_STAT;

   $$
   typedef struct {
         int write_ptr;
         int debug[MX_DEBUG_WORDS];
         int routine[MX_DEBUG_WORDS];
   } CBI_DSP_DEBUG;

