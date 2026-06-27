# VFD SWMR Utilities

aux_process.c:
==============
The `aux_process` utility applies a sequence of updater files to generate a locally maintained copy of the VFD SWMR metadata file.

This utility is primarily intended for use on the reader system in NFS-based workflows, where direct access to up-to-date metadata produced by the writer may be delayed.

The updater files are expected to be generated incrementally during writer execution and are processed in order to reconstruct the latest available metadata state.

**Usage:** 
```bash
aux_process [options] <md_file> <ud_path>
```

**Where:**  
  - `<md_file>`  
  The path to the metadata file. Must be on a POSIX file system. Note that the file may not exist yet.  
  - `<ud_path>`   
  The path of the updater files including the directory. For example, updater files named `updater_file.0`, `updater_file.1`, ..., `updater_file.n` should be specified as `/path/to/updater_file`. This will typically be in an NFS mounted file system.

**Options:**  
  - `-a --skip_aux`  
  Exit if VDS across multiple file is being enabled (to be implemented in the future).  
  - `-l --log_file`  
  Path to the log file. Default: no log file.  
  - `-m --md_chksum_path`  
  Path to the file containing the checksum values for testing purpose.  
  - `-p --polls_per_tick`  
  Number of times to poll for a new updater file per tick. Default: 10.  
  - `-s --stats`  
  Display stats on exit.  
  - `-t --tick_len`  
  Integer value indicating the tick length in tenths of a second.  
  - `-v --verbose`  
  Write log entries to stdout.

**Example:**
```bash
aux_process --verbose my_md_file /path/to/updater_file
```

**Note:**  
The `--log_file` option may need to be reworked. Errors are currently written to `stderr`, and selected command-line options are written to `stdout`; neither is written to the specified log file.

recovery_tool.c:
================
The `recovery_tool` applies a sequence of updater files to an HDF5 file in order to reconstruct a consistent metadata state after interruption.

This process restores the file to a state where it can be safely reopened using standard HDF5 APIs.

**Usage:** 
```bash
recovery_tool [options] <h5_file> <ud_path>
```

**Where:**
  - `<h5_file>`  
  Path to the HDF5 file.
  - `<ud_path>`  
  Path prefix of the updater files, including the directory. For example, updater files named `updater_file.0`, `updater_file.1`, ..., `updater_file.n` should be specified as `/path/to/updater_file`. 

**Options:**
  - `-h --help`  
  Print the usage message and exit.  
  - `-p --posix`  
  Indicate that the HDF5 file is on POSIX file system; HDF5 file will be kept open during the sequence of the metadata modifications. (Currently, only POSIX-compliant systems are supported).  
  - `-v --verbose`  
  Prints detailed information about each updater file being processed, including headers, change lists, and data operations, to stdout.  
  - `-l --log_file <log_file>`  
  Specify path of a log file for log entries. (Will ignore verbose option) 

**Example:**
```bash
recovery_tool --verbose path/to/h5_file.h5 /path/to/updater_file
```

**Requirement:**  
The `h5clear` utility must be available to this program. The path to `h5clear` must either be present in the system `PATH` or specified through the `H5CLEAR_PATH` environment variable.

### Notes:
#### Platform support
This tool currently supports POSIX-compliant systems only and is not expected to function on Windows.

#### Logging behavior
The `--log_file` option is incomplete: some output is still written to `stdout` or `stderr` regardless of this setting, and logging behavior is not fully consistent with the `--verbose` option.


crasher.c
=========
This utility is used to test the recovery tool by deliberately terminating a running process. It executes the provided command and then sends a SIGKILL (kill -9) signal after a specified delay. This makes it possible to interrupt a VFD SWMR writer at any point during HDF5 file creation and evaluate how recoverable the resulting file is.

By default, the command’s standard output is redirected to \<command\>.out. This behavior can be overridden with the `-p` option to print output directly to the console.

**Usage:**
```bash
crasher [options] <delay> <command> [args]
```

**Where:**  
  - `<delay>`  
  Time in seconds to wait before crashing the process.  
  Decimal values are supported (e.g., 1.5, 0.25).  
  Maximum precision: 6 decimal places. 
  - `<command> [args]`  
  The command to execute. Any additional arguments are passed directly to the command.

**Options:**
  - `-h`  
  Print the usage message and exit.  
  - `-v`  
  Prints detailed information.  
  - `-p`  
  Prints command's output to console instead of redirecting to \<command\>.out.

**Example:**
```bash
crasher -v -p 5 ./my_program arg1 arg2
```

test_crash_recovery.sh
======================
### Purpose
Intended for development purposes.

This script automates validation of the VFD SWMR recovery mechanism. It is intended for developers testing changes to the recovery process or verifying that VFD SWMR writer programs can be recovered correctly after unexpected termination. The script repeatedly exercises the recovery workflow across a range of crash timings and validates the recovered HDF5 files automatically.

### Usage
```bash
./test_crash_recovery.sh [-h] [-d] [-v] [-k] [test1 test2 ...]
```

Run crash recovery tests for VFD SWMR.
For each test iteration, a VFD SWMR writer process is executed and then forcibly
terminated after a specified delay using the crasher utility. The resulting HDF5
state is then recovered using the recovery_tool and validated using H5LS and H5DUMP
before and after recovery to verify correctness of the recovery process.

### Warning 
WARNING: This test script has the potential to write multiple TERABYTES of data to your
filesystem. Even if you do not keep the generated files (using the `-k` option),
the script still performs all writes during each test iteration. The difference
is that files are deleted after each run, so disk usage does not accumulate, but
total write volume to the storage device remains the same.

This may result in significant wear on storage devices and long execution times,
depending on the number of tests and dataset sizes used, as well as the type of
storage used (ssd vs hhd).

### Options
  - `-h`  Show the help message and exit
  - `-d`  Choose a specific delay in seconds (e.g. 0.5, 1.1, ...)
          NOTE: A single specific test must be selected with this option.
  - `-v`  Enable verbose output
  - `-k`  Keep output files from each iteration. Useful for debugging.  
          WILL GENERATE DOZENS, IF NOT HUNDREDS, OF FILES.  
          THIS MAY CONSUME TERABYTES OF DISK SPACE

Files preserved when using the -k option:
```text
  <test>_recovery.out.<count>            - Recovery tool output and error messages  
  <test>_h5clear_pre.out.<count>         - H5clear status reset before validation  
  <test>_h5clear_post.out.<count>        - H5clear status reset after validation  
  <test>_validation_pre.out.<count>      - File validation before recovery  
  <test>_validation_post.out.<count>     - File validation after recovery  
  <writer>.out.<count>                   - Writer tool output and error messages  
  <expected HDF5 file(s)>.<count>        - Generated HDF5 files, including the base file  
                                            and derived variants (<name>_a.h5, <name>_b.h5)  
  Where:
    - <test> is the test name (standard, bigset, sparse, or remove)
    - <count> is the iteration number
    - <writer> is the name of the actual writer program used in the test.
```

### Test selections:
```text
  standard  Run standard writer crash test (vfd_swmr_writer)
  bigset    Run bigset writer crash test (vfd_swmr_bigset_writer)
  sparse    Run sparse writer crash test (vfd_swmr_sparse_writer)
  remove    Run remove writer crash test (vfd_swmr_remove_writer)
```
If no tests are specified, all tests will be run.

### Additional information
All output files are placed in a newly made crash_test/ directory inside the current working directory.

Examples:
```bash
  ./test_crash_recovery.sh                    # Run all tests
  ./test_crash_recovery.sh -v -k standard     # Run standard test with verbose output and keep files
  ./test_crash_recovery.sh bigset sparse      # Run only bigset and sparse tests
  ./test_crash_recovery.sh -d 0.5 bigset      # Run a single crash test for bigset with 0.5s delay
```

Test Process (for each iteration):
1. Generate initial HDF5 file (if required by the test)
2. Run the writer tool and crash it after <delay> seconds using crasher utility
3. Run h5clear to reset status flags on the crashed file
4. Validate the file before recovery using h5ls and h5dump (might fail might not)
5. Apply updater files using recovery_tool to recover the HDF5 file
6. Test recovery using h5ls and h5dump (expected to succeed)
7. Clean up temporary files (unless -k flag is used or HDF5_NOCLEANUP is set)

The test continues incrementing the crash delay until the writer tool completes normally without being 
crashed, at which point the test for that configuration ends.

**Configurable Environment Variables:** 
  - HDF5TestExpress  
  Controls test thoroughness (0=exhaustive, 1=default, 2+=quick)

  - ONLY_RUN_EACH_TEST_ONCE  
  Some tests define multiple configuration sets. By default, this option is enabled and each test runs only the first configuration set. Setting this variable to false enables execution of all configuration sets, which can significantly increase total runtime.

*Example:* `HDF5TestExpress=0 ONLY_RUN_EACH_TEST_ONCE=false ./test_crash_recovery.sh`

> *Note:* The script should automatically select the correct project dir, but will fail 
> if you move relevant files from their expected spots.


exec_local_socket_test.sh
=========================
### Purpose
Intended for development purposes.

This script is the local-socket counterpart to `exec_nfs_socket_test.sh`. It provides a convenient way to run and debug the VFD SWMR socket communication tests locally without requiring an NFS-mounted filesystem.

The script supports the `attrdset`, `bigset`, `dsetchks`, `dsetops`, `gfail`, `group`, and `zoo` VFD SWMR test programs using the same options as `test/test_vfd_swmr.sh`. Individual tests may be selected, or multiple tests may be run in a single invocation.

The writer and reader roles are intended to be run in separate terminal sessions, communicating over local sockets. All generated files are placed in a `local_socket_test/` directory created in the current working directory.

### Usage
```bash
./exec_local_socket_test [-h] <test> <role>
```

**Where:**  
- `-h`
Prints a help message and exits.

- `<test>`  
The VFD SWMR test program to run.
Can be one of the following:
  'all', 'attrdset', 'bigset', 'gfail', 'group',  
  'group_basic', 'group_attrs', 'os_group_attrs', or 'zoo'.  
  Note: 'all' runs all tests, 'group' runs all group-related
  tests.

- `<role>`  
'reader' or 'writer' to indicate which role to run.  
Also accepts just 'r' or 'w'.


exec_nfs_socket_test.sh
=======================
### Purpose
Intended for development purposes.

This script is the NFS-based counterpart to `exec_local_socket_test.sh`. It provides a convenient way to run and debug the VFD SWMR socket communication tests in a networked environment using an NFS-mounted filesystem.

The script supports the `attrdset`, `bigset`, `dsetchks`, `dsetops`, `gfail`, `group`, and `zoo` VFD SWMR test programs using the same options as `test/test_vfd_swmr.sh`. It has been configured to pass the writer's IP address to each reader process to establish the socket connection and uses slightly longer delays for selected tests to account for NFS filesystem latency. Individual tests may be selected, or multiple tests may be run in a single invocation.

The writer and reader roles are intended to be run on separate systems that share an NFS-mounted working directory. All generated files are placed in an `nfs_socket_test/` directory created in the current working directory.

> *Note:* The IP_ADDRESS variable at the top of script must be editted to contain a valid IP address string of the writer system for socket connection.

### Usage
```bash
./exec_nfs_socket_test.sh <test> <role> [md_dir]
```
**Where:**
- `-h`
Prints a help message and exits.

- `<test>`  
The VFD SWMR test program to run.
Can be one of the following:
  'all', 'attrdset', 'bigset', 'gfail', 'group',  
  'group_basic', 'group_attrs', 'os_group_attrs', or 'zoo'.  
  Note: 'all' runs all tests, 'group' runs all group-related
  tests.

- `<role>`  
'reader' or 'writer' to indicate which role to run.  
Also accepts just 'r' or 'w'.

### Additional Information
The writer and reader roles should be started on separate systems, with the writer started first so that the socket connection can be established correctly.

For the `bigset` test, the auxiliary process must create the external metadata file on a local POSIX filesystem. Therefore, when running the reader, [md_dir] must specify a valid local POSIX directory. This argument is ignored by the writer.