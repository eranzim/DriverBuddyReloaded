# Driver Buddy Reloaded Quickstart

## Table of Contents

- [Driver Buddy Reloaded Quickstart](#driver-buddy-reloaded-quickstart)
  - [Table of Contents](#table-of-contents)
  - [Installation](#installation)
  - [Quick Usage](#quick-usage)
    - [Advanced Usage](#advanced-usage)
  - [About Driver Buddy Reloaded](#about-driver-buddy-reloaded)
    - [Finding DispatchDeviceControl](#finding-dispatchdevicecontrol)
    - [Labelling WDM and WDF Structures](#labelling-wdm-and-wdf-structures)
    - [Finding and Decoding IOCTL Codes](#finding-and-decoding-ioctl-codes)
    - [Flagging Functions](#flagging-functions)
    - [Finding DeviceName](#finding-devicename)
    - [Dumping Pooltags](#dumping-pooltags)
  - [Known Caveats and Limitations](#known-caveats-and-limitations)
  - [Credits and Acknowledgements](#credits-and-acknowledgements)

## Installation

Copy `DriverBuddyReloaded` folder and `DriverBuddyReloaded.py` file into the IDA plugins folder (e.g. `C:\Program Files\IDA Pro 7.6\plugins\`) or wherever you have installed IDA.

If you are using Python v. 3.x run the `idapyswitch.exe` binary, present in IDA's folder, from an admin command prompt.

## Quick Usage

To use the auto-analysis feature:

1. Start IDA and load a Windows kernel driver.
2. Go to `Edit -> Plugins -> Driver Buddy Reloaded` or press `CTRL+ALT+A` to start the auto-analysis.
3. Check the "Output" window for the analysis results.
4. A `DriverName.sys-2021-12-10-TIME_STAMP-DriverBuddyReloaded_autoanalysis.txt` file, containing the analysis results,
   will be written under IDA's DB directory.

To decode an IOCTL:

1. Place the mouse cursor on the line containing a suspected IOCTL code.
2. Right-click and select `Driver Buddy Reloaded -> Decode IOCTL`; alternatively press the `CTRL+ALT+D` shortcut.

To decode ALL IOCTLs within a function:

1. Place the mouse cursor on the first instruction of the function you believe to be the IOCTL dispatcher (
   `DispatchDeviceControl`, `DispatchInternalDeviceControl`, `Possible_DispatchDeviceControl_#`)
2. Right-click and select `Driver Buddy Reloaded -> Decode ALL IOCTLs in Function`; alternatively press the `CTRL+ALT+F`
   shortcut.
3. A `DriverName.sys-2021-12-10-TIME_STAMP-IOCTLs.txt`/`DriverName.sys-2021-12-10-TIME_STAMP-IOCTLs.txt_dumb.txt` file,
   containing all the decoded IOCTLs up to that moment, will be written under IDA's DB directory.

### Advanced Usage

- The [vulnerable_function_lists](DriverBuddyReloaded/vulnerable_functions_lists) directory contains a lists of potentially
  dangerous/problematic functions, Windows APIs and opcodes; a brief description on why a specific function/API has been listed is
  provided. You can edit the `custom` list including driver's specific functions.
  
  **Note**: `winapi_function_prefixes` will partial match to start of function name (e.g. `Zw` will match `ZwClose`, `ZwCommitComplete` and so on) while `winapi_functions` will perform exact matches only.
- In [find_opcodes.py](DriverBuddyReloaded/find_opcodes.py), the `find_opcode_data` option will prevent Driver Buddy
  Reloaded to find opcodes in data sections. Switching it to `True` will print something along
  this [line](https://github.com/VoidSec/DriverBuddyReloaded/issues/11):
  `Found jnz     short loc_15862 in sub_15820 at 0x00015852`
  Usually, going at the showed address and re-defining the selection as code will bring the searched opcode back.
  
  **Watch out**: switching it to `True`, will generates more false positives!

## About Driver Buddy Reloaded

**Driver Buddy Reloaded** is an IDA Pro Python plugin that helps automate some tedious Windows Kernel Drivers reverse
engineering tasks. It has a number of handy features, such as:

* Identifying the type of the driver
* Locating `DispatchDeviceControl` / `DispatchInternalDeviceControl` functions
* Populating common structures for `WDF` and `WDM` drivers
    * Attempts to identify and label structures like the `IRP` and `IO_STACK_LOCATION`
    * Label calls to `WDF` functions that would normally be unlabeled
* Finding and decoding IOCTL codes
* Flagging functions prone to misuse
* Finding potential `DeviceName`
* Dumping `Pooltags`

![](/screenshots/auto-analysis.png)

### Finding DispatchDeviceControl

The tool can automatically locate and identify the `DispatchDeviceControl` routine. This function is used to route all
incoming `DeviceIoControl` codes to the specific driver function associated with that code. Automatically identifying
this function makes finding the valid `DeviceIoControl` codes for each driver much quicker. Additionally, when
investigating possible vulnerabilities in a driver due to a crash, knowing the location of this function helps narrow
the focus to the specific function call associated with the crashing `DeviceIoControl` code.

When the analysis is successful some subs will be renamed as follow:

- `DriverEntry`: the original first driver-supplied routine that is called after a driver is loaded. It is responsible
  for initializing the driver.
- `Real_Driver_Entry`: usually the function where the execution from `DriverEntry` has been transferred to. It is
  usually where the `DeviceName` is initialized.
- `DispatchDeviceControl`/`DispatchInternalDeviceControl`: if the tool was able to recover the functions at some
  specific offsets, the functions will then be renamed with the appropriate name.
- `Possible_DispatchDeviceControl_#`: if the tool was not able to recover `DispatchDeviceControl`
  or `DispatchInternalDeviceControl`, it employs an experimental searching, following the execution flow, and checking
  for cases where the function is loading known `IO_STACK_LOCATION` & `IRP` addresses; indicating that the function
  could be the DispatchDeviceControl. As it is based on heuristic, it could return more than one result, and it is prone
  to false positives.

![](/screenshots/finding-dispatchdevicecontrol.png)

### Labelling WDM and WDF Structures

Several driver structures are shared among all `WDM`/`WDF` drivers. The tool is able to automatically identify these
structures, such as the `IO_STACK_LOCATION`, `IRP`, and `DeviceObject` structures and can help save time during the
reverse engineering process and provide context to areas of the driver where these functions are in use.

![](/screenshots/WDM-structures.png)

### Finding and Decoding IOCTL Codes

While reversing drivers, it is common to come across IOCTL codes as part of the analysis. These codes, when decoded,
reveal useful information and may draw focus to specific parts of the driver where vulnerabilities are more likely to
exist.

By right-clicking on a potential IOCTL code, a context menu option is presented (alternatively using the
`Ctrl+Alt+D` shortcut when the cursor is on the line containing a suspected IOCTL code) and can be used to decode the
value. This will print out a table with all decoded IOCTL codes. By right-clicking on a decoded IOCTL code, in the
disassembly view, it's possible to mark it as invalid; this will leave any non-IOCTL comment intact.

If you right-click, alternatively using the
`Ctrl+Alt+F` shortcut, on the first instruction of the function you believe to be the IOCTL dispatcher (
`DispatchDeviceControl`, `DispatchInternalDeviceControl`, `Possible_DispatchDeviceControl_#`) under the Driver Buddy
Reloaded menu, a “**Decode All**” option appears, this attempt to decode all the IOCTL codes it can find in the
function. This is a bit hacky but most of the time it can speed things up.

- A `DriverName.sys-2021-12-10-TIME_STAMP-IOCTLs.txt`/`DriverName.sys-2021-12-10-TIME_STAMP-IOCTLs.txt_dumb.txt` file,
  containing all the decoded IOCTLs up to that moment, will be written under IDA's DB directory.

![](/screenshots/IOCTL-table.png)
![](/screenshots/decode-IOCTL.png)

### Flagging Functions

Driver Buddy Reloaded has lists of C/C++ functions, opcodes and Windows APIs (defined in
the [vulnerable_function_lists](DriverBuddyReloaded/vulnerable_functions_lists) directory) that are commonly vulnerable
or that can facilitate buffer overflow conditions. All found instances are reported back during the auto-analysis and
can help while looking for possible user-controlled code paths reaching sensitive functions.

![](/screenshots/flagging-functions.png)

### Finding DeviceName

The tool automatically attempts to find the drivers registered device paths (`DeviceName`), if no paths can be found by
looking at Unicode strings inside the binary, then the analyst can manually try to use
Madiant’s [FLOSS](https://github.com/mandiant/flare-floss/) in an attempt to find obfuscated paths.

![](/screenshots/devicename.png)

### Dumping Pooltags

During the auto-analysis, the tool also dumps the `Pooltags` used by the binary in a format that works
with `pooltags.txt`. The output can then be copy-pasted at the end of the file and later picked up by WinDbg.

- A `DriverName.sys-2021-12-10-TIME_STAMP-pooltags.txt` file, containing all the dumped Pooltags, will be written under
  IDA's DB directory.

![](/screenshots/pooltag.png)

## Known Caveats and Limitations

- Only IOCTL values >= `0x10000` will be automatically decoded, thus to prevent an high number of false positives. [Issue #15](https://github.com/VoidSec/DriverBuddyReloaded/issues/15)
- Experimental `DispatchDeviceControl` searching works only for x64 drivers
- Shortcuts are incompatible with F-Secure's [win_driver_plugin](https://github.com/FSecureLABS/win_driver_plugin)
- Shortcuts are incompatible with [findcrypt-yara](https://github.com/polymorf/findcrypt-yara)
- In [find_opcodes.py](DriverBuddyReloaded/find_opcodes.py), the `find_opcode_data` option will prevent Driver Buddy
  Reloaded to find opcodes in data sections. Switching it to `True` will print something along
  this [line](https://github.com/VoidSec/DriverBuddyReloaded/issues/11):
  `Found jnz     short loc_15862 in sub_15820 at 0x00015852`
  Usually, going at the showed address and re-defining the selection as code will bring the searched opcode back.
  **Watch out**: It is prone to false positives!

## Credits and Acknowledgements

- Created in 2021 by [Paolo Stagno](https://voidsec.com/) aka [@Void_Sec](https://twitter.com/Void_Sec):
    - Made it compatible with Python 3.x
    - Made it compatible with IDA 7.x
    - Updated C/C++ function and Windows APIs list
    - Various bug fixing
    - Various improvements
    - Integrated part of the functionalities presents in F-Secure's win_driver_plugin
- [DriverBuddy](https://github.com/nccgroup/DriverBuddy) was originally written by Braden Hollembaek and Adam Pond of
  NCC Group.
- Using Satoshi Tanda's [IOCTL decoder](https://github.com/tandasat/WinIoCtlDecoder).
- The WDF functions struct is based on [Red Plait's work](http://redplait.blogspot.ru/2012/12/wdffunctionsidc.html) and
  was ported to IDA Python by Nicolas Guigo, later updated by Braden Hollembaek and Adam Pond.
- Using Sam Brown's F-Secure [win_driver_plugin](https://github.com/FSecureLABS/win_driver_plugin) to retrieve device
  name and pool tags, specifically Alexander Pick [fork](https://github.com/alexander-pick/win_driver_plugin).
- The original code for adding items to the right-click menu (and possibly some other random snippets) came
  from '[herrcore](https://gist.github.com/herrcore/b3143dde185cecda7c1dee7ffbce5d2c)'.
- Prodly developed using **PyCharm** for [Open Source development](https://www.jetbrains.com/community/opensource/#support) by JetBrains
