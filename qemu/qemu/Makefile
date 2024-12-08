all: run

KERNEL = vmlinuz
INITRD = initramfs.cpio.gz
DISK   = disk.qcow2

initramfs:
	cp /usr/lib/modules/5.4.0-152-generic/kernel/drivers/net/ethernet/intel/e1000/e1000.ko initramfs
	cd initramfs && \
		find . -print0 \
		| cpio --null -ov --format=newc \
		| gzip -9 > ../initramfs.cpio.gz

${KERNEL}:
	sudo cp -L /boot/vmlinuz ./vmlinuz && \
	sudo chmod o+r ./vmlinuz

run: ${KERNEL} initramfs
	qemu-system-x86_64 \
		-serial mon:stdio \
		-hda ${DISK} \
		-kernel ${KERNEL} \
		-initrd ${INITRD} \
		-netdev user,id=net1 \
		-device e1000,netdev=net1 \
		-append "console=ttyS0" \
		-nographic 
		# -netdev tap,id=net2,ifname=tap0,script=no,downscript=no \
		# -device e1000,id=net2\

mount:
	@sudo modprobe nbd
	@sudo qemu-nbd --connect /dev/nbd0 ${DISK} && sleep 1
	@sudo mount /dev/nbd0 fs

umount:
	@sudo umount fs || exit -1
	@sudo qemu-nbd --disconnect /dev/nbd0 && sleep 1
	@sudo rmmod nbd
	
.PHONY: initramfs run
