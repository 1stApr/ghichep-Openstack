# Hướng dẫn tạo image ubuntu cho Openstack
- Hướng dẫn này được thực hiện trên máy chủ Ubuntu Desktop  20.04

## Cài đặt một số package cần thiết

```
sudo apt install qemu qemu-kvm libvirt-clients libvirt-daemon-system virtinst bridge-utils
```
```
apt install virt-manager
```

## Kiểm tra xem mạng libvirt đã được bật hay chưa

Kiểm tra libvirt

```
virsh net-list
```

Bật libvirt nếu chưa bật

```
virsh net-start default
```

## Tải xuống file cài đặt ubuntu

```
wget -o bionic-mini.iso http://archive.ubuntu.com/ubuntu/dists/bionic/main/installer-amd64/current/images/netboot/mini.iso
```

## Tạo file qcow2

```
qemu-img create -f qcow2 bionic.qcow2 10G
```

```
chmod 777 bionic.qcow2
```

## Tạo máy ảo

```
virt-install --virt-type kvm --name bionic --ram 1024 \
  --cdrom=bionic-mini.iso \
  --disk bionic.qcow2,bus=virtio,size=10,format=qcow2 \
  --network network=default \
  --graphics vnc,listen=0.0.0.0 --noautoconsole \
  --os-type=linux --os-variant=ubuntu18.04
```

## Bắt đầu quá trình cài đặt hệ điều hành

## Sau khi cài đặt hệ điều hành

**Thực hiện trong máy ảo vừa cài đặt**

Cài đặt cloud-init package trong máy ảo

```
apt install cloud-init
```

Cài đặt metadata được image sử dụng

```
dpkg-reconfigure cloud-init
```

Tắt máy ảo

```
/sbin/shutdown -h now
```

## Hoàn tất cài đặt

```
virt-sysprep -d bionic
virsh undefine bionic
```
