Configurable VFD SWMR and Recovery Tool
---------------------------------------

This repository is a modified version of the on feature/vfd_swmr branch of the main [HDF5 GitHub repo](https://github.com/HDFGroup/hdf5/) based on the HDF5 version 1.13.2-1. It contains the implementation of the HDF5 Virtual File Driver (VFD) Single-Writer/Multiple-Reader (SWMR) framework, a configurable runtime infrastructure for coordinated SWMR applications, and a recovery tool for restoring HDF5 file consistency after failures or abnormal termination.

For licensing information see COPYING_* files in the top source directory.

Major features include:

- Runtime configuration of VFD SWMR readers and writers through a common text-based configuration language.
- Support for coordinated reader/writer deployment using configuration files or environment variables.
- Metadata journaling for improved robustness and fault tolerance on POSIX and NFS file systems.
- Updated `h5dump` and `h5ls` command-line tools to use configuration file when reading HDF5 file under construction.
- A recovery tool that reconstructs consistent HDF5 file state from journal information following interrupted write operations.
- [Design document](doc/VFD_SWMR_Parallel_Page_buffer_sketch_260601.pdf) for extending VFD SWMR to parallel HDF5 environments.
- User and developer documentation, demonstration examples, and utilities supporting journaling and recovery workflows.

The project improves the usability, reliability, deployability, and resilience of HDF5 applications operating in SWMR mode.

For description of VFD SWMR, installation instructions and usage on POSIX and NFS systems see [VFD SWMR User's Guide](doc/vfd-swmr-user-guide.md). Check [utils/vfd_swmr/README.md](utils/vfd_swmr/README.md) file instructions how to run recovery tool and NFS VFD SWMR tests.

Limitations:

- The VFD SWMR tests were not ported to Windows platforms.
- Limited testing: the distrucbition was tested only on Linux and macOS platforms with gcc and clang compilers.
- VFD SWMR and recovery tool tests have to be run manually.

Contact info@lifeboat.llc if you need help with the configurable VFD SWMR and recovery tool, or if you have questions or suggestions.




