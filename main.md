## Chapter 1.3 Installing Ansible

enable
```
[root@ansible1 ~]# vi /etc/yum.repos.d/redhat.repo 

[ansible-automation-platform-2.5-for-rhel-9-x86_64-rpms]
name = Red Hat Ansible Automation Platform 2.5 for RHEL 9 x86_64 (RPMs)
baseurl = https://cdn.redhat.com/content/dist/layered/rhel9/x86_64/ansible-automation-platform/2.5/os
enabled = 1
```
install ansible
```
dnf install ansible-navigator -y
```
check version
```
[root@ansible1 ~]# ansible-navigator --version
ansible-navigator 25.8.0

[root@ansible1 ~]# ansible --version
ansible [core 2.16.14]
  config file = /etc/ansible/ansible.cfg
  configured module search path = ['/root/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python3.11/site-packages/ansible
  ansible collection location = /root/.ansible/collections:/usr/share/ansible/collections
  executable location = /usr/bin/ansible
  python version = 3.11.11 (main, Aug 21 2025, 00:00:00) [GCC 11.5.0 20240719 (Red Hat 11.5.0-5)] (/usr/bin/python3.11)
  jinja version = 3.1.6
  libyaml = True

```

## Chapter 2.  Implementing an Ansible Playbook

1. Khái niệm Inventory

Inventory là danh sách các host (máy chủ) mà Ansible quản lý.

2. Loại Inventory
- Static inventory: viết sẵn trong file text (định dạng INI hoặc YAML).
- Dynamic inventory: được sinh tự động bởi plugin (ví dụ lấy từ AWS, GCP...).

3. Static Inventory (INI-style)

Cấu trúc phổ biến:
```
[webservers]
web1.example.com
web2.example.com
192.0.2.42

[db-servers]
db1.example.com
db2.example.com
```
- Dòng trong ngoặc vuông [ ] là tên nhóm.
- Mỗi dòng sau đó là tên host hoặc IP.

🧩 4. Host có thể thuộc nhiều nhóm

Ví dụ:
```
[webservers]
web1.example.com
web2.example.com

[db-servers]
db1.example.com
db2.example.com

[east-datacenter]
web1.example.com
db1.example.com

[production]
web1.example.com
web2.example.com
db1.example.com
db2.example.com
```
5. Hai nhóm đặc biệt

- all → chứa mọi host trong inventory.
- ungrouped → chứa các host không nằm trong nhóm nào khác.

example
```
[all]
web1.example.com
web2.example.com
db1.example.com
db2.example.com
backup.example.com

[ungrouped]
backup.example.com
```

6. Nested Groups (Nhóm lồng nhau)

Dùng hậu tố `:children` để gom nhóm:
```
[usa]
washington1.example.com
washington2.example.com

[canada]
ontario01.example.com
ontario02.example.com

[north-america:children]
usa
canada
```

→ Nhóm `north-america` gồm tất cả host trong `usa` và `canada`.

7. Rút gọn tên host bằng Range

Giúp tiết kiệm thời gian khi có nhiều host tương tự nhau:
```
server[01:20].example.com   # server01 → server20
192.168.[4:7].[0:255]       # 192.168.4.0 → 192.168.7.255
[a:c].dns.example.com        # a, b, c.dns.example.com
```

Lưu ý: có thể dùng cả số và chữ cái. Nếu có số 0 ở đầu thì phải giữ nguyên khi gọi (server01 ≠ server1).

8. Kiểm tra Inventory

Dùng lệnh:
```
ansible-navigator inventory -i inventory -m stdout --list
```
example
```
{
    "all": {
        "children": ["dbservers", "ungrouped", "webservers"]
    },
    "dbservers": {
        "hosts": ["db1.example.com", "db2.example.com"]
    },
    "webservers": {
        "hosts": ["web1.example.com", "web2.example.com"]
    },
    "ungrouped": {
        "hosts": ["backup.example.com"]
    }
}
```

→ Liệt kê toàn bộ inventory (groups & hosts).
```
ansible-navigator inventory -i inventory
```
→ Giao diện điều hướng, có thể chọn “Browse Groups” hoặc “Browse Hosts”.

Example

Create file `inventory.ini`
```
[root@ansible1 ~]# mkdir test_ansible
[root@ansible1 ~]# cat test_ansible/inventory.ini 
[webservers]
localhost ansible_connection=local

[root@ansible1 test_ansible]# ansible-inventory -i inventory.ini --list
{
    "_meta": {
        "hostvars": {
            "localhost": {
                "ansible_connection": "local"
            }
        }
    },
    "all": {
        "children": [
            "ungrouped",
            "webservers"
        ]
    },
    "webservers": {
        "hosts": [
            "localhost"
        ]
    }
}

[root@ansible1 test_ansible]# ansible all -i /root/test_ansible/inventory.ini -m ping
localhost | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}

# giao diện TUI (text-based UI), tạo file user-level:
[root@ansible1 ~]# cat  ~/.ansible-navigator.yml
---
ansible-navigator:
  execution-environment:
    enabled: false

[root@ansible1 ~]# ansible-navigator inventory -i /root/test_ansible/inventory.ini

```
- Browse groups (0) → duyệt các nhóm host trong inventory.
  - Ví dụ: [webservers], [dbservers], [ungrouped].
  -  có thể chọn một nhóm để xem host bên trong.
- Browse hosts (1) → duyệt tất cả các host.
  - Ví dụ: localhost, web1.example.com, …
  - Dễ kiểm tra host nào đang có trong inventory.

Cách sử dụng:
- Gõ 0 → Enter → xem danh sách nhóm → gõ số nhóm để mở.
- Gõ 1 → Enter → xem danh sách host → chọn host để xem biến (hostvars) nếu có.
- ESC → thoát menu.
> Đây là giao diện trực quan, giúp  kiểm tra inventory mà không cần lệnh JSON dài.


## 2.3 Managing Ansible Configuration Files

Cấu hình Ansible
 có thể tạo và chỉnh sửa hai tệp trong mỗi thư mục dự án Ansible để cấu hình hành vi của Ansible và lệnh `ansible-navigator`. Thư mục dự án Ansible là thư mục mà  chạy các playbook bằng cách sử dụng lệnh `ansible-navigator`.

- `ansible.cfg`, cấu hình hành vi của một số công cụ Ansible.
- `ansible-navigator.yml`, thay đổi các tùy chọn mặc định cho lệnh `ansible-navigator`.

```
[root@ansible1 ~]# vi /etc/ansible/ansible.cfg 
[root@ansible1 ~]# vi /etc/ansible-navigator/ansible-navigator.yml


# 10/29/2025
[root@ansible1 ~]# cat  /etc/ansible-navigator/ansible-navigator.yml
---
ansible-navigator:
  execution-environment:
    enabled: false
```

3. Kiểm tra cấu hình hiện tại
```
ansible-navigator config
```
time: 10/29/2025
```
 44│Default ask pass                                         True           default        False
 45│Default ask vault pass                                   True           default        False
 46│Default become                                           True           default        False
 47│Default become ask pass                                  True           default        False
 48│Default become exe                                       True           default        
 49│Default become flags                                     True           default        
 50│Default become method                                    True           default        
 51│Default become user                                      True           default        root 
 92│Default remote port                                      True           default        None
 93│Default remote user                                      True           default        None
```

| Vị trí file                             | Phạm vi ảnh hưởng                         | Khi nào nên dùng                                                                                                                 |
| --------------------------------------- | ----------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `/etc/ansible/ansible.cfg`              | Toàn hệ thống (mọi user, mọi project)     | Khi  muốn **toàn bộ hệ thống / team** dùng chung 1 cấu hình (ví dụ trong môi trường lab hoặc production có chuẩn thống nhất). |
| `~/.ansible.cfg`                        | Chỉ ảnh hưởng đến **user đó**             | Khi  muốn chỉ **mình ** dùng cấu hình này, không ảnh hưởng tới người khác.                                                 |
| `./ansible.cfg` (trong thư mục project) | Chỉ ảnh hưởng **project hiện tại**        | Thường dùng nhất trong thực tế — giúp mỗi project có cấu hình riêng (ví dụ user, become, inventory…).                          |
| Biến môi trường `ANSIBLE_CONFIG`        | Chỉ ảnh hưởng **phiên làm việc hiện tại** | Dùng tạm thời khi  muốn test 1 cấu hình khác mà không muốn sửa file thật.                                                     |

Cách Ansible “quyết” cấu hình (ưu tiên)  
ANSIBLE_CONFIG env → ./ansible.cfg (cwd) → ~/.ansible.cfg → /etc/ansible/ansible.cfg.
Vì vậy thay đổi cho project đặt trong ./ansible.cfg là an toàn, không ảnh hưởng toàn hệ thống.
## Managing Ansible Settings

Example
```
[root@ansible1 test_ansible]# ll
total 12
-rw-r--r--. 1 root root   91 Oct 29 15:34 ansible.cfg
-rw-r--r--. 1 root root 3313 Oct 29 15:20 ansible-navigator.log
-rw-r--r--. 1 root root  101 Oct 29 15:15 inventory.ini

[root@ansible1 test_ansible]# cat ansible.cfg 
[defaults]
inventory = ~/test_ansible/inventory.ini
remote_user = nghiahv
ask_pass = True

[root@ansible1 test_ansible]# cat inventory.ini 
[webserver]
#localhost ansible_connection=local
web1 ansible_host=192.168.38.31 ansible_user=nghiahv

[root@ansible1 test_ansible]# ansible webserver -m ping
SSH password: 
web1 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
```


## 2.5 Writing and Running Playbooks

Executing a Dry Run
```
[root@ansible1 test_ansible]# ansible-navigator run -m stdout ping-myself.yml --check

PLAY [Ping test] ***************************************************************

TASK [Gathering Facts] *********************************************************
ok: [web1]

TASK [Ping remote host] ********************************************************
ok: [web1]

PLAY RECAP *********************************************************************
web1                       : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0 
```
So sánh
```
$ ansible-navigator run ping-myself.yml -m stdout
$ ansible-navigator run ping-myself.yml
```

- `-m stdout`: Chạy Ansible Navigator ở chế độ stdout, tức là hiển thị kết quả trực tiếp ra terminal như khi dùng ansible-playbook. Không có giao diện tương tác.

- Không có `-m stdout`: Sử dụng chế độ mặc định là interactive mode, tức là có giao diện bán tương tác trong terminal, cho phép điều hướng qua các phần kết quả.

## 2.7 Implementing Multiple Plays

YAML Syntax

YAML Comments
```
# This is a YAML comment
hostname: server1  # This is also a YAML comment
```

YAML Strings
- Chuỗi không bắt buộc phải đặt trong dấu nháy.
- Có thể dùng ' hoặc " nếu chuỗi có ký tự đặc biệt hoặc khoảng trắng.
```
message: Hello world
message: 'Hello world'
message: "Hello world"
```

Chuỗi nhiều dòng
```
address: |
        Example Company
        123 Main Street
        Atlanta, GA 30303
```
Dùng > để gộp thành một dòng, bỏ xuống dòng:
```
fold_newlines: >
        This is an example
        of a long string,
        that will become
        a single sentence once folded.
```
→ Kết quả: "This is an example of a long string, that will become a single sentence once folded."

YAML Dictionaries (key-value)

Dạng khối
```
name: svcrole
svcservice: httpd
svcport: 80
```
Dạng dòng (inline):
```
{name: svcrole, svcservice: httpd, svcport: 80}
```

YAML Lists

Dạng khối
```
hosts:
  - servera
  - serverb
  - serverc
```
Dạng dòng (inline):

```
hosts: [servera, serverb, serverc]
```
Obsolete Playbook Shorthand

Dạng viết tắt
```
- name: Shorthand form
  ansible.builtin.service: name=httpd enabled=true state=started
```

Dạng chuẩn (nên dùng):
```
- name: Normal form
  ansible.builtin.service:
    name: httpd
    enabled: true
    state: started
```

# Chapter 3.  Managing Variables and Facts

Variables in Playbooks

Biến giúp bạn tái sử dụng giá trị trong nhiều task mà không phải viết lại. Ví dụ:

```
vars:
  web_package: httpd

tasks:
  - name: Install web package
    ansible.builtin.dnf:
      name: "{{ web_package }}"
      state: present
```
Defining Variables in Playbooks

Trong playbook (dùng vars):

```
- hosts: all
  vars:
    user: joe
    home: /home/joe
```
Trong file bên ngoài (dùng vars_files):

```
- hosts: all
  vars_files:
    - vars/users.yml

# File users.yml sẽ chứa:
user: joe
home: /home/joe
```

**Using Variables in Playbooks**
- Dùng dấu `{{ variable_name }}` để gọi biến

- Nếu biến nằm đầu dòng, phải đặt trong dấu nháy "{{ variable }}" để tránh lỗi YAML

Example
```
name: "{{ service }}"

# sai 
name: {{ service }}  # dễ gây lỗi
```

**Host Variables and Group Variables**

















































