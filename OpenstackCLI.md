# Openstack CLI cheat sheet
- Tài liệu này ghi chép về một số command trong Openstack
- Các lệnh được liệt kê theo từng Service 

----------

# CLI

## Keystone
- Tạo User **User1**
```
openstack user create --domain default --project trade --description "Developer" --email email@example.com --password openstack User1
```

- Thêm User **User1** vào Project **Project1** với role **member**
```
openstack role add --user User1 --project Project1 _member_
```

- Tạo **my-corp** Domain
``` 
openstack domain create my-corp
```

- Thêm user **User2** vào domain **my-corp**
```
openstack user create --domain my-corp User2 --password PASSWORD
```

- Thêm Project **development** vào domain **my-corp**
```
openstack project create --domain my-corp development
```

- Thêm group **interns** vào domain **my-corp**
```
openstack group create --domain my-corp interns
```

- Thêm user **User2** vào group my-corp
```
openstack group add user --group-domain my-corp interns User2
```

- Thêm group **interns** vào project **development**
```
openstack role add --project-domain acme-corp --project development --group interns _member_
```

- Thêm  Service mới tên trove với kiểu **database** 
```
openstack service create --name trove --description "OpenStack Trove" database
```

- Tạo Endpoint cho kiểu **database** với region RegionOne
```
openstack endpoint create --region RegionOne database public http://192.168.56.56:8779/v1.0/
openstack endpoint create --region RegionOne database internal http://192.168.56.56:8779/v1.0/
openstack endpoint create --region RegionOne database admin http://192.168.56.56:8779/v1.0/
```

- Chỉnh sửa Quota

Lấy danh sách các project
```
openstack project list
```
Gắn Quota cho project
```
Openstack quota set --instances 20 ID_Project
```

## Glance
- Tạo image **cirros** 
```
openstack image create --file cirros-0.3.5-x86_64-disk.img --disk-format qcow2 --min-disk 1 --min-ram 512 --property description='Cirros Cloud Image' cirros
```

- Tải xuống image **cirros**
```
openstack image save --file /tmp/downloadedimage.img cirros
```

- Chia sẻ image **cirros** với  **Project1**
```
openstack image add project cirros ID_Project1
```
## Nova
- Tạo Flavor **flavor1**
```
openstack flavor create --id 200 --vcpus 1 --ram 512 --disk 1 "flavor1"
```

- Khởi chạy instance **instance1**
```
openstack server create --image cirros --flavor m1.tiny --key-name my-keypair instance1
```

- Tạo snapshot từ **instance1** (Shutdown instance trước khi thực hiện)
```
openstack server image create --name instance1-snapshot instance1
```

- Start/Stop/Suspend/Resume Instance **instance1**
```
nova stop instance1
nova start instance1
nova suspend instance1
nova resume instance1
```

## Neutron
-Tạo Network Private
```
openstack network create tenant-network1
openstack subnet create --network tenant-network1 --subnet-range  192.168.1.0/24 tenant-subnet1
```

- Tạo Network Provider
```
openstack network create --provider-physical-network public --provider-network-type flat provider-network --external
openstack subnet create --network provider-network --subnet-range 172.16.1.0/24 --gateway 172.16.1.1 --no-dhcp provider-subnet
```

- Tạo security group
```
openstack security group create sg1
```

- Tạo rule security group
```
openstack security group rule create --protocol tcp --ingress --dst-port 22 --src-ip 0.0.0.0/0 sg1
```

- Tạo router và assign network private
```
openstack router create router1
openstack router add subnet router1 tenant-subnet1
```

- Assign network provider với  router1
```
neutron router-gateway-set router1 provider-network
```

- Tạo floating ip
```
openstack floating ip create --floating-ip-address 172.16.1.100 provider-network
```

- Assign floating ip
```
openstack server add floating ip instance1 172.16.1.100
```

## Cinder
- Tạo volume
```
openstack volume create --size 1 volume1 --description "Exam volume"
```

- Manage Volume
```
Sudo -i
Fdisk -l
Fdisk /dev/vdb
Mkfs.ext3 /dev/vdb1
Df -h
Mount /dev/vdb1 /mnt
Umount /dev/vdb1 /mnt
```

- Attach volume volume1 vào instance instance1
```
openstack server add volume instance1 volume1
```

- Tạo snapshot volume cho volume1
```
openstack snapshot create --name volume1-snapshot volume1
```

- Tạo volume từ snapshot
```
openstack volume create --snapshot volume1-snapshot --size 1 restored-snapshot-volume1
```

## Swift
- Tạo container
```
swift post container1
```

- Tải file lên container
```
swift upload container1 object1.txt
```

- Xóa file từ  container
```
swift delete container1 object1.txt
```

## Heat
- Tạo stack stack1
```
openstack stack create -t test-stack.yaml --parameter Flavor=m1.tiny --parameter Image=<image-uuid> --parameter Net=<network-uuid> --parameter
ServerName=stackserver1 --parameter VolumeName=stackvolume1 stack1
```

- Cập nhật stack bằng cách thay thế serverName
```
openstack stack update --existing --parameter
ServerName=stackserverupdate stack1
```

- Xóa stack
```
Openstack stack delete <stack-name>
```
