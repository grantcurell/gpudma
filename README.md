# GPUDirect RDMA Example

## Helpful Links

https://stackoverflow.com/questions/52125610/visual-studio-remote-linux-headers

## Installation on Windows
1. Clone repo and get source code of NVIDIA-Linux-x86_64-X.Y driver the same version as installed on your system.
2. Extract it to the gpudma project directory and create symbolic link "nvidia" on the NVIDIA-Linux-x86_64-X.Y driver directory.
Default location is ~/gpudma; For another location you must set variable GPUDMA_DIR, for example: GPUDMA_DIR=/xprj/gpudma TODO update
3. Build the NVIDIA driver in nvidia/kernel. We need only Module.symvers file from nvidia/kernel directory.
4. Build gpumem module.
5. Build application app
6. Build application app_template

## Installation on Linux

### Platform Used for Testing

Hardware Dell R840 with Mellanox ConnectX-6 and Nvidia P4000

```
NAME="Red Hat Enterprise Linux"
VERSION="8.4 (Ootpa)"
ID="rhel"
ID_LIKE="fedora"
VERSION_ID="8.4"
PLATFORM_ID="platform:el8"
PRETTY_NAME="Red Hat Enterprise Linux 8.4 (Ootpa)"
ANSI_COLOR="0;31"
CPE_NAME="cpe:/o:redhat:enterprise_linux:8.4:GA"
HOME_URL="https://www.redhat.com/"
DOCUMENTATION_URL="https://access.redhat.com/documentation/red_hat_enterprise_linux/8/"
BUG_REPORT_URL="https://bugzilla.redhat.com/"

REDHAT_BUGZILLA_PRODUCT="Red Hat Enterprise Linux 8"
REDHAT_BUGZILLA_PRODUCT_VERSION=8.4
REDHAT_SUPPORT_PRODUCT="Red Hat Enterprise Linux"
REDHAT_SUPPORT_PRODUCT_VERSION="8.4"
Red Hat Enterprise Linux release 8.4 (Ootpa)
Red Hat Enterprise Linux release 8.4 (Ootpa)
```

### Instructions

TODO update

1. While in theory you can install the drivers for a new version by compiling for it, I was unable to get this to work. Subsequently I suggest you run  `subscription-manager release --set=8.4` to ensure your version of RHEL stays at the desired version.
2. Run:
```bash
subscription-manager repos --enable rhel-8-for-$(uname -i)-baseos-debug-rpms
subscription-manager repos --enable rhel-8-for-$(uname -i)-baseos-source-rpms
subscription-manager repos --enable rhel-8-for-$(uname -i)-appstream-debug-rpms
subscription-manager repos --enable rhel-8-for-$(uname -i)-appstream-source-rpms
dnf group install "Development Tools" -y
dnf install -y tk tcsh tcl gcc-gfortran kernel-modules-extra gcc-g++ gdb rsync ninja-build make zip
```

3. Download the RHEL version of https://docs.mellanox.com/category/mlnxofedib.
   1. If you are using another distribution these instructions should still generally work
4. Once you have downloaded the MLNX_OFED ISO mount it on your RHEL installation and run 
5. Find driver with `sudo update-pciids` TODO - update
6. Load the new drivers with `/etc/init.d/openibd restart`
7. git clone https://github.com/karakozov/gpudma.git
8. cp ~/Downloads/NVIDIA-Linux-x86_64-367.57.run ~/gpudma
9. ./NVIDIA-Linux-x86_64-367.57.run -x
10. ln -svf NVIDIA-Linux-x86_64-367.57 nvidia 
11. cd ~/gpudma/nvidia/kernel && make
12. cd ~/gpudma/module && make
13. cd ~/gpudma/app && make

## Load driver

1. cd ~/gpudma/module && ./drvload.sh
2. Check driver: ls -l /dev/gpumem
3. crw-rw-rw-. 1 root root 10, 55 Apr  2 21:57 /dev/gpumem

## Run app example

cd ~/gpudma/app && ./gpu_direct

The application will create a CUDA context and allocate GPU memory. The memory pointer of the allocated area is then passed to the gpumem module. The Gpumem module then gets the addresses of all of the physical pages of the allocated area and the GPU page sizes. (TODO - I'm a bit confused by what is meant here.)

Gpumem module get address of all physical 
pages of the allocates area and GPU page size. Application can get addresses and do mmap(), 
fill data pattern and free all of them. Than release GPU memory allocation and unlock pages.

Test must finish with the message: "Test successful"

## Build and run app_template

app_template must be built with Nsight Eclipse Edition from NVIDIA.

Command line for launch:  **app_template** **-count** ncount **-size** nsize
* ncount - block counts for read, 0 - for infinity cycle; Default is 16;
* nsize  - size of one buffers in kbytes. Maximum size is 65536. Default is 256;

Main mode is infinity cycle (ncount=0). There are two command for launch application:
* run_cycle_1M - launch with buffers of 1 megabytes
* run_cycle_64M - launch with buffers of 64 megabytes

Infinity cycle must be executed only from console. Nsight Eclipse Edition cannot correct display status line with "\r" symbol. If you can do it then send me about it, please.
For launch application from Nsight Eclipse Edition use non-zero value for count argument. This is enough for debugging.

There are main executing stages:

1. Create exemplar TF_TestCnt - launch thread for working with CUDA
2. Prepare
  1. Open device
  2. Allocate three buffers with size <nsize> and map in the BAR1 - class CL_Cuda
  3. Allocate 64 kbytes buffer for struct TaskMonitor
  4. Allocate page-locked HOST memory for td->hostBuffer 
  5. Allocate page-locked HOST memory for struct TaskHostMonitor

3. Launch main cycle - TF_TestCnt::Run()
  1. Launch thread for filling buffers - TF_TestCnt::FillThreadStart()
  2. Launch kernel for checking data - run_checkCounter()
  3. Check flag in the host memory and start DMA transfer - cudaMemcpyAsync()
  4. Check data: TestCnt::CheckHostData()
  5. Measuring velocity of data transfer

4. Periodical launch function TF_TestCnt::StepTable() from function main() for display status information. It is working only for infinity cycle mode. Function display several parameters:
  1. CUDA_RD - number of received buffers to CUDA
  2. CUDA_OK - number of correct buffers to CUDA
  3. CUDA_ERR - number of incorrect buffers to CUDA
  4. HOST_RD - number of received buffers to HOST
  5. HOST_OK - number of correct buffers to HOST
  6. HOST_ERR - number of incorrect buffers to HOST
  7. E2C_CUR - current velocity of data transfer from external device to CUDA
  8. E2C_AVR - average velocity of data transfer from external device to CUDA
  9. C2H_CUR - current velocity of data transfer from CUDA to HOST
  10. C2H_AVR - average velocity of data transfer from CUDA to HOST

5. Function run_checkCounter() launch wrap of 32 thread for checking data. 
  Thread 0 is difference from another:
   * Read ts->irqFlag in the global memory and write it in the local wrap memory.
   * Write checking data to output buffers

  Thread 0 and another threads :
   * Check flag ptrMonitor->flagExit and exit if it is set.
   * Check received data  
   * Write first 16 errors to struct "check"

6. Display result after exiting from main cycle - TF_TestCnt::GetResult()
7. Free memory

Some notes:
* app_template/create_doc.sh - create documentation via doxygen
* There is class CL_Cuda_private for internal data for CL_Cuda
* There is file task_data.h with structs:
  * TaskData - internal task for TF_TestCnt
  * TaskMonitor - struct for shared memory in the CUDA 
  * TaskHostMonitor - struct for shared memory in the HOST
  * TaskBufferStatus - struct for work with one buffer
  * TaskCheckData - struct for error data
  * const int TaskCounts=32 - number of threads in the wrap

