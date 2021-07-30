# Tạo image Ubuntu Bionic (18.04)
- Tài liệu này hướng dẫn tạo image Ubuntu Bionic cho Glance
- Thực hiện trên máy tính chạy hệ điều hành ubuntu
## Cài đặt các package cần thiết
```
apt install qemu qemu-kvm libvirt-clients libvirt-daemon-system virtinst bridge-utils virt-manager
```
## Kiểm tra xem libvirt network có được bật hay không
```
virsh net-list
```

Nếu libvirt network không được bật thì chạy lệnh sau:
```
virsh net-start default
```
## Tải file iso ubuntu
```
wget -o bionic-mini.iso http://archive.ubuntu.com/ubuntu/dists/bionic/main/installer-amd64/current/images/netboot/mini.iso
```

## Cấp quyền truy cập cho file iso vừa tải xuống
```
chmod 777 bionic-mini.iso
```

## Tạo file qcow2
```
qemu-img create -f qcow2 bionic.qcow2 10G
```

## Cấp quyền truy cập file qcow2 vừa tạo
```
chmod 777 bionic.qcow2
```

## Tạo máy ảo 
```
virt-install --virt-type kvm --name bionic --ram 1024 \
  --cdrom=bionic-mini.iso \
  --disk bionic.qcow2,bus=virtio,size=10,format=qcow2 \
  --network network=default \
  --graphics vnc,listen=0.0.0.0 --noautoconsole \
  --os-type=linux --os-variant=ubuntu18.04
```

## Cài đặt máy ảo
## Sau khi cài xong máy ảo

Cài các package cần thiết trong máy ảo(tùy chọn)
```
apt install git
```

Cài  cloud-init package trong máy ảo
```
apt install cloud-init
```

```
dpkg-reconfigure cloud-init
```

Tắt máy ảo
```
/sbin/shutdown -h now
```

## Dọn dẹp và hoàn tác libvirt domain
```
virt-sysprep -d bionic
virsh undefine bionic
```

# Cách chuyển đổi qua lại các định dạng của image

- Lệnh  **qemu-img convert** có thể chuyển đổi giữa các định dạng sau **qcow2**(KVM, Xen), **qed**(KVM), **raw**, **vdi**(VirtualBox), **vhd**(Hyper-V), và **vmdk**(VMware).
- Định dạng khuyến khích dùng là **qcow2**

## Công thức chung
```
qemu-img convert -f old-format -O new-format image.old-format image.new-format
```

## Ví dụ chuyển image raw thành qcow2
```
qemu-img convert -f raw -O qcow2 image.img image.qcow2
```
## Ví dụ chuyển image vmdk thành raw
```
qemu-img convert -f vmdk -O raw image.vmdk image.img
```
