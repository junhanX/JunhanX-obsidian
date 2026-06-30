qemu版本选择 qemu 9.2.4 
```
../configure   --target-list=x86_64-softmmu   --enable-debug   --enable-kvm   --enable-slirp   --enable-tools   --enable-trace-backends=simple   --disable-strip   --disable-gtk   --disable-vnc
```


构建debug内核环境
```
编译内核
编译busybox
和host共享文件夹

busybox中的init准备
静态编译busybox
mkdir -p initramfs/{bin,sbin,etc,proc,sys,dev,usr/bin,usr/sbin}
cd initramfs
wget https://busybox.net/downloads/busybox-1.36.1.tar.bz2
tar -xvf busybox-1.36.1.tar.bz2
cd busybox-1.36.1
make defconfig
make menuconfig
Settings  --->
    [*] Build static binary (no shared libs)
make -j$(nproc)
make install CONFIG_PREFIX=../initramfs
cd ../initramfs

cat > init << 'EOF'
mount -t proc proc /proc
mount -t sysfs sysfs /sys
mount -t devtmpfs devtmpfs /dev

mkdir -p /mnt/host

mount -t 9p \
    -o trans=virtio,version=9p2000.L \
    host0 \
    /mnt/host

exec /bin/sh
EOF

find . -print0 | cpio --null -ov --format=newc | gzip -9 > initramfs.img

启动代码
#!/bin/bash

/nvme-volume/hanzj/Code/qemu/build/qemu-system-x86_64 \
    -m 4G \
    -smp 4 \
    -cpu host \
    -enable-kvm \
    -kernel /nvme-volume/hanzj/Code/linux-6.18.36/arch/x86/boot/bzImage \
    -initrd /nvme-volume/hanzj/Code/debug-kernel-busybox/initramfs/initramfs.img \
    -append "console=ttyS0 rdinit=/init nokaslr loglevel=8 panic=-1" \
    -nographic \
    -serial mon:stdio \
    -virtfs local,path=/nvme-volume/hanzj/Code/debug-kernel-busybox/share,mount_tag=host0,security_model=none,id=host0
```