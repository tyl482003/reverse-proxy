# Triển khai WordPress Cluster với Reverse Proxy trên GCP bằng Ansible

---

## 1. Kiến trúc hệ thống

**Các thành phần VM:**
- 1 VM: Ansible Manager (dùng để cài đặt và quản lý các server)
- 1 VM: Database Server (cài MariaDB/MySQL)
- 1 VM: Reverse Proxy (cài Nginx làm reverse proxy)
- 3 VM: Web Server (cài WordPress + Nginx + PHP-FPM)

**Sơ đồ:**

```
End user
   |
[Reverse Proxy (Nginx)]
   |
--------------------------
|    |     |     |
wp1  wp2   wp3   (WordPress + Nginx + PHP-FPM)
   |
[Database Server (MariaDB/MySQL)]
```

---

## 2. Chuẩn bị trên Google Cloud Platform (GCP)

### a. Tạo SSH key trên máy local
```sh
ssh-keygen -t rsa -b 4096
```
- Dùng file private key để SSH vào các VM, file public key thêm vào khi tạo VM.

### b. Tạo 6 VM với các thông số ví dụ:

| VM Name         | Vai trò         | Public IP | Internal IP | OS          | Cấu hình |
|-----------------|-----------------|-----------|-------------|-------------|----------|
| ansible-mgr     | Ansible Manager | Có        | 10.0.0.2    | Ubuntu 22.04| e2-medium|
| db-server       | Database        | Không     | 10.0.0.3    | Ubuntu 22.04| e2-small |
| proxy-server    | Reverse Proxy   | Có        | 10.0.0.4    | Ubuntu 22.04| e2-micro |
| wp1             | WordPress       | Không     | 10.0.0.5    | Ubuntu 22.04| e2-micro |
| wp2             | WordPress       | Không     | 10.0.0.6    | Ubuntu 22.04| e2-micro |
| wp3             | WordPress       | Không     | 10.0.0.7    | Ubuntu 22.04| e2-micro |

**Lưu ý:**
- Chỉ `ansible-mgr` và `proxy-server` cần public IP.
- Tất cả các VM nên nằm cùng 1 VPC/subnet (ví dụ: `wordpress-subnet`).
- Khi tạo VM, thêm SSH public key vào trường SSH key của user tương ứng.

### c. Mở firewall:
- Cho phép port 22 (SSH) vào ansible-mgr từ IP của bạn.
- Cho phép port 80 vào proxy-server từ Internet.
- Cho phép port 3306 (MySQL) từ các web server đến db-server.
- Cho phép traffic nội bộ giữa các VM (trong cùng subnet).

---

## 3. SSH vào ansible-mgr bằng private key
```sh
ssh -i ~/.ssh/id_rsa <user>@<ansible-mgr-public-ip>
```

---

## 4. Cài đặt Ansible và các công cụ cần thiết trên ansible-mgr

```sh
sudo apt update
sudo apt install -y ansible tree git
```

---

``` text
└── ansible-project
    ├── ansible.cfg
    ├── group_vars
    │   ├── database.yml
    │   ├── proxy.yml
    │   └── wordpress.yml
    ├── inventory
    │   └── hosts
    ├── playbooks
    │   └── site.yml
    └── roles
        ├── database
        │   ├── handlers
        │   │   └── main.yml
        │   └── tasks
        │       └── main.yml
        ├── proxy
        │   ├── handlers
        │   │   └── main.yml
        │   ├── tasks
        │   │   └── main.yml
        │   └── templates
        │       └── nginx-proxy.conf.j2
        └── wordpress
            ├── tasks
            │   └── main.yml
            └── templates
                └── wp-config.php.j2
```

## 5. Tạo cấu trúc thư mục dự án Ansible

```sh
mkdir -p ansible-project/{group_vars,inventory,playbooks,roles/{database/{handlers,tasks},proxy/{handlers,tasks,templates},wordpress/{tasks,templates}}} \
&& touch ansible-project/ansible.cfg \
ansible-project/group_vars/{database.yml,proxy.yml,wordpress.yml} \
ansible-project/inventory/hosts \
ansible-project/playbooks/site.yml \
ansible-project/roles/database/handlers/main.yml \
ansible-project/roles/database/tasks/main.yml \
ansible-project/roles/proxy/handlers/main.yml \
ansible-project/roles/proxy/tasks/main.yml \
ansible-project/roles/proxy/templates/nginx-proxy.conf.j2 \
ansible-project/roles/wordpress/tasks/main.yml \
ansible-project/roles/wordpress/templates/wp-config.php.j2

```

---

## 6. Nội dung chi tiết từng file

### 6.1 `ansible.cfg`
```ini
[defaults]
become_ask_pass=False
inventory = inventory/hosts
remote_user = <user>
roles_path = roles
host_key_checking = False
retry_files_enabled = False
```  
**Lưu ý:** Thay `<user>` bằng username VM của bạn trên GCP (thường là `ubuntu`).

---

### 6.2 `inventory/hosts`
```ini
[ansible_manager]
ansible-mgr ansible_host= internal ip ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/id_rsa

[database]
db-server ansible_host=internal ip ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/id_rsa

[proxy]
proxy-server ansible_host=internal ip ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/id_rsa

[wordpress]
wp1 ansible_host=internal ip ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/id_rsa
wp2 ansible_host=internal ip ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/id_rsa
wp3 ansible_host=internal ip ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/id_rsa
```

---

### 6.3 `group_vars/database.yml`
```yaml
db_name: wordpress
db_user: wp_user
db_password: supersecret123
```

---

### 6.4 `group_vars/wordpress.yml`
```yaml
db_host: internal ip
db_name: wordpress
db_user: wp_user
db_password: supersecret123
wp_proxy_ip: external ip
```

---

### 6.5 `group_vars/proxy.yml`
```yaml
wordpress_servers:
  - internal ip
  - internal ip
  - internal ip
```

---

### 6.6 `playbooks/site.yml`
```yaml
- hosts: database
  become: yes
  vars_files:
  - ../group_vars/database.yml
  roles:
    - database

- hosts: wordpress
  become: yes
  vars_files:
    - ../group_vars/wordpress.yml
  roles:
    - wordpress

- hosts: proxy
  become: yes
  vars_files:
    - ../group_vars/proxy.yml
  roles:
    - proxy
```

---

### 6.7 `roles/database/handlers/main.yml`
```yaml
- name: restart mysql
  service:
    name: mariadb
    state: restarted
```

---

### 6.8 `roles/database/tasks/main.yml`
```yaml
- name: Cài đặt MariaDB
  apt:
    name: mariadb-server
    state: present
    update_cache: yes

- name: Cài đặt PyMySQL cho Ansible
  apt:
    name: python3-pymysql
    state: present

- name: Tạo database cho WordPress
  mysql_db:
    name: "{{ db_name }}"
    state: present
    login_unix_socket: /var/run/mysqld/mysqld.sock

- name: Tạo user cho WordPress
  mysql_user:
    name: "{{ db_user }}"
    password: "{{ db_password }}"
    priv: "{{ db_name }}.*:ALL"
    host: '%'
    state: present
    login_unix_socket: /var/run/mysqld/mysqld.sock

- name: Mở MySQL cho các web server
  lineinfile:
    path: /etc/mysql/mariadb.conf.d/50-server.cnf
    regexp: '^bind-address'
    line: 'bind-address = 0.0.0.0'
  notify: restart mysql

- name: Cài đặt UFW
  apt:
    name: ufw
    state: present
    update_cache: yes

- name: Mở port 3306 cho các web server
  ufw:
    rule: allow
    port: 3306
    proto: tcp
```

---

### 6.9 `roles/wordpress/tasks/main.yml`
```yaml
- name: Cài đặt Nginx, PHP-FPM, và các package cần thiết
  apt:
    name:
      - nginx
      - php-fpm
      - php-mysql
      - wget
      - unzip
    state: present
    update_cache: yes

- name: Tải WordPress
  get_url:
    url: https://wordpress.org/latest.zip
    dest: /tmp/wordpress.zip

- name: Giải nén WordPress vào /var/www/html
  unarchive:
    src: /tmp/wordpress.zip
    dest: /var/www/html/
    remote_src: yes

- name: Đảm bảo thư mục /var/www/html/wordpress thuộc về www-data
  file:
    path: /var/www/html/wordpress
    owner: www-data
    group: www-data
    state: directory
    recurse: yes

- name: Tạo file cấu hình wp-config.php
  template:
    src: wp-config.php.j2
    dest: /var/www/html/wordpress/wp-config.php
    owner: www-data
    group: www-data

- name: Cấu hình Nginx cho WordPress
  copy:
    dest: /etc/nginx/sites-available/wordpress
    content: |
      server {
          listen 80;
          server_name _;
          root /var/www/html/wordpress;
          index index.php index.html index.htm;

          location / {
              try_files $uri $uri/ /index.php?$args;
          }

          location ~ \.php$ {
              include snippets/fastcgi-php.conf;
              fastcgi_pass unix:/var/run/php/php-fpm.sock;
              fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
              include fastcgi_params;
          }

          location ~ /\.ht {
              deny all;
          }

          client_max_body_size 100M;
      }

- name: Kích hoạt site wordpress và tắt site default
  file:
    src: /etc/nginx/sites-available/wordpress
    dest: /etc/nginx/sites-enabled/wordpress
    state: link
    force: yes

- name: Xóa site default nếu tồn tại
  file:
    path: /etc/nginx/sites-enabled/default
    state: absent

- name: Restart Nginx và PHP-FPM
  service:
    name: nginx
    state: restarted

- name: Restart PHP-FPM
  service:
    name: php8.1-fpm
    state: restarted
  ignore_errors: yes
```

---

### 6.10 `roles/wordpress/templates/wp-config.php.j2`
```php
<?php
// Fix cho reverse proxy/lấy IP thật
if (isset($_SERVER['HTTP_X_FORWARDED_FOR'])) {
    $ips = explode(',', $_SERVER['HTTP_X_FORWARDED_FOR']);
    $_SERVER['REMOTE_ADDR'] = trim($ips[0]);
}
if (isset($_SERVER['HTTP_X_FORWARDED_PROTO']) && $_SERVER['HTTP_X_FORWARDED_PROTO'] == 'https') {
    $_SERVER['HTTPS'] = 'on';
}

define('DB_NAME', '{{ db_name }}');
define('DB_USER', '{{ db_user }}');
define('DB_PASSWORD', '{{ db_password }}');
define('DB_HOST', '{{ db_host }}');
define('DB_CHARSET', 'utf8');
define('DB_COLLATE', '');

$table_prefix  = 'wp_';

define('WP_DEBUG', false);

// Cấu hình base URL qua proxy
define('WP_SITEURL', 'http://{{ wp_proxy_ip }}/wordpress');
define('WP_HOME', 'http://{{ wp_proxy_ip }}/wordpress');

if ( !defined('ABSPATH') )
    define('ABSPATH', dirname(__FILE__) . '/');

require_once(ABSPATH . 'wp-settings.php');
```

---

### 6.11 `roles/proxy/tasks/main.yml`
```yaml
- name: Cài đặt Nginx cho reverse proxy
  apt:
    name: nginx
    state: present
    update_cache: yes

- name: Tạo file cấu hình Nginx reverse proxy
  template:
    src: nginx-proxy.conf.j2
    dest: /etc/nginx/sites-available/proxy
  notify: restart nginx

- name: Bật site proxy
  file:
    src: /etc/nginx/sites-available/proxy
    dest: /etc/nginx/sites-enabled/proxy
    state: link
    force: yes

- name: Xóa default site nếu có
  file:
    path: /etc/nginx/sites-enabled/default
    state: absent

- name: restart nginx
  become: yes
  ansible.builtin.service:
    name: nginx
    state: restarted
```

---

### 6.12 `roles/proxy/templates/nginx-proxy.conf.j2`
```nginx
upstream wordpress_cluster {
    {% for ip in wordpress_servers %}
    server {{ ip }};
    {% endfor %}
}

server {
    listen 80;
    server_name _;

    location / {
        proxy_pass http://wordpress_cluster;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 90;
    }

    # Tùy chọn: giới hạn upload lớn hơn mặc định
    client_max_body_size 100M;
}
```
### 6.13 `roles/proxy/handlers/main.yml`
```nginx
- name: restart nginx
  become: yes
  ansible.builtin.service:
    name: nginx
    state: restarted
```

---

## 7. Triển khai

```sh
cd ~/ansible-project
ansible-playbook -i inventory/hosts playbooks/site.yml
```

---

## 8. Kiểm tra hệ thống

- Truy cập `http://<proxy-server-public-ip>/wordpress` để kiểm tra giao diện WordPress.
- Đăng nhập, thử tạo bài viết, upload ảnh.
- Vào db-server kiểm tra dữ liệu đã lưu vào MySQL.

---

## 9. Lưu ý bảo mật & mở rộng

- Chỉ mở port 22 cho ansible-mgr và proxy-server từ IP quản trị.
- Port 80 chỉ mở ở proxy-server.
- Các web server và db-server chỉ dùng private IP.
- Để scale thêm web server, tạo VM mới, bổ sung vào `inventory/hosts` và `group_vars/proxy.yml`, sau đó chạy lại playbook.
- Có thể bổ sung HTTPS cho proxy bằng Let's Encrypt.

---

## 10. Kết luận

Bạn đã hoàn thành hệ thống WordPress cluster với reverse proxy, có thể tự động mở rộng, cân bằng tải và bảo mật tốt trên hạ tầng GCP.

---
