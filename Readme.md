1. Install Docker:
  sudo apt-get docker.io
  sudo usermod -aG docker $USER   //Add user into group docker
  reboot device
  docker images   //Test docker

2. Create Dockerfile
  FROM ubuntu:22.04

  ENV DEBIAN_FRONTEND=noninteractive

  RUN apt-get update && apt-get install -y gawk wget git-core diffstat unzip \
              texinfo gcc-multilib build-essential chrpath socat cpio python-is-python3 \
              python3 python3-pip python3-pexpect xz-utils debianutils iputils-ping \
              python3-git python3-jinja2 libegl1-mesa libsdl1.2-dev xterm locales \
              && apt-get clean \
              && rm -rf /var/lib/apt/lists/*

  RUN groupadd -g 1000 dev \
              && useradd -u 1000 -g dev -d /home/dev dev \
              && mkdir /home/dev \
              && chown -R dev:dev /home/dev

  RUN locale-gen en_US.UTF-8
  ENV LANG en_US.UTF-8
  USER dev

  WORKDIR /home/dev

3. Build image:
  docker build -t pi0-yocto .

4. create run.sh
  docker run -it --name pi0-yocto \
  -v ~/Yocto/pi0_build:/home/dev/yocto \
  --user root \
  pi0-yocto /bin/bash

5. Create folder pi0_build
  mkdir pi0_build
  chmod +x run.sh
  ./run.sh

6. Install lz4 and zstd
  apt update
  apt install -y file lz4 zstd

7. Exit container to exit root because don't use Bitbake as root
  exit
  docker start pi0-yocto
  docker exec -it pi0-yocto /bin/bash

8. clone source
  cd yocto
  git clone -b kirkstone git://git.yoctoproject.org/poky.git
  cd poky
  git clone -b kirkstone https://github.com/agherzan/meta-raspberrypi.git
  source oe-init-build-env
  edit 2 files in build/conf/bblayers.conf and local.conf
    add path to folder meta-raspberrypi in bblayers.conf
    add MACHINE ??= "raspberrypi0-2w"
        IMAGE_FSTYPES ?= "rpi-sdimg"
        ENABLE_UART="1"
        IMAGE_INSTALL:append= " linux-firmware kernel-modules iw wpa-supplicant dhcpcd "
        IMAGE_FEATURES += " ssh-server-dropbear"
        MACHINE_FEATURES:append = " wifi"
        RPI_USE_U_BOOT = "1"

9. build image
  bitbake core-image-minimal

10. image in /build/tmp/deploy/images/raspberrypi0-2w/core-image-minimal-...rootfs.rpi-sdimg

11. Flash image in sdcard by Balena Etcher
  Or dd: sudo dd if=core-image-base-raspberrypi0-wifi.rpi-sdimg of=/dev/sdX bs=4M conv=fsync
    Use the actual device name of your SD card instead of sdX (be careful). For example: sdd
    check device name use: lsblk

12. Boot with UBoot
  Use minicom or screen
  With screen:
    connect uart pi via USB ttl
    plug the USB into your PC
    sudo screen /dev/tty... 115200
    supply power for pi
    pi will boot ...
    user: root
  With minicom:
    connect uart pi via USB ttl
    plug the USB into your PC
    sudo minicom -s
    setting minicom:
      A -> Serial port: /dev/...
      E -> Bps: 115200
      F -> Hardware Flow Control: NO
      G -> Software Flow Control: NO
    Save.
    sudo minicom -b 115200 -D /dev/tty...
    supply power for pi
    pi will boot ...
    user: root
    
13. Connect wifi after boot with UBoot
  vi /etc/wpa_supplicant.conf
  edit 
  {
      ssid="nameOfWifi"
      psk="password"
  }
  wpa_supplicant -B -Dnl80211 -iwlan0 -c /etc/wpa_supplicant.conf
  get ip to connect ssh: udhcpc -i wlan0
