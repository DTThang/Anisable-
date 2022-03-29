# 1. Giới thiệu 
- Trong ansible, các biến có độ ưu tiên sử dụng khác nhau, dưới đây là danh sách độ ưu tiên của biếu từ thấp đến cao 

1. command line values (for example, -u my_user, these are not variables)
2. role defaults (defined in role/defaults/main.yml)(1)
3. inventory file or script group vars(2)
4. inventory group_vars/all(3)
5. playbook group_vars/all(3)
6. inventory group_vars/*(3)
7. playbook group_vars/*(3)
8. inventory file or script host vars(2)
9. inventory host_vars/*(3)
10. playbook host_vars/*(3)
11. host facts / cached set_facts(4)
12. play vars
13. play vars_prompt
14. play vars_files
15. role vars (defined in role/vars/main.yml)
16. block vars (only for tasks in block)
17. task vars (only for the task)
18. include_vars
19. set_facts / registered vars
20. role (and include_role) params
21. include params
22. extra vars (for example, -e “user=my_user”)(always win precedence)

- Chú thích 
  - (1): Các task trong role xem chủ sở hữu mặc định của role. Các task được xác định bên ngoài của role xem là giá trị mặc định cuối cùng của role 
  - (2): Biến xác định trong file inventory hoặc cung cấp bởi inventory động 
  - (3): Bao gồm các var được thêm bởi 'vars plugins' giống như host_var và group_var được hêm vào bởi vars plugin mặc định đi kèm với ansible
  - (4): Khi được tạo đi kèm với option set_facts’s cacheable, biến có độ ưu tiên cao trong play, nhưng giống như độn ưu tiên của host fact khi chúng đề từ cache
- Nói chung, Ansible ưu tiên cho các biến được xác định gần đây hơn, tích cực hơn và có phạm vi rõ ràng hơn. Các biến trong thư mục mặc định bên trong role  dễ bị ghi đè


# 2. Ví dụ  
- Tạo một playbook.yml  với nội dung:
```
---
- name: writing something
  hosts: node1
  tasks:
    - debug:
            msg="My name is  name1 "
```
- Tạo  một file invertory với nội dung `node1 ansible_host=192.168.70.51 ansible_port=22 ansible_user=root` để chỉ định host 


## 1. command line values

- Là các option đi kèm theo dòng lệnh như   [-h], [--version], [-v], [-b], [--become-method BECOME_METHOD], [-i INVENTORY]
- Nhập lệnh `ansible-playbook -i inventory playbook.yml`
    ![image](image/Screenshot_25.png)

## 2. role defaults

- Các biến mặc định của ansible được đặt tại  role/defaults/main.yml
- Tạo một role tên test bằng ansible-galaxy
  - File roles/tests/tasks/main.yml
```
- debug:
    msg="My name is {{ name1 }}"
```
  - File roles/tests/defaults/main.yml
```
name1: thang
```
  - file playbook.yml
```
---
- name: writing something
  hosts: node1
  roles:
    - tests 
```
- Kết quả 
    ![image](image/Screenshot_26.png)

## 3. inventory file or script group vars
- Tạo file inventory với nội dung
```
[ubuntu]
node1 ansible_host=192.168.70.51 ansible_port=22 ansible_user=root
```


