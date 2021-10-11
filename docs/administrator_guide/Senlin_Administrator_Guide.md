# Cài đặt Senlin cli
```
apt install python3-senlinclient
```

# Profiles

## Tạo profile đầu tiên

```
mkdir senlin
cd senlin
```

```
vi cirros_basic.yaml
```

``` 
type: os.nova.server
version: 1.0
properties:
  name: cirros_server
  flavor: 1
  image: "Cirros"
  key_name: mykey
  networks:
    - network: Private01
  metadata:
    test_key: test_value
  user_data: >
    #!/bin/bash
    echo 'hello, world' > /tmp/test_file
```
Trong đó:
  - 1 là id flavor
  - Tên keypair là **mykey**
  - Tên neutron network là **Private01**
  - Tên glance image là Cirros

## Tạo profile object
```
openstack cluster profile create --spec-file /home/ubuntu/senlin/cirros_basic.yaml myserver
```

## Lấy danh sách các profile đã tạo

```
openstack cluster profile list
```

Các tùy chọn khác:
  - **--full-id** : Hiển thị đầy đủ ID trong danh sách
  ```
  openstack cluster profile list --full-id
  ```
  - **--sort** : Sắp xếp danh sách
  ```
  openstack cluster profile list --sort name:desc
  ```
  - **--filte** : Lọc danh sách
  ```
  openstack cluster profile list --filter "type=os.heat.stack-1.0"
  ```
  - **-limit** : Giới hạn danh sách được trả về
  ```
  openstack cluster profile list --limit 1
  ```
  - **--marker** : Bỏ qua các profile này
  ```
  openstack cluster profile list --limit 1 \
    --marker ceda64bd-70b7-4711-9526-77d5d51241c5
  ```

## Hiển thị các thông tin của profile
```
openstack cluster profile show myserver
```

## Cập nhật các thông tin của profile
Lưu ý: Hạn chế thay đổi thông tin profile để giữ tính nhất quán của cụm. Không thể thay đổi thông số kĩ thuật của một profile, cách duy nhất có thể làm là tạo một profile mới.

Thay đổi tên profile:
```
openstack cluster profile update --name new_server myserver
```

Tạo hoặc update metadate:
```
openstack cluster profile update --metadata version=2.2 myserver
```

## Xóa một profile

Chỉ có thể xóa một profile khi không có  cluster hoặc node nào đang tham chiếu đến profile đó.
```
openstack cluster profile delete myserver
```

-----------

# Cluster

## Tạo cluster
```
openstack cluster create --profile myserver mycluster
```

## Thay đổi kích thước cluster

### Chỉ định giá trị chính xác kích thước của cụm mới
```
openstack cluster resize --capacity 4 mycluster
```

### Chỉ định số lượng nút được thêm / bớt
  - Thêm 2 node vào cụm:
  ```
  openstack cluster resize --adjustment 2 mycluster
  ```
  - Xóa 2 node khỏi cụm:
   ```
  openstack cluster resize --adjustment -2 mycluster
  ```

### Thay đổi kích thước cụm theo %
  - Tăng kích thước cụm lên 30%
  ```
  openstack cluster resize --percentage 30 mycluster
  ```
  - Giảm kích thước cụm đi 30%
  ```
  openstack cluster resize --percentage -30 mycluster
  ```

### Thêm node cho cluster
```
openstack cluster expand mycluster
```
```
openstack cluster expand --count 2 mycluster
```

### Xóa node khỏi cluster
```
openstack cluster shrink mycluster
```
```
openstack cluster shrink --count 2 mycluster
```

## Kiểm tra cluster
```
openstack cluster check mycluster
```

## Khôi phục một cluster
Thao tác khôi phục sẽ xóa các nút khỏi cụm được chỉ định và tạo lại nó.
```
openstack cluster recover mycluster --check true
```

## Xóa một cluster
```
openstack cluster delete mycluster
```

----
# Policies
## Tạo Policy
Ví dụ deletion policy
```
vi delete_policy.yaml
```
```
type: senlin.policy.deletion
version: 1.0
description: A policy for choosing victim node(s) from a cluster for deletion.
properties:
  # The valid values include:
  # OLDEST_FIRST, OLDEST_PROFILE_FIRST, YOUNGEST_FIRST, RANDOM
  criteria: OLDEST_FIRST

  # Whether deleted node should be destroyed 
  destroy_after_deletion: True

  # Length in number of seconds before the actual deletion happens
  # This param buys an instance some time before deletion
  grace_period: 60

  # Whether the deletion will reduce the desired capacity of
  # the cluster as well
  reduce_desired_capacity: False
```


### Tạo policy object
```
openstack cluster policy create --spec-file /home/ubuntu/senlin/delete_policy.yaml dp01
```

## Gắn policy vào cluster
```
openstack cluster policy attach --policy dp01 mycluster
```
Kiểm tra policy đã gắn vào
```
openstack cluster policy binding list mycluster
```
```
openstack cluster policy binding show --policy dp01 mycluster
```
##  Xác minh policy đã hoạt động
```
openstack cluster members list mycluster
openstack cluster expand mycluster
openstack cluster members list mycluster
openstack cluster shrink mycluster
openstack cluster members list mycluster
```
Sau khi hoàn thành hoạt động scale-in, bạn sẽ thấy rằng nút cũ nhất của ​​cụm đã bị xóa sau khi chạy lệnh 60s.

-------

# Receiver

## Tạo webhook receiver
```
openstack cluster receiver create --cluster mycluster --action CLUSTER_SCALE_OUT --type webhook w_scale_out
```
```
openstack cluster receiver create --cluster mycluster --action CLUSTER_SCALE_IN --type webhook w_scale_in
```

## Kiểm tra danh sách các receiver
```
openstack cluster receiver list
```

## Kích hoạt receiver với CURL
w_scale_in
```
curl -X POST http://172.31.0.11:8778/v1/webhooks/91671d92-b2cc-4e90-90a9-3c9f40675f1d/trigger?V=2
```
w_scale_out
```
curl -X POST http://172.31.0.11:8778/v1/webhooks/9e450736-7352-4f97-9309-bbac0a261600/trigger?V=2
```

------
# Action

## Danh sách các action
  - CLUSTER_CREATE: Tạo một cụm
  - CLUSTER_DELETE: Xóa một cụm
  - CLUSTER_UPDATE: Cập nhật một cụm
  - CLUSTER_ADD_NODES: Thêm các node hiện có vào 1 cụm
  - CLUSTER_DEL_NODES: Xóa các node khỏi cụm
  - CLUSTER_REPLACE_NODES: Thay thế các node trong một cụm
  - CLUSTER_RESIZE: Điều chỉnh kích thước của cụm
  - CLUSTER_SCALE_IN: Thu nhỏ kích thước bằng cách xóa node khỏi cụm
  - CLUSTER_SCALE_OUT: Mở rộng kích thước bằng cachs thêm node vào cụm
  - CLUSTER_ATTACH_POLICY: Gắn policy vào cụm
  - CLUSTER_DETACH_POLICY: Tách policy khỏi cụm
  - CLUSTER_UPDATE_POLICY: Cập nhật các thuộc tính ràng buộc giữa một cụm và policy
  - CLUSTER_CHECK: Kiểm tra một cụm và chạy NODE_CHECK cho tất cả các node
  - CLUSTER_RECOVER: Khôi phục một cụm và chạy NODE_RECOVER cho tất cả các node ở trạng thái ERROR
  - NODE_CREATE: Tạo một node mới
  - NODE_DELETE: Xóa node đã có
  - NODE_UPDATE: Cập nhật các thuộc tính của node hiện có
  - NODE_JOIN: Thêm một node vào cụm hiện có
  - NODE_LEAVE: Tách một node khỏi cụm hiện có
  - NODE_CHECK: Kiểm tra một nút để xem liệu nút vật lý của nó có phải là "ACTIVE" hay không và cập nhật trạng thái của nó với "ERROR" nếu không
  - NODE_RECOVER: Khôi phục một node
























