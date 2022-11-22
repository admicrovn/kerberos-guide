# Cài đặt và cấu hình Kerberos Master - Slave

## Mục lục

- [1. Tổng quan về  Kerberos](#1)
- [2. Cài đặt và cấu hình Kerberos](#2)
  - [2.1 Chuẩn bị](#2.1)
  - [2.2 Các bước tiến hành](#2.2)
     - [2.2.1 Cài đặt Kerberos trên các máy chủ](#2.2.1)
     - [2.2.2 Cấu hình Kerberos trên Máy chủ Master](#2.2.2)
     - [2.2.3 Tạo principal và kiểm tra kết quả](#2.2.3)
     - [2.2.4 Tạo principal cho Kerberos Slave](#2.2.4)
     - [2.2.5 Sao chép các file cấu hình sang máy chủ Kerberos Slave](#2.2.5)
     - [2.2.6 Cấu hình máy chủ Kerberos Slave](#2.2.6)
     - [2.2.7 Khởi động Kerberos trên Slave và kích hoạt kropd](#2.2.7)
     - [2.2.8 Xuất dữ liệu trên Kerberos Master và đẩy sang Kerberos Slave](#2.2.8)
     - [2.2.9 Kiểm tra dữ liệu trên Kerberos Slave](#2.2.9)
- [3. Tham khảo](#3)

## 1. Tổng quan về  Kerberos
 <a name="1" />

**Đang cập nhật**

## 2. Cài đặt và cấu hình Kerberos
 <a name="2" />

### 2.1 Chuẩn bị
 <a name="2.1" />

Trong bài viết này, chúng ta cần 2 máy chủ để thực hiện: một máy đảm nhiệm làm KDC Master để quản lý và nhận yêu cầu và cung cấp ticket; một máy làm KDC Slave để có thể nhận yêu cầu cấp ticket từ phía người dùng lên khi KDC Master gián đoạn.

Thông tin cụ thể như sau:

| IP | OS | FQDN | Vai trò (Role) | 
| -- | -- | -- | -- |
| 10.10.10.1 | Ubuntu 20.04 | kdc1.hoangdh.lab | Kerberos Master
| 10.10.10.2 | Ubuntu 20.04 | kdc2.hoangdh.lab | Kerberos Slave

Thông tin Realm: `SECURE.LAB`

Đầu tiên, ta phải đặt lại hostname cho các máy chủ sao cho chuẩn và ghi chúng vào `/etc/hosts`.

Thực hiện SSH vào từng server để đặt hostname tương ứng:

Trên máy chủ `kdc1`:

```
hostnamectl set-hostname kdc1.hoangdh.lab
```

Trên máy chủ `kdc2`:

```
hostnamectl set-hostname kdc2.hoangdh.lab
```

Sau đó, ghi thông tin vào file `/etc/hosts` vào cả 2 máy `kdc1` và `kdc2`

> vim /etc/hosts

```
...
10.10.10.1 kdc1.hoangdh.lab kdc1
10.10.10.2 kdc2.hoangdh.lab kdc2
```

### 2.2 Các bước tiến hành
 <a name="2.2" />

#### 2.2.1 Cài đặt Kerberos trên các máy chủ
 <a name="2.2.1" />

 **Chú ý**: Bước thực hiện trên cả 2 Kerberos Master và Slave

Cài đặt Kerberos Server trên cả 2 máy chủ ở chế độ `noninteractive` - Không tương tác; với mục đích là chỉ cài đặt các gói liên quan và sẽ cấu hình bằng tay ở các bước tiếp theo.

```
DEBIAN_FRONTEND=noninteractive apt-get install krb5-kdc krb5-admin-server -y
```

Chờ một vài phút, quá trình cài đặt các gói thành công. Ta tiếp tục các bước dưới để cấu hình.

#### 2.2.2 Cấu hình Kerberos trên Máy chủ Master
 <a name="2.2.2" />

**Chú ý**: Bước thực hiện trên Kerberos Master

Như đã thông tin ở phần trên, chúng ta sử dụng `SECURE.LAB` là realm. Vì vậy, khi triển khai ở môi trường thực tế; hãy đổi lại thông tin này để phù hợp với dự án của bạn.

Sửa 2 file `/etc/krb5.conf` và `/etc/krb5kdc/kdc.conf`

> vim /etc/krb5.conf

```
[logging]
 default = FILE:/var/log/krb5libs.log
 kdc = FILE:/var/log/krb5kdc.log
 admin_server = FILE:/var/log/kadmind.log
 
[libdefaults]
 default_realm = SECURE.LAB
 dns_lookup_realm = false
 dns_lookup_kdc = false
 ticket_lifetime = 24h
 renew_lifetime = 7d
 forwardable = true
 
[realms]
 SECURE.LAB = {
  kdc = kdc1.hoangdh.lab
  kdc = kdc2.hoangdh.lab
  admin_server = kdc1.hoangdh.lab
 }
 
[domain_realm]
 .secure.lab = SECURE.LAB
 secure.lab = SECURE.LAB
```

> vim /etc/krb5kdc/kdc.conf

```
[kdcdefaults]
    kdc_ports = 88,750
    kdc_tcp_listen = 88

[realms]
    SECURE.LAB = {
        kadmind_port = 749
        max_life = 12h 0m 0s
        max_renewable_life = 7d 0h 0m 0s
        master_key_type = aes256-cts
        supported_enctypes = aes256-cts:normal aes128-cts:normal
        # If the default location does not suit your setup,
        # explicitly configure the following values:
        #    database_name = /var/krb5kdc/principal
        #    key_stash_file = /var/krb5kdc/.k5.ATHENA.MIT.EDU
        #    acl_file = /var/krb5kdc/kadm5.acl
    }

[logging]
    # By default, the KDC and kadmind will log output using
    # syslog.  You can instead send log output to files like this:
    kdc = FILE:/var/log/krb5kdc.log
    admin_server = FILE:/var/log/kadmin.log
    default = FILE:/var/log/krb5lib.log
```

- Khởi tạo REALM

Bước này sẽ tạo một cơ sở dữ liệu để lưu trữ các thông tin principal, mật khẩu được lưu trữ lại một file `stash`. 

> krb5_newrealm

Nhập mât khẩu cho Master key để truy cập vào database lưu trữ các principal.

Sau khi khởi tạo thành công, ta tiếp tục thêm rule cho phép user có quyền quản trị. Trong ví dụ này, ta phân cho các user có hậu tố là `admin@SECURE.LAB`.

> vim /etc/krb5kdc/kadm5.acl

```
*/admin@SECURE.LAB
```

Lưu lại file.

- Chỉnh sửa file systemd cho phép ghi log vào `/var/log`
<a name='allowlog' />

> cat /lib/systemd/system/krb5-kdc.service

```
[Unit]
Description=Kerberos 5 Key Distribution Center


[Service]
Type=forking
PIDFile=/var/run/krb5-kdc.pid
ExecReload=/bin/kill -HUP $MAINPID
EnvironmentFile=-/etc/default/krb5-kdc
ExecStart=/usr/sbin/krb5kdc -P /var/run/krb5-kdc.pid $DAEMON_ARGS
InaccessibleDirectories=-/etc/ssh -/etc/ssl/private  /root
ReadOnlyDirectories=/
ReadWriteDirectories=-/var/tmp /tmp /var/lib/krb5kdc -/var/run /run
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
Restart=on-abnormal


[Install]
WantedBy=multi-user.target
```

Thêm `/var/log` vào trường `ReadWriteDirectories`, nội dung đã chỉnh sửa như sau:

```
[Unit]
Description=Kerberos 5 Key Distribution Center


[Service]
Type=forking
PIDFile=/var/run/krb5-kdc.pid
ExecReload=/bin/kill -HUP $MAINPID
EnvironmentFile=-/etc/default/krb5-kdc
ExecStart=/usr/sbin/krb5kdc -P /var/run/krb5-kdc.pid $DAEMON_ARGS
InaccessibleDirectories=-/etc/ssh -/etc/ssl/private  /root
ReadOnlyDirectories=/
ReadWriteDirectories=-/var/tmp /tmp /var/lib/krb5kdc -/var/run /run /var/log/
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
Restart=on-abnormal


[Install]
WantedBy=multi-user.target
```

Lưu lại và reload systemd

> systemctl reload-daemon

Khởi động và kích hoạt KDC, KDC-Admin

```
systemctl start krb5-kdc
systemctl start krb5-kadmin
systemctl enable krb5-kdc
systemctl enable krb5-kadmin

```

#### 2.2.3 Tạo principal và kiểm tra kết quả
<a name="2.2.3" />

**Chú ý**: Bước thực hiện trên Kerberos Master

##### Tạo mới principal:

Trên máy chủ có cài sẵn kdc-admin, ta sử dụng lệnh để đăng nhập vào giao diện quản trị:

> kadmin.local

Tạo một principals có tên là `user1/test` và thoát khỏi `kadmin`: 

```
kadmin:  addprinc user1/test
WARNING: no policy specified for user1/test@SECURE.LAB; defaulting to no policy
Enter password for principal "user1/test@SECURE.LAB": 
Re-enter password for principal "user1/test@SECURE.LAB": 
Principal "user1/test@SECURE.LAB" created.
kadmin: exit
```

##### Kiểm tra kết quả:

Đăng nhập bằng câu lệnh sau

```
$ kinit -p user1/test@SECURE.LAB
Password for user1/test@SECURE.LAB: 
```

Kiểm tra bằng lệnh:

```
$ klist
Ticket cache: FILE:/tmp/krb5cc_0
Default principal: user1/test@SECURE.LAB

Valid starting       Expires              Service principal
09/03/2021 11:28:33  09/03/2021 23:28:33  krbtgt/SECURE.LAB@SECURE.LAB
	renew until 09/10/2021 11:28:33
```

Hủy phiên và xóa bỏ ticket bằng lệnh

```
$ kdestroy
$ klist
klist: No credentials cache found (filename: /tmp/krb5cc_0)
```

#### 2.2.4 Tạo principal cho Kerberos Slave
<a name="2.2.4" />

**Chú ý**: Bước thực hiện trên Kerberos Master

Trong quá trình đồng bộ dữ liệu từ Master-Slave, KDC dùng một principal là `host` để xác thực giữa các KDC với nhau. Để tạo chúng, ta sử dụng lệnh sau:

```
kadmin.local -q 'addprinc -randkey host/kdc1.secure.lab'
kadmin.local -q 'addprinc -randkey host/kdc2.secure.lab'
```

Thêm thông tin vào keytab mặc định: `/etc/krb5.keytab`

```
kadmin.local -q 'ktadd -k /etc/krb5.keytab host/kdc1.secure.lab'
kadmin.local -q 'ktadd -k /etc/krb5.keytab host/kdc2.secure.lab'
```

#### 2.2.5 Sao chép các file cấu hình sang máy chủ Kerberos Slave
<a name="2.2.5" />

Sau khi hoàn thành các thao tác cơ bản trên máy chủ Kerberos Master, ta sao chép các file cần thiết từ máy Master sang Slave. Danh sách cụ thể các file như sau:

| Tệp tin | Mô tả |
| -- | -- |
| /etc/krb5.conf | File cấu hình thông tin cho client |
| /etc/krb5kdc/kdc.conf | File cấu hình KDC |
| /etc/krb5kdc/.k5.SECURE.LAB | Stash file - Lưu trữ thông tin secret của Database |
| /etc/krb5kdc/kadm5.acl | Lưu trữ ACL, rule quản trị KDC |
| /etc/krb5.keytab | Lưu trữ thông tin xác thực khi đồng bộ dữ liệu |


**Lưu ý:** 2 file `/etc/krb5kdc/kadm5.acl` và `/etc/krb5.keytab` không bắt buộc phải sao chép sang Slave. Nhưng việc này cần thiết cho việc sau này ta chuyển Slave thành Master khi KDC Master chính gặp vấn đề.

Không sử dụng `cat` hoặc copy-paste nội dung của file sang máy chủ Slave. Tốt nhất là nén toàn bộ các file liên quan bằng `tar` và dùng `scp` để đưa sang máy Slave.

Ví dụ: 
```
tar -cvzf /opt/kdc-config.tgz /etc/krb5.conf /etc/krb5kdc/kdc.conf /etc/krb5kdc/.k5.SECURE.LAB /etc/krb5kdc/kadm5.acl /etc/krb5.keytab
scp /opt/kdc-config.tgz kdc2.hoangdh.lab:/opt/
```

### 2.2.6 Cấu hình máy chủ Kerberos Slave
<a name="2.2.6" />

**Chú ý**: Bước thực hiện trên Kerberos Slave

Đặt các file vừa copy từ máy chủ Master vào đúng vị trí tương ứng.

```
cd /opt/
tar -xzf /opt/kdc-config.tgz
```

Tiếp theo, cài đặt gói kpropd để phục vụ cho việc truyền dữ liệu từ master sang slave:

```
apt install krb5-kpropd -y
```

Sau khi sao chép đủ các file ở bước trên, ta tạo một file ACL cho phép nhận dữ liệu từ Master.

> vim /etc/krb5kdc/kpropd.acl

Với nội dung sau:

```
host/kdc1.secure.lab
host/kdc2.secure.lab
```

Và lưu lại file.

### 2.2.7 Khởi động Kerberos trên Slave và kích hoạt kropd
<a name="2.2.7" />

**Chú ý**: Bước thực hiện trên Kerberos Slave

Thực hiện lại bước [này](#allowlog) để cho phép tiến trình được ghi vào `/var/log`.

**Lưu ý:** Kerberos chỉ cho phép một KDC-KAdmin hoạt động trong một thời điểm.

```
systemctl start krb5-kdc
systemctl start krb5-kpropd

systemctl enable krb5-kdc
systemctl enable krb5-kpropd

systemctl stop krb5-kadmin
systemctl disable krb5-kadmin
```

Kiểm tra lại các tiến trình:

```
systemctl status krb5-kdc
systemctl status krb5-kpropd
systemctl status krb5-kadmin
```

### 2.2.8 Xuất dữ liệu trên Kerberos Master và đẩy sang Kerberos Slave
<a name="2.2.8" />

**Chú ý**: Bước thực hiện trên Kerberos Master

Xuất toàn bộ dữ liệu của KDC hiện tại và đẩy sang Slave `kdc2.hoangdh.lab`

```
kdb5_util dump /var/lib/krb5kdc/slave_trans
kprop -f /var/lib/krb5kdc/slave_trans kdc2.hoangdh.lab
```

**Mẹo nhỏ:**

Có thể bật chế độ debug để xem các tiến trình hoạt động:

```
KRB5_TRACE=/dev/stdout kdb5_util dump /var/lib/krb5kdc/slave_trans
KRB5_TRACE=/dev/stdout kprop -f /var/lib/krb5kdc/slave_trans kdc2.hoangdh.lab
```

Thiết lập crontab cho việc đồng bộ dữ liệu giữa Master và Slave

**Script đồng bộ:**

Tạo file mới với nội dung

> vim /opt/kdc-sync.sh

```
#!/bin/sh

kdclist = "kdc2.hoangdh.lab"

kdb5_util dump /usr/local/var/krb5kdc/slave_datatrans

for kdc in ${kdclist}
do
    kprop -f /usr/local/var/krb5kdc/slave_datatrans ${kdc}
done
```

Phân quyền cho script mới tạo:

> chmod +x /opt/kdc-sync.sh

**Thiết lập crontab**

Dựa vào mục đích sử dụng mà ta có thể đặt tần suất sao cho phù hợp. Ở ví dụ này, chúng ta đặt 6h/lần.

> crontab -e

```
...
0 */6 * * * /opt/kdc-sync.sh > /var/log/kdc-sync.log 2>&1
```

### 2.2.9 Kiểm tra dữ liệu trên Kerberos Slave
<a name="2.2.9" />

**Chú ý**: Bước thực hiện trên Kerberos Slave

- Liệt kê thư mục chứa dữ liệu:

```
ls -lah /var/lib/krb5kdc

total 15M
drwx------  4 root root 4.0K Sep  3 13:23 .
drwxr-xr-x 44 root root 4.0K Aug 18 21:23 ..
-rw-------  1 root root 8.5M Sep  3 13:23 from_master
-rw-------  1 root root 6.1M Sep  3 13:23 principal
-rw-------  1 root root 8.0K Sep  3 13:23 principal.kadm5
-rw-------  1 root root    0 Sep  3 12:44 principal.kadm5.lock
-rw-------  1 root root    0 Sep  3 13:23 principal.ok
```

- Liệt kê các principal

```
kadmin.local -q 'listprincs'
```

## 3. Tham khảo
 <a name="3" />

  - https://www.ibm.com/docs/en/streams/4.2.1?topic=authentication-introduction-kerberos
  - https://docs.cloudera.com/HDPDocuments/HDP3/HDP-3.1.0/authentication-with-kerberos/content/kerberos_overview.html
  - https://www.howtoforge.com/how-to-setup-kerberos-server-and-client-on-ubuntu-1804-lts/
  - http://web.mit.edu/kerberos/krb5-current/doc/krb_admins/install_kdc.html
  - https://www.programmerall.com/article/71411298839/
