Multi-stream is a technology that allows the SSD to receive host hints where “data streams” emerge to make better placement of data.

1. Package Content
   * patches_fio directory - patch(es) for fio benchmark tool
   * patches_nvme-cli directory - patch(es) Linux nvme-cli tool
   * patches_sg3_utils directory - patch(es) for Linux sg3_utils tool
   * patches_kernel directory - patch(es) for Linux kernel
   * patches_qemu directory - patch(es) for qemu
   * example directory - contains example c code demonstrating how Stream Directive is used in an application write
   * README.md - this file 

2. Install dependencies and required tools. Instruction provided in this README uses Ubuntu 16.04 LTS x64.
   $ sudo apt-get -y install build-essential debhelper libaio-dev git
   $ sudo apt-get build-dep linux-image-$(uname -r)

3. Obtain kernel source, patch, build and reboot
   $ git clone git://kernel.ubuntu.com/ubuntu/ubuntu-zesty.git ubuntu-kernel
   $ cd ubuntu-kernel
   $ git checkout -b multi-stream Ubuntu-4.9.0-15.16
   $ git apply ../patches_kernel/0001-consolidated_initial-support-for-nvme-directive-streams-and-sas-multistream.patch
   $ fakeroot debian/rules clean 
   $ AUTOBUILD=1 NOEXTRAS=1 fakeroot debian/rules binary-headers binary-generic 
   The compilation may take sometime. Machine with i7-3770 and 8GB RAM takes about 30min.
   $ sudo dpkg -i ../linux-image*4.9.0*.deb ../linux-headers*4.9.0*.deb
   $ sudo reboot

4. For NVMe device, obtain nvme-cli source. Apply patch, compile and install
   $ git clone https://github.com/linux-nvme/nvme-cli.git
   $ cd nvme-cli
   $ git apply ../patches_nvme-cli/0001-initial-support-for-streams-directive-commands.patch
   $ make && sudo make install
   $ cd ..

5. For SAS device, obtain sg3_utils source. Apply patch, compile and install
   $ git clone https://github.com/hreinecke/sg3_utils
   $ cd sg3_utils
   $ git apply ../patches_sg3_utils/0001-added-multi-stream-support.patch
   $ make && sudo make install
   $ cd ./si && make && sudo make install
   $ cd ../..

6. Obtain fio source. Apply patch, compile and install
   $ git clone https://github.com/axboe/fio.git
   $ cd fio
   $ git apply ../patches_fio/0001-new-fadvise-semantics.patch 
   $ make && sudo make install
   $ cd ..

7. Obtain qemu source. Apply patch, compile and install
   $ git clone git://git.qemu-project.org/qemu.git
   $ cd qemu 
   $ git apply ../patches_qemu/0001-nvme-streams-directive-initial-support.patch 
   $ ./configure --enable-linux-aio --target-list=x86_64-softmmu --enable-kvm --enable-jemalloc
   $ make && sudo make install
   $ cd ..

Notes:
   * For NVMe device, there is a regression test script within the nvme-cli tool after the patch. For a quick test on device supporting Stream Directive:
     # cd nvme-cli
     # sudo ./regress_directive -d /dev/nvme1n1 -w
   * The nvme-cli tool contains a "scirpts" directory that has examples to issue directive send/receive operations. An example fio job file is also availabe in this directory to issue stream writes to SSD. 
   * Under the scripts directory, there are scripts that perform individual directive operations. Please see README.txt under this directory for details.
   * There are blktrace trace point added to the nvme driver. You may monitor the write transaction with the stream ID when running the IO on the NVMe device.
     e.g.,
     # sudo blktrace -d /dev/nvme1n1 -o - | blkparse -i -
	...
	259,1   26        0     0.092882552     0  m   N off:8192; len:8; dtype:1; dspec:1; nsid:1
	259,1   20        0     0.094085987     0  m   N off:16384; len:8; dtype:1; dspec:2; nsid:1
	259,1   20        0     0.095293892     0  m   N off:24576; len:8; dtype:1; dspec:3; nsid:1
	259,1   20        0     0.096466457     0  m   N off:32768; len:8; dtype:1; dspec:4; nsid:1
	...
