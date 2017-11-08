# os-bootstrap
Bootstrapping Linux OSes, specifically Fedora and Ubuntu


## Example Build

Notes:
  * Disk controller and disk options were chosen as disks sitting on LVM with SSD drives and trim support was desired.

### Ubuntu VM

``` sh
VM_NAME="${VM_NAME:-vm-default}"
qemu-img create -f qcow2 /var/lib/libvirt/images/${VM_NAME}.qcow2 8G

virt-install --connect qemu:///system --virt-type kvm --accelerate \
  --name ${VM_NAME} \
  --os-type=linux --os-variant ubuntu16.04 \
  --ram 1024 --vcpus 2 \
  --controller scsi,model=virtio-scsi \
  --disk path=/var/lib/libvirt/images/${VM_NAME}.qcow2,format=qcow2,bus=scsi,cache=writethrough,discard=unmap \
  --network network=default,model=virtio \
  --console pty,target_type=serial \
  --watchdog default \
  --memballoon virtio \
  --autostart \
  --location http://archive.ubuntu.com/ubuntu/dists/xenial/main/installer-amd64/ \
  --extra-args 'console=ttyS0,115200n8 serial auto=true priority=critical debian-installer/allow_unauthenticated_ssl=true
    url=https://raw.githubusercontent.com/digitalr00ts/os-bootstrap/master/ubuntu/xenial.cfg'

# Have not found a way to enable autodeflate w/ virt-install
sed -i'' "s/<memballoon model='virtio'>/<memballoon model='virtio' autodeflate='on'>/" /etc/libvirt/qemu/${VM_NAME}.xml

kvm-nbd -c /dev/nbd0 /var/lib/libvirt/images/${VM_NAME}.qcow2
partprobe /dev/nbd0
mkdir -p /mnt/vm-image/
mount /dev/nbd0p2 /mnt/vm-image/
mount /dev/nbd0p1 /mnt/vm-image/boot
chroot /mnt/vm-image/
#mount --bind /dev/ /mnt/dev
#mount -t proc none /proc
#mount -t sysfs none /sys

ln -s /lib/systemd/system/getty@.service /etc/systemd/system/getty.target.wants/getty@ttyS0.service

sed -i 's/quiet splash//' /etc/default/grub

# https://help.ubuntu.com/community/Installation/LowMemorySystems
echo 'DISABLED_MODULES="ath_hal fc fglrx fwlanusb ltm nv"' >> /etc/default/linux-restricted-modules-common

#disable hibernate
rm /etc/initramfs-tools/conf.d/resume
sudo update-initramfs -u

umount /proc/ /sys/ /dev/
exit

umount -R /mnt
kvm-nbd -d /dev/nbd0
```

### Fedora VM

``` sh
VM_NAME="${VM_NAME:-vm-default}"
qemu-img create -f qcow2 /var/lib/libvirt/images/${VM_NAME}.qcow2 8G

virt-install --connect qemu:///system --virt-type kvm --accelerate \
  --name ${VM_NAME} \
  --os-type=linux --os-variant fedora26 \
  --ram 1024 --vcpus 2 \
  --controller scsi,model=virtio-scsi \
  --disk path=/var/lib/libvirt/images/${VM_NAME}.qcow2,format=qcow2,bus=scsi,cache=writethrough,discard=unmap \
  --network network=default,model=virtio \
  --console pty,target_type=serial \
  --watchdog default \
  --memballoon virtio \
  --autostart \
  --location https://dl.fedoraproject.org/pub/fedora/linux/releases/26/Everything/x86_64/os/ \
  --extra-args 'console=ttyS0, inst.text inst.sshd
    inst.ks=https://raw.githubusercontent.com/digitalr00ts/kickstart/f26/server-core.cfg'

# Add enabling ttyS0
```
