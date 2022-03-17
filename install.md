# Cài đặt triểu khai Ansible trên Ubuntu 20.04
## 1. Chuẩn bị
### 1.1 Mô hình triển khai

![image](image/Screenshot_1.png)

### 1.2 IP Planning 

Hostname|	hardware|	Interface|	Disk 
---|---|---|---
ansible	|2 CPU - 2GB RAM|	enp1s0: 192.168.70.50|	30GB
node1	|2 CPU - 2GB RAM	|enp1s0: 192.168.70.51 |30GB
node2|2 CPU - 2GB RAM	|enp1s0: 192.168.70.52	|30GB



## 2. Cài đặt 
### 2.1 Chuẩn bị

- **Cập nhật gói, update OS**
```
apt-get update -y 
apt-get upgrade -y 
apt-get dist-upgrade -y
apt-get autoremove 
```

- **Disable firewalld**
```
systemctl disable ufw
systemctl stop ufw
systemctl status ufw
```

### 2.2 Cài đặt 

- **Khởi tạo user cho dịch vụ Ansible**(2 node)
  - Tạo 1 user Ansible với quyền admin sử dụng cho mục đích kết nối đến các node trong cụm
```
adduser ansible
usermod -aG sudo ansible
```

- **Add file hosts**(3 node)
  - Thêm tên vào file hosts để các server có thể nhận dạng nhau 

```
echo "192.168.70.50 ansible" >> /etc/hosts
echo "192.168.70.51 node1" >> /etc/hosts
echo "192.168.70.52 node2" >> /etc/hosts
```


- **Thực hiện thiết lập SSH key**
  - Đứng tại node AnsibleServer tạo SSH Key, sau đó copy các sang node còn lại. Mục tiêu là sử dụng keypair để không phải nhập password mỗi khi đăng nhập vào các client.

  ```
  ssh-keygen
  ```
  ![image](image/Screenshot_2.png)

  - Thực hiện copy file key sang các node còn lại

``` 
ssh-copy-id ansible@node2
ssh-copy-id ansible@node3
```
  ![image](image/Screenshot_3.png)


### 2.2 Cài đặt Ansible
- Thực hiên cài đặt Ansible trên 1 node có vai trò quản lý điều khiển tất cả các node còn lại, đó được gọi là node controller.
- Ở đây là ansible

```
apt install ansible -y
```


### 2.3 Khai báo file inventory
- Mặc định thì danh sách các host mà AnsibleServer điều khiển sẽ nằm ở file /etc/ansible/host.
- Tạo một file backup 
```
cp /etc/ansible/hosts /etc/ansible/hosts.bak
```

- Thực hiện thêm thông tin danh sách các node trong bài lab vào file /etc/ansible/hosts có nội dung:
```
[hosts1]
192.168.70.51
[hosts2]
192.168.70.52
```

- Kiểm tra danh sách các node đã được khai báo trong file inventory
```
ansible all --list-hosts
```

  ![image](image/Screenshot_4.png)
- Trong trường hợp cần kiểm tra host trong các group ví dụ group: hosts1
```
ansible hosts1 --list-host
```
  ![image](image/Screenshot_4.png)

- Ở phiên bản Ansible 2.0 trở lên file /etc/ansible/hosts có thể thực hiện bổ sụng thêm 1 số tùy chọn về user và password
- Nội dung sau khi chỉnh sửa:
```
[hosts1]
node1 ansible_host=192.168.70.51 ansible_port=22 ansible_user=root
[hosts2]
node2 ansible_host=192.168.70.52 ansible_port=22 ansible_user=root
```

- Trong đó:

  - node1: hostname của node
  - ansible_host: Địa chỉ IP tương ứng với hostname của node
  - ansible_port: Port SSH được sử dụng trên node tương ứng
  - ansible_user: User mà node controller sử dụng để tương tác với các node còn lại thông qua dịch vụ ssh


- Kiểm tra
  - Kiểm tra khai báo thông tin node chính xác hay chưa sử dụng câu lệnh ansible với các tùy chọn all là tất cả các node được khai báo, -m ping thể hiện sử dụng module ping để kiểm tra kết nối đến các node.
``` 
ansible all -m ping
```
  ![image](image/Screenshot_6.png)

