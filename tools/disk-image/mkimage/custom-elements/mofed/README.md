# NVIDIA MLNX_OFED Linux Drivers Installation for Rocky 8.7

This element will map to DIB the MLNX_OFED ISO file downloaded to the build host, and install MLNX_OFED compiled for kernel release used in guest image

Linux Drivers URL:
https://www.mellanox.com/products/infiniband-drivers/linux/mlnx_ofed

Requires:
- MLNX_OFED ISO file on build host
- export DIB_MOFED_FILE=<ISO file location on build host>
- when building MLNX_OFED on CentOS-Stream image modify install.d/70-mofed-install script to include the CentOS-Stream matching RHEL base distribution in mlnxofedinstall flags, for example "--distro RHEL8.7"
