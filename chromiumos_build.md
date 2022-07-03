# chromiumos build

### Prepare

- Ubuntu 18.04 / 22.04 on Laptop
- 16GiB RAM
- 200G+ disk

### Software install

#### Install basic software

```bash
sudo apt-get install git gitk git-gui curl xz-utils \
     python3-pkg-resources python3-virtualenv python3-oauth2client \
     docker.io
```

#### Configure git

```
git config --global user.email "you@example.com"
git config --global user.name "Your Name"
```

#### Install depot_tools

```
git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
export PATH=$HOME/depot_tools:$PATH
# Set PATH in bash
echo "export PATH=$HOME/depot_tools:$PATH" >> ~/.bashrc
```

### Get chromiumos source

```bash
mkdir -p ~/chromiumos
cd ~/chromiumos
repo init -u https://chromium.googlesource.com/chromiumos/manifest -b release-R96-14268.B
repo sync -j4
```

### Create cros build environment

```bash
cros_sdk --create
```

### Build chromiumos

This actually uses [crossdev](https://wiki.gentoo.org/wiki/Crossdev) and [binpkg](https://wiki.gentoo.org/wiki/Binary_package_guide)

#### Init board build environment

```bash
# outside
cros_sdk --enter

# inside
export BOARD=amd64-generic
# init board build environment
setup_board --board=${BOARD}
```

Actually `setup_board` uses crossdev to create cross-build-environment

#### Set user password

```bash
./set_shared_user_password.sh
```

#### Compile packages

```bash
./build_packages --board=${BOARD}
```

Use `binpkg` acceleration to avoid to rebuild packages.

#### Build root filesystem

```bash
./build_image --board=${BOARD} --noenable_rootfs_verification test
```

terminal output:
```bash
16:51:46 INFO    : Done. Image(s) created in /mnt/host/source/src/build/images/amd64-generic/R96-14268.91.2022_07_02_1624-a1

16:51:46 INFO    : Test image created as chromiumos_test_image.bin
16:51:46 INFO    : To copy the image to a USB key, use:
16:51:46 INFO    :   cros flash usb:// ../build/images/amd64-generic/R96-14268.91.2022_07_02_1624-a1/chromiumos_test_image.bin
16:51:46 INFO    : To flash the image to a Chrome OS device, use:
16:51:46 INFO    :   cros flash YOUR_DEVICE_IP ../build/images/amd64-generic/R96-14268.91.2022_07_02_1624-a1/chromiumos_test_image.bin
16:51:46 INFO    : Note that the device must be accessible over the network.
16:51:46 INFO    : A base image will not work in this mode, but a test or dev image will.
16:51:46 INFO    : To run the image in a virtual machine, use:
16:51:46 INFO    :   cros_vm --start --image-path=../build/images/amd64-generic/R96-14268.91.2022_07_02_1624-a1/chromiumos_test_image.bin --board=amd64-generic
```

#### Start/Stop qemu virtual manchine

```bash
# start
cros_vm --start --image-path=../build/images/amd64-generic/R96-14268.91.2022_07_02_1624-a1/chromiumos_test_image.bin --board=amd64-generic

# stop
cros_vm --stop
```

start vm & terminal output(use my old build test image):

```bash
# start vm
cros_vm --start --image-path=../build/images/amd64-generic/R96-14268.91.2022_07_02_1054-a1/chromiumos_test_image.bin --board=amd64-generic

checking /mnt/host/source/chroot/usr/libexec/qemu/bin/qemu-system-x86_64
17:01:46.873: INFO: run: /mnt/host/source/chroot/usr/libexec/qemu/bin/qemu-system-x86_64 --version
17:01:47.225: INFO: QEMU version 5.2.0
17:01:47.225: INFO: Pid file: /tmp/cros_vm_9222/kvm.pid
17:01:47.226: INFO: run: kill -9 3012194
17:01:47.233: INFO: SSH port 9222 in use...
17:01:52.251: INFO: run: /mnt/host/source/chroot/usr/libexec/qemu/bin/qemu-system-x86_64 -m 8G -smp 8 -vga virtio -daemonize -pidfile /tmp/cros_vm_9222/kvm.pid -chardev 'pipe,id=control_pipe,path=/tmp/cros_vm_9222/kvm.monitor' -serial file:/tmp/cros_vm_9222/kvm.monitor.serial -mon 'chardev=control_pipe' -cpu 'SandyBridge,-invpcid,-tsc-deadline,check,vmx=on' -usb -device usb-tablet -device 'virtio-net,netdev=eth0' -device 'virtio-scsi-pci,id=scsi' -device virtio-rng -device 'scsi-hd,drive=hd' -drive 'if=none,id=hd,file=/mnt/host/source/src/build/images/amd64-generic/R96-14268.91.2022_07_02_1054-a1/chromiumos_test_image.bin,cache=unsafe,format=raw' -netdev 'user,id=eth0,net=10.0.2.0/27,hostfwd=tcp:127.0.0.1:9222-:22' -enable-kvm
VNC server running on 127.0.0.1:5900
17:01:52.342: INFO: run: ssh -p 9222 '-oConnectTimeout=30' '-oConnectionAttempts=4' '-oNumberOfPasswordPrompts=0' '-oProtocol=2' '-oServerAliveInterval=15' '-oServerAliveCountMax=8' '-oStrictHostKeyChecking=no' '-oUserKnownHostsFile=/dev/null' '-oIdentitiesOnly=yes' -i /tmp/ssh-tmpisa_zb5b/testing_rsa root@localhost -- true
17:02:10.568: INFO: (stdout):
Warning: Permanently added '[localhost]:9222' (ED25519) to the list of known hosts.
```

##### bug 1 nested virtualization

I used PVE to create an ubuntu virtual machine.
The output of chromiumos after starting is as follows

```bash
sudo -- /mnt/host/source/chroot/usr/libexec/qemu/bin/qemu-system-x86_64 -m 8G -smp 8 -vga virtio -chardev 'pipe,id=control_pipe,path=/tmp/cros_vm_9222/kvm.monitor' -serial file:/tmp/cros_vm_9222/kvm.monitor.serial -mon 'chardev=control_pipe' -cpu 'SandyBridge,-invpcid,-tsc-deadline,check' -usb -device usb-tablet -device 'virtio-net,netdev=eth0' -device 'virtio-scsi-pci,id=scsi' -device virtio-rng -device 'scsi-hd,drive=hd' -drive 'if=none,id=hd,file=/mnt/host/source/src/build/images/amd64-generic/R96-14268.91.2022_07_01_2102-a1/chromiumos_test_image.bin,cache=unsafe,format=raw' -netdev 'user,id=eth0,net=10.0.2.0/27,hostfwd=tcp:127.0.0.1:9222-:22' -enable-kvm
VNC server running on 127.0.0.1:5900
KVM: entry failed, hardware error 0x7
EAX=00000000 EBX=00000000 ECX=00000000 EDX=000206a1
ESI=00000000 EDI=00000000 EBP=00000000 ESP=00000000
EIP=0000fff0 EFL=00000002 [-------] CPL=0 II=0 A20=1 SMM=0 HLT=0
ES =0000 00000000 0000ffff 00009300
CS =f000 ffff0000 0000ffff 00009b00
SS =0000 00000000 0000ffff 00009300
DS =0000 00000000 0000ffff 00009300
FS =0000 00000000 0000ffff 00009300
GS =0000 00000000 0000ffff 00009300
LDT=0000 00000000 0000ffff 00008200
TR =0000 00000000 0000ffff 00008b00
GDT=     00000000 0000ffff
IDT=     00000000 0000ffff
CR0=60000010 CR2=00000000 CR3=00000000 CR4=00000000
DR0=0000000000000000 DR1=0000000000000000 DR2=0000000000000000 DR3=0000000000000000
DR6=00000000ffff0ff0 DR7=0000000000000400
EFER=0000000000000000
Code=04 66 41 eb f1 66 83 c9 ff 66 89 c8 66 5b 66 5e 66 5f 66 c3 <ea> 5b e0 00 f0 30 36 2f 32 33 2f 39 39 00 fc 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
```

This error can be bypassed by not using -enable-kvm, but it is very slow.
It seems to be an instruction set issue, because dell r720 is X79 platform.
It is normal for modern instruction sets to be missing.
No problems associated with 11th core laptop.

#### SSH connect vm

User: root Password: test0000 Port: 9222

```
ssh root@localhost -p 9222
```

#### Put google api key to system

```bash
# remount read-only root filesystem
mount -o remount,rw /
cat << EOF >> /etc/chrome_dev.conf
GOOGLE_API_KEY=AIzaxxxxxx
GOOGLE_DEFAULT_CLIENT_ID=16580xxxxxxxxx
GOOGLE_DEFAULT_CLIENT_SECRET=GOCSPXxxxxxx
EOF
```

##### bug 2 nano

Paste the text and it will trigger the bug

```bash
Bad toggle -- please report a bug
```

Using vim to fix it.

#### VNC connect vm

```
remmina -c vnc://localhost:5900
```

### Kernel replacement


amd64-generic rootfs in /build/amd64-generic/

```bash
# 9999 missing keyword
mkdir -p /build/amd64-generic/etc/portage/package.keywords
cat <<EOF
sys-kernel/chromeos-kernel-5_10
EOF | sudo tee -a /build/amd64-generic/etc/portage/package.keywords/cros

# Unmask 5.10 kernel
mkdir -p /build/amd64-generic/etc/portage/package.unmask
cat <<EOF
sys-kernel/chromeos-kernel-5_10
EOF | sudo tee -a /build/amd64-generic/etc/portage/package.unmask/cros

# Use virtual/linux-sources kernel-5_10 USE to enable 5.10 kernel
mkdir -p /build/amd64-generic/etc/portage/package.use/
cat <<EOF
virtual/linux-sources -kernel-4_14 kernel-5_10
EOF | sudo tee -a /build/amd64-generic/etc/portage/package.use/cros

# Uninstall 4.14 kernel
emerge-${BOARD} -C chromeos-kernel-4_14

# Rebuild virtual/linux-sources 
emerge-${BOARD} --nodeps -av1 virtual/linux-sources

# Install 5.10 kernel
emerge-${BOARD} --nodeps -avg chromeos-kernel-5_10

# Update kernel
~/chromiumos/src/scripts/update_kernel.sh --remote=127.0.0.1   --ssh_port 9222
```

check kernel version:

```bash
uname -a
# Linux localhost 5.10.118 #1 SMP PREEMPT Sat Jul 2 11:47:12 CST 2022 x86_64 Intel Xeon E312xx (Sandy Bridge) GenuineIntel GNU/Linux
```

### Devserver on Docker

tips: pass in the https_proxy variable

#### Dockerfile

```Dockerfile
FROM ubuntu:22.04

ARG HTTPS_PROXY

ENV DEBIAN_FRONTEND noninteractive

RUN apt update && apt install -y git gitk git-gui curl xz-utils \
        python3-pkg-resources python3-virtualenv python3-oauth2client sudo

RUN useradd cros --create-home
RUN echo 'cros ALL=(ALL:ALL) NOPASSWD: ALL' >> /etc/sudoers

USER cros

RUN git config --global user.email "dev@example.com" && \
        git config --global user.name "dev"

RUN git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git /home/cros/depot_tools

ENV PATH /home/cros/depot_tools:$PATH
RUN mkdir /home/cros/chromiumos
WORKDIR /home/cros/chromiumos

RUN repo init -u https://chromium.googlesource.com/chromiumos/manifest.git -g minilayout -b release-R96-14268.B

ENV https_proxy ${HTTPS_PROXY}

RUN repo sync -v -j4

RUN sudo apt install locales && \
    sudo locale-gen en_US.UTF-8 && \
    sudo update-locale LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8 LC_MESSAGES=en_US.UTF-8

ENV HTTPS_PROXY=
ENV https_proxy=

EXPOSE 8080
ENTRYPOINT [ "/home/cros/depot_tools/cros_sdk" ]
```

#### build docker

```bash
docker build -t cros_sdk:test \
    --build-arg HTTPS_PROXY="$https_proxy" \
    --no-cache -f Dockerfile 
```

#### run docker

```bash
docker run -it --privileged --network host -e https_proxy="$https_proxy" cros_sdk:test
```

The page display is as follows
```
Welcome to the Dev Server!

Here are the available methods, click for documentation:

build
check_health
controlfiles
is_staged
latestbuild
list_image_dir
list_suite_controls
locate_file
setup_telemetry
stage
symbolicate_dump
update
xbuddy
xbuddy_capacity
xbuddy_translate
```

#### Problem

I was planning to create a cros_sdk chroot when creating docker, but Docker does not support privileged creation of container images.


### Connecting the devserver

#### Set correct `lsb-release` devserver

`lsb-release`:

```
CHROMEOS_RELEASE_NAME=Chromium OS
CHROMEOS_AUSERVER=http://revy-tp:8080/update
CHROMEOS_DEVSERVER=http://revy-tp:8080
CHROMEOS_RELEASE_KEYSET=devkeys
CHROMEOS_RELEASE_TRACK=testimage-channel
CHROMEOS_RELEASE_BUILD_TYPE=Test Build - revy
CHROMEOS_RELEASE_DESCRIPTION=14268.91.2022_07_02_1930 (Test Build - revy) developer-build amd64-generic
CHROMEOS_RELEASE_BOARD=amd64-generic
CHROMEOS_RELEASE_BRANCH_NUMBER=91
CHROMEOS_RELEASE_BUILD_NUMBER=14268
CHROMEOS_RELEASE_CHROME_MILESTONE=96
CHROMEOS_RELEASE_PATCH_NUMBER=2022_07_02_1930
CHROMEOS_RELEASE_VERSION=14268.91.2022_07_02_1930
GOOGLE_RELEASE=14268.91.2022_07_02_1930
```

I choose to modify /etc/hosts

```bash
# remount /
mount -o remount,rw /
echo "10.x.x.x revy-tp" >> /etc/hosts
```

#### Generate an update payload 

```bash
# Generate payload
../../chromite/bin/cros_generate_update_payload \
--src-image=chromiumos_test_image_4.14.bin \
--tgt-image=chromiumos_test_image_5.10.bin \
--output=dist/payload.bin

ls dist/
# payload.bin  payload.bin.json  payload.bin.log
```

#### Copy to Docker

##### Method1: Use python3 simplehttpserver to start fileserver

```
# in native cr build env
pushd dist
     python3 -m http.server -p 8081 &
popd

# in docker
docker exec -it xxxx /bin/bash

pushd ~/chromiumos/src/platform/dev/static
     curl -O http://localhost:8081/payload.bin
     curl -O http://localhost:8081/payload.bin.json
popd
```

##### Method2: docker cp

```
docker cp /home/revy/chromiumos/src/scripts/dist/payload.bin* xxx:/home/revy/chromiumos/src/platform/dev/static
```

#### Update Chromium OS

```bash
update_engine_client --update
```

*Client output error*

```
2022-07-02T09:08:34.515298Z INFO update_engine_client: [update_engine_client.cc(514)] Forcing an update by setting app_version to ForcedUpdate.
2022-07-02T09:08:34.515346Z INFO update_engine_client: [update_engine_client.cc(516)] Initiating update check.
2022-07-02T09:08:34.515785Z INFO update_engine_client: [update_engine_client.cc(550)] Waiting for update to complete.
2022-07-02T09:08:34.641494Z ERROR update_engine_client: [update_engine_client.cc(211)] Update failed, current operation is UPDATE_STATUS_IDLE, last error code is ErrorCode::kOmahaErrorInHTTPResponse(37)
```

*Server output error*

```
ERROR:root:Failed to read app data from chromiumos_base_image.bin-package-sizes.json ('appid')
[02/Jul/2022:19:23:47] HTTP
Traceback (most recent call last):
  File "/usr/lib64/python3.6/site-packages/cherrypy/_cprequest.py", line 630, in respond
    self._do_respond(path_info)
  File "/usr/lib64/python3.6/site-packages/cherrypy/_cprequest.py", line 689, in _do_respond
    response.body = self.handler()
  File "/usr/lib64/python3.6/site-packages/cherrypy/lib/encoding.py", line 221, in __call__
    self.body = self.oldhandler(*args, **kwargs)
  File "/usr/lib64/python3.6/site-packages/cherrypy/_cpdispatch.py", line 54, in __call__
    return self.callable(*self.args, **self.kwargs)
  File "/usr/lib/devserver/devserver.py", line 1133, in update
    return updater.HandleUpdatePing(data, label, **kwargs)
  File "/usr/lib64/devserver/autoupdate.py", line 153, in HandleUpdatePing
    update_app_index=nebraska.AppIndex(local_payload_dir))
  File "/usr/lib64/devserver/nebraska.py", line 567, in __init__
    self._Scan()
  File "/usr/lib64/devserver/nebraska.py", line 583, in _Scan
    app = AppIndex.AppData(metadata)
  File "/usr/lib64/devserver/nebraska.py", line 704, in __init__
    self.appid = app_data[self.APPID_KEY]
KeyError: 'appid'
```

Tried several times but still can't resolve the error.

### Reference

[developer_guide](https://chromium.googlesource.com/chromiumos/docs/+/master/developer_guide.md)

[Kernel Development](https://chromium.googlesource.com/chromiumos/docs/+/HEAD/kernel_development.md)

[devserver](https://chromium.googlesource.com/chromiumos/chromite/+/refs/heads/master/docs/devserver.md)

[depot_tools](https://commondatastorage.googleapis.com/chrome-infra-docs/flat/depot_tools/docs/html/depot_tools_tutorial.html#_setting_up)

[Ubuntu update-locale](https://manpages.ubuntu.com/manpages/bionic/man8/update-locale.8.html)

[Chromium OS Debugging Tips](https://www.chromium.org/chromium-os/how-tos-and-troubleshooting/debugging-tips/)

[Chrome OS Update Process - Update-Payload-Generation](https://chromium.googlesource.com/aosp/platform/system/update_engine/+/HEAD/README.md#Update-Payload-Generation)

[Portage](https://wiki.gentoo.org/wiki/Portage)

[crossdev](https://wiki.gentoo.org/wiki/Crossdev)

[binpkg](https://wiki.gentoo.org/wiki/Binary_package_guide)
