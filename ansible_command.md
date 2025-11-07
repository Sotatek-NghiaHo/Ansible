Tong quat
```
ansible <group_or_host> -i <inventory_file> -m <module>

# example
ansible-navigator run <ten_playbook.yml> -m stdout

```

Kiểm tra cấu hình hiện tại
```
ansible-navigator config
```

Hoặc nếu bạn muốn tìm nhanh các module liên quan
```
ansible-navigator doc -l | grep ping
```

xem nội dung chi tiết của module ansible.builtin.ping bằng ansible-navigator
```
ansible-navigator doc ansible.builtin.ping

ansible-navigator doc ansible.builtin.ping -m stdout
```
Lệnh này sẽ hiển thị:
- Mô tả chức năng của module
- Các tham số (nếu có)
- Ví dụ sử dụng
- Ghi chú đặc biệt

---







