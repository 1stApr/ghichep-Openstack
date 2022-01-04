# Cài đặt Openstack Với Kolla-Ansible
- Tài liệu này hướng dẫn triển khai cài đặt Openstack trên các máy chủ Ubuntu Server 20.04 LTS
- Ví dụ trong tài liệu này được thực hiện trên hệ thống gồm 13 máy chủ: 
  - 1 máy chủ Operator 
  - 3 máy chủ Controller
  - 3 máy chủ Compute
  - 3 máy chủ Block storage
  - 3 máy chủ Object storage
 

## Tài liệu tham khảo

[Ansible](https://docs.ansible.com/)

[Docker](https://docs.docker.com/)

[Openstack Victoria Deployment](https://docs.openstack.org/victoria/deploy/index.html)

[Window Server Image](https://1drv.ms/u/s!Ah2QX3LCAkYQi5B4zGVbbQm6u3kAJA?e=fvtate)

## 1 Cài đặt hệ điều hành

** Openstack hỗ trợ các hệ điều hành sau:**

-   Ubuntu Server
-   RHEL/CentOS
-   Debian

Khuyến khích sử dụng Ubuntu Server 20.04 LTS

### 1.1 Yêu cầu phần cứng

Để hoạt động ổn định cần tối thiểu 3 máy chủ đáp ứng các yêu cầu phần cứng sau:

|     |		Tối thiểu	     |	  Khuyến cáo    |  
|----------|---------------------|-------------------|
| CPU | 2 CPU  6 core @ 2.4 GHz | 4CPU 6 core @ 2.67 GHz  |
|RAM|32 GB|96 GB |
|DISK|HDD: 1 Disk SAS 500GB (7200 rpm) | HDD: 3 Disk SAS 500GB (7200 rpm) |

Hoặc các cấu hình tương tự khác:
 -   1x SSD 500+ GB
 -   1x HDD (7200 rpm) 500+ GB và 1x SSD 250+ GB 
 -   1x HDD (15000 rpm) 500+ GB

### 1.2 Phân vùng ổ đĩa cài hệ điều hành

-  Swap: 32 GB 
-  /boot:  1 GB
-  /boot/efi : 1 GB
-  /: 100 GB
-  /var: Dung lượng ổ cứng còn lại cho hết vào phân vùng /var ( >300GB)

Trên các máy chủ Block Storage và Object Storage ngoài ổ đĩa cài hệ điều hành thì cần thêm ổ đĩa để lưu trữ dữ liệu (Trên Block Storage cần thêm 2 ổ đĩa, trên Object Storage cần thêm 3 ổ đĩa).

### 1.3 Yêu cầu về mạng

Mỗi máy chủ cần có ít nhất 2 giao diện mạng (NICs)( Khuyến khích có 3 hoặc 4 NICs ). Một vài loại mạng cơ bản được dùng:
 -  **Management network**: Được sử dụng để giao tiếp giữa các thành phần của Openstack.
 - **External network**: Được sử dụng để cho các máy ảo truy cập Internet.
 - **Corporate network**: Được sử dụng để cho các máy chủ truy cập Internet.
 - **Storage network**: Được sử dụng để truy cập vào các hệ thống lưu trữ bên ngoài.

### 1.4 Tham khảo cấu hình mạng trong file netplan:

```
# This is the network config written by 'subiquity'
network:
  ethernets:
    eno1: {}
    eno2: {}
    eno3:
      addresses: [10.10.170.200/24]
      gateway4: 10.10.170.1
      nameservers:
        addresses:
        - 8.8.8.8
        - 1.1.1.1
    eno4:
      addresses: [10.10.24.200/24]
      nameservers:
        addresses:
        - 8.8.8.8
        - 1.1.1.1
  bonds:
    bond0:
      interfaces: [eno1,eno2]
      parameters:
        mode: 802.3ad
  version: 2

```

**Sau khi cài đặt xong hệ điều hành và mạng cần chú ý:**

- Các đường mạng: 
  - eno3: Sử dụng để cho các máy chủ truy cập Internet và để ssh vào các máy chủ.
  - eno4: Sử dụng để giao tiếp giữa các thành phần của Openstack
  - bonds: Sử dụng để cho các máy ảo trên Openstack truy cập Internet

  **Nếu các máy chủ lưu trữ dữ liệu để riêng thì các máy chủ này không cần đường bonds**.

  Kiểm tra các máy chủ có kết nối Internet hay chưa:

  ```
  ping -c 4 8.8.8.8 -I eno1
  ```

- Xem các phân vùng ổ đĩa:

  ```
  lsblk
  ```

----------

## 2. Cài đặt một số package cần thiết

### 2.1 Trên tất cả các máy chủ

```
sudo apt update
```

Cài đặt SSH
```
apt install ssh 
```

Chỉnh  timezone

```
timedatectl set-timezone Asia/Ho_Chi_Minh
```

----------

### 2.2 Trên các máy chủ Block Storage 

** Tạo phân vùng Cinder**

Ví dụ phân vùng cho Cinder là /dev/sdb và /dev/sdc:

⚠️ Tất cả dữ liệu trên phân vùng /dev/sdb và /dev/sdc sẽ bị mất!

```
apt install lvm2
```

```
pvcreate /dev/sdb /dev/sdc
vgcreate cinder-volumes /dev/sdb /dev/sdc
```

### 2.3 Trên các máy chủ Object Storage

Thêm dòng **net.ipv4.ip_forward=1** vào **/etc/sysctl.conf**

```
vi /etc/sysctl.conf
```

```
net.ipv4.ip_forward=1
```

Tải lại các cài đặt:

```
sudo sysctl -p
```

Chuẩn bị ổ đĩa để làm thiết bị lưu trữ Swift

⚠️ Tất cả dữ liệu trên phân vùng sdc, sdd và sde sẽ bị mất!

```
index=0
for d in sdc sdd sde; do
    parted /dev/${d} -s -- mklabel gpt mkpart KOLLA_SWIFT_DATA 1 -1
    sudo mkfs.xfs -f -L d${index} /dev/${d}
    (( index++ ))
done
```

----------

### 2.4 Trên các máy chủ Operator node

#### 2.4.1 Cài đặt sshpass

```
apt install sshpass
```

Tạo khóa ssh
```
ssh-keygen -t rsa -N '' -f ~/.ssh/id_rsa
```

#### 2.4.2 Thêm các máy chủ vào file /etc/hosts

```
vi /etc/hosts
```

Ví dụ:

```
10.10.24.201 control01
10.10.24.202 control02
10.10.24.203 control03

10.10.24.211 compute01
10.10.24.212 compute02
10.10.24.213 compute03

10.10.24.221 storage01
10.10.24.222 storage02
10.10.24.223 storage03
```

#### 2.4.3 Cài đặt các package

Cài đặt các Python và các thư viện phụ thuộc

```
sudo apt install python3-dev libffi-dev gcc libssl-dev -y
```

Cài đặt pip

```
sudo apt install python3-pip -y 
```

Đảm bảo phiên bản mới nhất của pip được cài đặt

```
sudo pip3 install -U pip
```

Cài đặt Ansible

```
pip3 install "ansible==2.10.7"
```

Cài đặt Kolla-ansible

```
sudo pip3 install kolla-ansible
```

Tạo thư mục /etc/kolla 

```
sudo mkdir -p /etc/kolla
sudo chown $USER:$USER /etc/kolla
```

Copy các file globals.yml, passwords.yml vào thư mục /etc/kolla 

```
cp -r /usr/local/share/kolla-ansible/etc_examples/kolla/* /etc/kolla
```

Copy all-in-one và multinode files 

```
cp /usr/local/share/kolla-ansible/ansible/inventory/* .
```

#### 2.4.4 Configure Ansible

Thêm các dòng sau vào trong file ansible.cfg

```
vi /etc/ansible/ansible.cfg
```

```
[defaults]
host_key_checking=False
pipelining=True
forks=100
```

#### 2.4.5 Chỉnh sửa file multinode

```
vi multinode
```

```
[all:vars]
ansible_connection=ssh
ansible_become=true
ansible_ssh_port=22
ansible_ssh_user=root
ansible_ssh_pass=SSH_PASS
ansible_sudo_pass=SUDO_PASS

[control]
control01
control02
control03

[network]
control01
control02
control03

[monitoring]
control01
control02
control03

[storage]
storage01
storage02
storage03

[compute]
compute01
compute02
compute03

[deployment]
localhost  ansible_connection=local

```

Kiểm tra cấu hình multinode có đúng hay không

```
ansible -i multinode all -m ping
```

Chạy trình tạo mật khẩu ngẫu nhiên

```
kolla-genpwd
```

#### 2.4.6 Chỉnh sửa file cấu hình chính cho Kolla-Ansible

```
vi /etc/kolla/globals.yml
```

```
kolla_base_distro: "ubuntu"
kolla_install_type: "source"
openstack_release: "wallaby"
neutron_bridge_name: "br-ex"
network_interface: "eno2"
neutron_external_interface: "bond0"
kolla_internal_vip_address: "10.10.24.250"
enable_chrony: "yes"
enable_neutron_provider_networks: "yes"
enable_haproxy: "yes"
nova_compute_virt_type: "kvm"

enable_cinder: "yes"
enable_cinder_backend_lvm: "yes"
enable_cinder_backup: "no"

enable_prometheus: "yes"
enable_grafana: "yes"

enable_heat: "yes"
enable_senlin: "yes"

enable_swift : "yes"
enable_swift_s3api: "yes"
kolla_external_vip_interface: "{{ network_interface }}"
api_interface: "{{ network_interface }}"
storage_interface: "{{ network_interface }}"
swift_storage_interface: "{{ storage_interface }}"
swift_replication_interface: "{{ swift_storage_interface }}"
glance_backend_swift: "yes"
```

## 3 Triển khai cài đặt

Thực hiện trên máy chủ Operator node

### 3.1 Bootstrap servers với kolla-ansible

```
kolla-ansible -i multinode bootstrap-servers
```

### 3.2 Cấu hình cho Swift

#### 3.2.1 Chuẩn bị tạo Ring

```
STORAGE_NODES=(10.10.24.31 10.10.24.32 10.10.24.33)
KOLLA_SWIFT_BASE_IMAGE="kolla/ubuntu-source-swift-base:wallaby"
mkdir -p /etc/kolla/config/swift
```

#### 3.2.2 Tạo Object Ring

```
docker run \
  --rm \
  -v /etc/kolla/config/swift/:/etc/kolla/config/swift/ \
  $KOLLA_SWIFT_BASE_IMAGE \
  swift-ring-builder \
    /etc/kolla/config/swift/object.builder create 10 3 1
```

```
for node in ${STORAGE_NODES[@]}; do
    for i in {0..2}; do
      docker run \
        --rm \
        -v /etc/kolla/config/swift/:/etc/kolla/config/swift/ \
        $KOLLA_SWIFT_BASE_IMAGE \
        swift-ring-builder \
          /etc/kolla/config/swift/object.builder add r1z1-${node}:6000/d${i} 1;
    done
done
```

#### 3.2.3 Tạo Account Ring

```
docker run \
  --rm \
  -v /etc/kolla/config/swift/:/etc/kolla/config/swift/ \
  $KOLLA_SWIFT_BASE_IMAGE \
  swift-ring-builder \
    /etc/kolla/config/swift/account.builder create 10 3 1
```

```
for node in ${STORAGE_NODES[@]}; do
    for i in {0..2}; do
      docker run \
        --rm \
        -v /etc/kolla/config/swift/:/etc/kolla/config/swift/ \
        $KOLLA_SWIFT_BASE_IMAGE \
        swift-ring-builder \
          /etc/kolla/config/swift/account.builder add r1z1-${node}:6001/d${i} 1;
    done
done
```

#### 3.2.4 Tạo Container Ring

```
docker run \
  --rm \
  -v /etc/kolla/config/swift/:/etc/kolla/config/swift/ \
  $KOLLA_SWIFT_BASE_IMAGE \
  swift-ring-builder \
    /etc/kolla/config/swift/container.builder create 10 3 1
```

```
for node in ${STORAGE_NODES[@]}; do
    for i in {0..2}; do
      docker run \
        --rm \
        -v /etc/kolla/config/swift/:/etc/kolla/config/swift/ \
        $KOLLA_SWIFT_BASE_IMAGE \
        swift-ring-builder \
          /etc/kolla/config/swift/container.builder add r1z1-${node}:6002/d${i} 1;
    done
done
```

#### 3.2.5 Cân bằng lại các file Ring

```
for ring in object account container; do
  docker run \
    --rm \
    -v /etc/kolla/config/swift/:/etc/kolla/config/swift/ \
    $KOLLA_SWIFT_BASE_IMAGE \
    swift-ring-builder \
      /etc/kolla/config/swift/${ring}.builder rebalance;
done
```

### 3.2 Thực hiện kiểm tra trước khi triển khai cho các máy chủ:

```
kolla-ansible -i multinode prechecks
```

### 3.3 Tải về các Docker image:

```
kolla-ansible -i multinode pull
```

### 3.4 Triển khai cài đặt

Tắt openvswitch nếu có

```
systemctl | grep "openvswitch"
systemctl stop openvswitch-switch
```

```
kolla-ansible -i multinode deploy
```

--------------------

## 4 Sử dụng OpenStack

Trên máy chủ controller

### 4.1 Cài OpenStack CLI client:

```
apt install python3-openstackclient -y
```

Tạo file admin-openrc:

```
kolla-ansible post-deploy
```

```
. /etc/kolla/admin-openrc.sh
cp /etc/kolla/admin-openrc.sh admin-openrc.sh
```

### 4.2 Lấy người dùng quản trị và tenant IDs

```
ADMIN_USER_ID=$(openstack user list | awk '/ admin / {print $2}')
ADMIN_PROJECT_ID=$(openstack project list | awk '/ admin / {print $2}')
ADMIN_SEC_GROUP=$(openstack security group list --project ${ADMIN_PROJECT_ID} | awk '/ default / {print $2}')
```

### 4.3 Cấu hình Sec Group

```
openstack security group rule create --ingress --ethertype IPv4 \
   --protocol icmp ${ADMIN_SEC_GROUP}
openstack security group rule create --ingress --ethertype IPv4 \
   --protocol tcp --dst-port 22 ${ADMIN_SEC_GROUP}
openstack security group rule create --ingress --ethertype IPv4 \
   --protocol tcp --dst-port 8000 ${ADMIN_SEC_GROUP}
openstack security group rule create --ingress --ethertype IPv4 \
   --protocol tcp --dst-port 8080 ${ADMIN_SEC_GROUP}
 ```
 
### 4.4 Cấu hình nova public-key và quotas

```
openstack keypair create --public-key ~/.ssh/id_rsa.pub mykey
```

```
openstack quota set --instances 40 ${ADMIN_PROJECT_ID}
openstack quota set --cores 40 ${ADMIN_PROJECT_ID}
openstack quota set --ram 96000 ${ADMIN_PROJECT_ID}
```

```
openstack flavor create --id 1 --ram 512 --disk 1 --vcpus 1 m1.tiny
openstack flavor create --id 2 --ram 2048 --disk 20 --vcpus 1 m1.small
openstack flavor create --id 3 --ram 4096 --disk 40 --vcpus 2 m1.medium
openstack flavor create --id 4 --ram 8192 --disk 80 --vcpus 4 m1.large
openstack flavor create --id 5 --ram 16384 --disk 160 --vcpus 8 m1.xlarge

openstack flavor create --id 6 --ram 512 --disk 0 --vcpus 1 t2.nano
openstack flavor create --id 7 --ram 1024 --disk 0 --vcpus 1 t2.micro
openstack flavor create --id 8 --ram 2048 --disk 0 --vcpus 1 a1.medium
openstack flavor create --id 9 --ram 4096 --disk 0 --vcpus 2 a1.large
openstack flavor create --id 10 --ram 8192 --disk 0 --vcpus 4 a1.xlarge

openstack flavor create --id 11 --ram 16384 --disk 0 --vcpus  2 r5.large
openstack flavor create --id 12 --ram 31768 --disk 0 --vcpus  4 r5.xlarge
```

### 4.5 Tải lên Openstack image cirros

```
wget http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img
```

```
apt install python3-glanceclient -y
```

```
glance image-create --name "cirros" \
  --file cirros-0.4.0-x86_64-disk.img \
  --disk-format qcow2 --container-format bare \
  --visibility=public
```
### 4.6 Cài LoadBalancer 

Thêm các dòng sau vào file globals.yml

```
vi /etc/kolla/global.yml
```
```
enable_octavia: "yes"

octavia_certs_country: US
octavia_certs_state: Oregon
octavia_certs_organization: OpenStack
octavia_certs_organizational_unit: Octavia
octavia_network_interface: "enp0s9"
octavia_auto_configure: no
enable_horizon_octavia: "{{ enable_octavia | bool }}"
```

Tạo chứng chỉ Octavia

```
kolla-ansible octavia-certificates
```

Cài đặt Octavia với Kolla-Ansible

```
kolla-ansible -i multinode deploy --tags common,horizon,octavia
```

### 4.7 Mẫu file globals.yml

```
prechecks_enable_host_ntp_checks: "false"
kolla_base_distro: "ubuntu"
kolla_install_type: "source"
openstack_release: "xena"
neutron_bridge_name: "br-ex"
network_interface: "eno2"
neutron_external_interface: "bond0"
kolla_internal_vip_address: "10.10.24.250"
enable_chrony: "yes"
enable_neutron_provider_networks: "yes"
enable_haproxy: "no"
nova_compute_virt_type: "qemu"
enable_cinder: "yes"
enable_cinder_backend_lvm: "yes"
enable_cinder_backup: "no"

enable_grafana: "yes"
enable_prometheus: "yes"

enable_heat: "yes"
enable_trove: "yes"
enable_senlin: "yes"
enable_octavia: "yes"
enable_designate: "yes"
enable_cloudkitty: "yes"


enable_swift : "yes"
enable_swift_s3api: "yes"

api_interface: "{{ network_interface }}"
storage_interface: "{{ network_interface }}"
swift_storage_interface: "{{ storage_interface }}"
kolla_external_vip_interface: "{{ network_interface }}"
swift_replication_interface: "{{ swift_storage_interface }}"

octavia_certs_country: US
octavia_certs_state: Oregon
octavia_certs_organization: OpenStack
octavia_certs_organizational_unit: Octavia
octavia_network_interface: "enp0s9"

octavia_auto_configure: no

enable_horizon_heat: "{{ enable_heat | bool }}"
enable_horizon_trove: "{{ enable_trove | bool }}"
enable_horizon_senlin: "{{ enable_senlin | bool }}"
enable_horizon_octavia: "{{ enable_octavia | bool }}"

```


## 5 Các lỗi hay gặp và lưu ý khác

**Không thể chạy lệnh apt update?**

  - Thêm **nameserver 8.8.8.8** vào **/etc/resolv.conf**

**Không thể tải image prometheus**

  - Kiểm tra timezone

  ```
  date
  ```

**Lỗi Container openvswitch_db restart**

```
systemctl | grep "openvswitch"
systemctl stop openvswitch-switch
```

**Thêm SAN storage**

  - install open-iscsi

  ```
  apt install open-iscsi
  systemctl start iscsid
  systemctl start iscsid.socket
  systemctl start open-iscsi
  ```

**TASK [service-rabbitmq : nova | Ensure RabbitMQ users exist] failed**

  - Delete Docker container rabbitmq
  - Stop iscsid

  ```
  systemctl stop iscsid
  systemctl stop iscsid.socket
  ```

**Lỗi khi đặt enable_haproxy: "yes" trong file globals**

  - Xóa Docker container mariadb mariadb_clustercheck trước khi deploy

**Khi đặt kolla_internal_vip_address giống ip controller**

  - Đặt enable_haproxy: "no" trong file globals

**Đặt lại mật khẩu admin Grafana**

```
docker exec -u0 -it grafana bash
grafana-cli admin reset-admin-password <NEW_PASS>
```

**Không thể tải Docker images khi dùng proxy**

  - Tạo thư mục cho Docker service

  ```
  mkdir /etc/systemd/system/docker.service.d
  ```

  - Tạo file http-proxy.conf và thêm các biến HTTP_PROXY:

  ```
  touch /etc/systemd/system/docker.service.d/http-proxy.conf
  ```

  ```
  vi /etc/systemd/system/docker.service.d/http-proxy.conf
  ```

  ```
  [Service]
  Environment="HTTP_PROXY=http://10.61.11.42:3128/"
  Environment="HTTPS_PROXY=http://10.61.11.42:3128/"
  ```

  - Tải lại thay đổi:

  ```
  sudo systemctl daemon-reload
  ```

  - Khởi động lại Docker:

  ```
  sudo systemctl restart docker
  ```

  - Xem cấu hình proxy đúng hay chưa:

  ```
  sudo systemctl show --property Environment docker
  ```

**Lấy mật khẩu truy cập Horizon Dashboard**

```
grep "keystone_admin" /etc/kolla/passwords.yml
```

**Lấy mật khẩu truy cập Grafana Dashboard**

```
grep "grafana_admin" /etc/kolla/passwords.yml
```

**Deploy thêm service**

Ví dụ deploy thêm Octavia
```
kolla-ansible -i multinode deploy --tags common,horizon,octavia
```

**Kiểm tra lại cấu hình**
```
cat /etc/kolla/globals.yml | egrep -v '^#|^$'
```

----
