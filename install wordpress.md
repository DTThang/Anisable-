# 1 Tạo thư mục theo cấu trúc  
- Lệnh 
```
mkdir wordpress
cd wordpress
mkdir roles vars files
cd roles
mkdir -p apache/tasks php/tasks mysql/tasks wordpress/tasks firewall/tasks
```

```
/root/wordpress
├── files
├── roles
│   ├── apache
│   │   └── tasks
│   ├── firewall
│   │   └── tasks
│   ├── mysql
│   │   └── tasks
│   ├── php
│   │   └── tasks
│   └── wordpress
│       └── tasks
└── vars

```


# 2. Tạo biến ansible
- Xác định một biến để lưu trữ thông tin về những thứ phổ biến bao gồm, người dùng MySQL, máy chủ Apache, Cổng, Mật khẩu, phần mở rộng PHP, v.v.


```
vi wordpress/vars/default.yml
```

Nội dung file 
```
#PHP Settings
php_modules: [ 'php', 'php-curl', 'php-gd', 'php-mbstring', 'php-xml', 'php-xmlrpc', 'php-soap', 'php-intl', 'php-zip' ]
 
#MySQL Settings
mysql_root_password: "your-root-password"
mysql_db: "wpdb"
mysql_user: "wpuser"
mysql_password: "1"
 
#HTTP Settings
http_host: "wp.example.com"
http_conf: "wp.example.com.conf"
http_port: "80"
```

- Trong đó 

Giá trị | ý nghĩa 
---|---
php_modules|là danh sách tất cả các module php mà wordpress yêu cầu  
mysql_root_password | là mật khẩu mysql người dùng root mà bạn muốn cài đặt 
mysql_db  | là tên database của wordpress mà bạn muốn tạo 
mysql_user| là tên người dùng database mà bạn muốn tạo
mysql_password |là mật khẩu người dùng mysql mà bạn muốn tạo 
http_host| là tên FQDN của trang web wordpress 
http_conf |là tên file cấu hình của wordpress 
http_port| là port của apache webserver


# 3. Tạo file Apache Virtual Host Template 
- Tạo một teamplate cho apache virtual host config. Ansible sẽ copy file template đến host đích

```
vi wordpress/files/httpd.conf
```

- Nội dung file 

```
<VirtualHost *:{{ http_port }}>
   ServerAdmin webmaster@localhost
   ServerName {{ http_host }}
   ServerAlias www.{{ http_host }}
   DocumentRoot /var/www/{{ http_host }}/wordpress
   ErrorLog /var/log/httpd/error.log
   CustomLog /var/log/httpd/access.log combined
 
   <Directory /var/www/{{ http_host }}/wordpress>
         Options Indexes FollowSymLinks
     AllowOverride all
     Require all granted
   </Directory>
</VirtualHost>
```

# 4. Tạo Playbook cho Apache Roles 
-  Tạo playbook cho the Apache role để cài đặt và cấu hình  Apache trên host. Bao gồm: 
  - Cài đặt apache
  - Khởi động apache server vầ cho phép khởi động khi boot
  - Tạo thư mục Apache web root
  - Copy file cấu hình template  Apache virtual host  từ ansible controler đến ansible host


- Tạo file trong thư mục tasks
```
 vi wordpress/roles/apache/tasks/main.yml
```

  - Nội dung: 
```
- name: Install HTTP Packages
  yum: name=httpd update_cache=yes state=latest
 
- name: Start apache2 service
  systemd: name=httpd state=started enabled=yes
 
- name: Create Apache Document Root
  file:
    path: "/var/www/{{ http_host }}"
    state: directory
    owner: "apache"
    group: "apache"
    mode: '0755'
 
- name: Set up Apache VirtualHost
  template:
    src: "files/httpd.conf"
    dest: "/etc/httpd/conf.d/{{ http_conf }}"
    owner: root
    group: root
    mode: u=rw,g=r,o=r
```

# 5. Tạo playbook cho PHP role

- Task: 
  - Cài đặt PHP Remi repository.
  - Khởi động mặc định PHP repository và cho phép Remi repository.
  - Cài đặt PHP and modules yêu cầu .


- Tạo file playbook 
```   
vi wordpress/roles/php/tasks/main.yml
```
  - Nội dung
```
- name: Install PHP Remi Repository
  yum: name=https://rpms.remirepo.net/enterprise/remi-release-8.rpm update_cache=yes state=latest

- name: Enable PHP Remi Repository
  command: dnf module reset php -y
  command: dnf module enable php:remi-8.1 -y

- name: Install PHP Extensions
  yum: name={{ item }} update_cache=yes state=latest
  loop: "{{ php_modules }}"

```
# 6. Tạo playbook cho Mysql

- task: 
  - Cài Mysql và các package khác
  - Khỏi động mysql và cho phép khởi động khi boot
  - Thiết lập Mysql root password
  - tạo database cho wordpress
  - Tạo người dùng datbase cho wordpress

- Chỉnh sửa file  `wordpress/roles/mysql/tasks/main.yml`
```
vi wordpress/roles/mysql/tasks/main.yml
```
  - Nội dung
```
# MySQL Configuration
    - name: Install MySQL Packages
      yum: name={{ item }} update_cache=yes state=latest
      loop: [ 'mysql-server', 'php-mysqlnd', 'python3-PyMySQL' ]
 
    - name: Start mysqld service
      systemd: name=mysqld state=started enabled=yes
 
    - name: Set MySQL root Password
      mysql_user:
        login_host: 'localhost'
        login_user: 'root'
        login_password: ''
        name: 'root'
        password: '{{ mysql_root_password }}'
        state: present
 
    - name: Creates database for WordPress
      mysql_db:
        name: "{{ mysql_db }}"
        state: present
        login_user: root
        login_password: "{{ mysql_root_password }}"
 
    - name: Create MySQL user for WordPress
      mysql_user:
        name: "{{ mysql_user }}"
        password: "{{ mysql_password }}"
        priv: "{{ mysql_db }}.*:ALL"
        state: present
        login_user: root
        login_password: "{{ mysql_root_password }}"
```

# 7. Tạo playbook  cho wordpress role

- Task 
  - Tải xuống và giải nén WordPress vào thư mục gốc của web Apache
  - Thiết lập ownership cho wordpress
  - Thiết lập phân quyền cho thư mục 
  - Thiết lập phân quyền cho files
  - đổi tên wordpress giống như file cấu hình
  - Định nghĩa database trong file cấu hình wordpress
  - Khởi động lại apache service

- Tạo file 
```
vi wordpress/roles/wordpress/tasks/main.yml
```
  - Nội dung
```
---
# WordPress Configuration
 
    - name: Download and unpack latest WordPress
      unarchive:
        src: https://wordpress.org/latest.tar.gz
        dest: "/var/www/{{ http_host }}"
        remote_src: yes
        creates: "/var/www/{{ http_host }}/wordpress"
 
    - name: Set ownership
      file:
        path: "/var/www/{{ http_host }}"
        state: directory
        recurse: yes
        owner: apache
        group: apache
 
    - name: Set permissions for directories
      shell: "/usr/bin/find /var/www/{{ http_host }}/wordpress/ -type d -exec chmod 750 {} \\;"
 
    - name: Set permissions for files
      shell: "/usr/bin/find /var/www/{{ http_host }}/wordpress/ -type f -exec chmod 640 {} \\;"
 
    - name: Copy sample config file
      command: mv /var/www/{{ http_host }}/wordpress/wp-config-sample.php /var/www/{{ http_host }}/wordpress/wp-config.php creates=/var/www/{{ http_host }}/wordpress/wp-config.php
      become: yes
 
    - name: Update WordPress config file
      lineinfile:
        path: "/var/www/{{ http_host }}/wordpress/wp-config.php"
        regexp: "{{item.regexp}}"
        line: "{{item.line}}"
      with_items:
        - {'regexp': "define\\( 'DB_NAME', '(.)+' \\);", 'line': "define( 'DB_NAME', '{{mysql_db}}' );"}
        - {'regexp': "define\\( 'DB_USER', '(.)+' \\);", 'line': "define( 'DB_USER', '{{mysql_user}}' );"}
        - {'regexp': "define\\( 'DB_PASSWORD', '(.)+' \\);", 'line': "define( 'DB_PASSWORD', '{{mysql_password}}' );"}
     
    - name: Restart httpd service
      systemd: name=httpd state=restarted
      become: yes
```

# 8. Tạo playbook cho firewall role 
- Tasks
  - disable SELinux
  - Tắt SElinux
  - Cấu hình firewall cho phép apache server
  - Reaload rule firewall

- Tạo file 
```
vi wordpress/roles/firewall/tasks/main.yml
```
  - Nội dung
```
# Firewall Configuration
    - name: Disable SELinux Permanently (Reboot Required)
      selinux: state=disabled
 
    - name: Disable SELinux Without Reboot
      command: /sbin/setenforce 0
 
    - name: Configure Firewall
      firewalld: zone=public service=http permanent=yes state=enabled
 
    - name: Reload Firewall
      systemd: name=firewalld state=reloaded
```

# 9. Tạo file main playbool
- Tạo file 
```
vi wordpress/playbook.yml
```
- Nội dung
```
- hosts: node3
  gather_facts: False
  become: true
  vars_files:
    - vars/default.yml
  roles:
    - apache
    - php
    - mysql
    - wordpress
    - firewall
```

- Chạy playbook
```
ansible-playbook wordpress/playbook.yml
```

Kết quả 
```
[root@localhost ~]# ansible-playbook wordpress/playbook.yml
[WARNING]: While constructing a mapping from /root/wordpress/roles/php/tasks/main.yml, line 4, column 3, found a duplicate dict key (command). Using last
defined value only.

PLAY [node1] ****************************************************************************************************************************************************

TASK [apache : Install HTTP Packages] ***************************************************************************************************************************
ok: [node1]

TASK [Start apache2 service] ************************************************************************************************************************************
ok: [node1]

TASK [apache : Create Apache Document Root] *********************************************************************************************************************
ok: [node1]

TASK [apache : Set up Apache VirtualHost] ***********************************************************************************************************************
ok: [node1]

TASK [php : Install PHP Remi Repository] ************************************************************************************************************************
ok: [node1]

TASK [php : Enable PHP Remi Repository] *************************************************************************************************************************
[WARNING]: Consider using the dnf module rather than running 'dnf'.  If you need to use command because dnf is insufficient you can add 'warn: false' to this
command task or set 'command_warnings=False' in ansible.cfg to get rid of this message.
changed: [node1]

TASK [php : Install PHP Extensions] *****************************************************************************************************************************
changed: [node1] => (item=php)
ok: [node1] => (item=php-curl)
changed: [node1] => (item=php-gd)
ok: [node1] => (item=php-mbstring)
ok: [node1] => (item=php-xml)
changed: [node1] => (item=php-xmlrpc)
changed: [node1] => (item=php-soap)
changed: [node1] => (item=php-intl)
changed: [node1] => (item=php-zip)

TASK [mysql : Install MySQL Packages] ***************************************************************************************************************************
changed: [node1] => (item=mysql-server)
changed: [node1] => (item=php-mysqlnd)
changed: [node1] => (item=python3-PyMySQL)

TASK [Start mysqld service] *************************************************************************************************************************************
changed: [node1]

TASK [mysql : Set MySQL root Password] **************************************************************************************************************************
[WARNING]: Module did not set no_log for update_password
changed: [node1]

TASK [mysql : Creates database for WordPress] *******************************************************************************************************************
changed: [node1]

TASK [mysql : Create MySQL user for WordPress] ******************************************************************************************************************
changed: [node1]

TASK [wordpress : Download and unpack latest WordPress] *********************************************************************************************************
changed: [node1]

TASK [wordpress : Set ownership] ********************************************************************************************************************************
changed: [node1]

TASK [wordpress : Set permissions for directories] **************************************************************************************************************
changed: [node1]

TASK [wordpress : Set permissions for files] ********************************************************************************************************************
changed: [node1]

TASK [wordpress : Copy sample config file] **********************************************************************************************************************
changed: [node1]

TASK [wordpress : Update WordPress config file] *****************************************************************************************************************
changed: [node1] => (item={'regexp': "define\\( 'DB_NAME', '(.)+' \\);", 'line': "define( 'DB_NAME', 'wpdb' );"})
changed: [node1] => (item={'regexp': "define\\( 'DB_USER', '(.)+' \\);", 'line': "define( 'DB_USER', 'wpuser' );"})
changed: [node1] => (item={'regexp': "define\\( 'DB_PASSWORD', '(.)+' \\);", 'line': "define( 'DB_PASSWORD', '1' );"})

TASK [wordpress : Restart httpd service] ************************************************************************************************************************
changed: [node1]

TASK [firewall : Disable SELinux Permanently (Reboot Required)] *************************************************************************************************
[WARNING]: SELinux state temporarily changed from 'enforcing' to 'permissive'. State change will take effect next reboot.
changed: [node1]

TASK [firewall : Disable SELinux Without Reboot] ****************************************************************************************************************
changed: [node1]

TASK [firewall : Configure Firewall] ****************************************************************************************************************************
ok: [node1]

TASK [firewall : Reload Firewall] *******************************************************************************************************************************
changed: [node1]

PLAY RECAP ******************************************************************************************************************************************************
node1                      : ok=23   changed=17   unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

```

# Kiểm tra tại node

```
[root@localhost ~]# mysql -u wpuser -p
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 11
Server version: 8.0.26 Source distribution

Copyright (c) 2000, 2021, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> exit
Bye
```

