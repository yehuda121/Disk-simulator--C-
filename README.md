# Disk-simulator-in-C
Introduction:
This project is a File System Simulation program that emulates file operations on a virtual disk. It allows you to create, open, close, read, write, delete, copy, and rename files within the simulated file system.

Prerequisites:
Before running this program, ensure you have the following prerequisites:
- C++ compiler
- Standard C++ libraries
- POSIX-compatible operating system (for system calls)

How to Use
//using the shell
1. Compile the code:
   g++ main.cpp -o file_system_sim
2. Run the program:
   ./file_system_sim
3. Follow the on-screen menu to interact with the file system.

Features:
- Create and manage files.
- Open and close files.
- Write data to files.
- Read data from files.
- Delete files.
- Copy files.
- Rename files.

Code Structure
- `fsInode`: Represents file system inodes with various attributes.
- `FileDescriptor`: Manages file descriptors associated with opened files.
- `fsDisk`: Simulates a disk with file system operations.
- `main()`: The program's entry point and menu interface.
