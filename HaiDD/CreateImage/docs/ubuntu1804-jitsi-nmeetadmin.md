# Tài liệu hướng dẫn đóng image Jitsi tích hợp trang quản trị Meetnow Admin trên Ubuntu 18.04

## Thực hiện:
- Đóng image trên KVM

## Thông số cài đặt:
- OS: Ubuntu 18.04 LTS

- Thông số phiên bản các service của Jitsi:
    - jicofo=1.0-541-1 
    - jitsi-meet=2.0.4384-1
    - jitsi-meet-prosody=1.0.3969-1 
    - jitsi-meet-turnserver=1.0.3969-1 
    - jitsi-meet-web=1.0.3969-1 
    - jitsi-meet-web-config=1.0.3969-1 
    - jitsi-videobridge2=2.1-164-gfdce823f-1


# Đóng image
## 1. Thiết lập cơ bản
### Setup cơ bản Ubuntu:
- Bật máy ảo, truy cập bằng tài user ubuntu và bắt đầu thực hiện cài đặt môi trường.
    ```
    sudo su

    # Đặt mật khẩu cho root
    passwd
    Enter new UNIX password: <root_passwd>
    Retype new UNIX password: <root_passwd>
    ```


- Cấu hình cho phép ssh bằng user root `/etc/ssh/sshd_config`
    ```
    sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/'g /etc/ssh/sshd_config

    sed -i 's/#PasswordAuthentication yes/#PasswordAuthentication yes/'g /etc/ssh/sshd_config

    service sshd restart
    ```

- Disable firewalld
    ```
    systemctl disable ufw
    systemctl stop ufw
    systemctl status ufw
    ```

- Logout ra khỏi VM:
    ```
    logout
    ```

- Login lại bằng user root

- Xóa user ubuntu
    ```
    userdel ubuntu
    rm -rf /home/ubuntu
    ```

- Đổi timezone về Asia/Ho_Chi_Minh
    ```
    timedatectl set-timezone Asia/Ho_Chi_Minh
    ```

- Bổ sung env locale
    ```
    echo "export LC_ALL=C" >>  ~/.bashrc
    ```

- Disable ipv6
    ```
    echo "net.ipv6.conf.all.disable_ipv6 = 1" >> /etc/sysctl.conf 
    echo "net.ipv6.conf.default.disable_ipv6 = 1" >> /etc/sysctl.conf 
    echo "net.ipv6.conf.lo.disable_ipv6 = 1" >> /etc/sysctl.conf
    sysctl -p

    cat /proc/sys/net/ipv6/conf/all/disable_ipv6
    ```
    **OUTPUT**: 1: OK, 0: Not OK

- Kiểm tra và xóa swap:
    - Kiểm tra swap:
        ```
        cat /proc/swaps
        Filename                                Type            Size    Used    Priority
        /swap.img                               file            4038652 0       -2
        ```

    - Xóa swap:
        ```
        swapoff -a
        rm -rf /swap.img
        ```

    - Xóa cấu hình swap file trong file /etc/fstab
        ```
        sed -Ei '/swap.img/d' /etc/fstab
        ```

    - Kiểm tra lại:
        ```
        free -m
        ```

## 2. Cài đặt các gói cần thiết
- Update
    ```
    apt-get update -y 
    apt-get upgrade -y 
    apt-get dist-upgrade -y
    apt-get autoremove -y
    ```

- Cấu hình để instance báo log ra console và đổi name Card mạng về eth* thay vì ens, eno
    ```
    sed -i 's|GRUB_CMDLINE_LINUX=""|GRUB_CMDLINE_LINUX="net.ifnames=0 biosdevname=0 console=tty1 console=ttyS0"|g' /etc/default/grub

    update-grub
    ```

### 2.1. Cấu hình network sử dụng ifupdown thay vì netplan
- Disable netplan
    ```
    apt-get --purge remove netplan.io -y
    rm -rf /usr/share/netplan
    rm -rf /etc/netplan

    apt-get update
    apt-get install -y ifupdown
    ```

- Tạo file interface
    ```
    cat << EOF > /etc/network/interfaces
    auto lo
    iface lo inet loopback
    auto eth0
    iface eth0 inet dhcp
    EOF
    ```

- Reboot máy, kiểm tra card eth0

### 2.2. Để sau khi boot máy ảo, có thể nhận đủ các NIC gắn vào:
```
apt-get install netplug -y

wget https://raw.githubusercontent.com/danghai1996/thuctapsinh/master/HaiDD/CreateImage/scripts/netplug_ubuntu -O netplug

mv netplug /etc/netplug/netplug

chmod +x /etc/netplug/netplug
```

### 2.3. Disable snapd service:
Kiểm tra snap:
```
df -H

Filesystem      Size  Used Avail Use% Mounted on
udev            2.1G     0  2.1G   0% /dev
tmpfs           414M  6.1M  408M   2% /run
/dev/vda2        22G  2.2G   18G  11% /
tmpfs           2.1G     0  2.1G   0% /dev/shm
tmpfs           5.3M     0  5.3M   0% /run/lock
tmpfs           2.1G     0  2.1G   0% /sys/fs/cgroup
/dev/loop0       93M   93M     0 100% /snap/core/7270
tmpfs           414M     0  414M   0% /run/user/0
```

List danh sách snap
```
snap list

Name  Version    Rev   Tracking       Publisher   Notes
core  16-2.39.3  7270  latest/stable  canonical*  core
```

Remove snapd package
```
apt purge snapd -y
```
Kiểm tra lại
```
df -H
Filesystem      Size  Used Avail Use% Mounted on
udev            2.1G     0  2.1G   0% /dev
tmpfs           414M  6.1M  408M   2% /run
/dev/vda2        22G  2.1G   18G  11% /
tmpfs           2.1G     0  2.1G   0% /dev/shm
tmpfs           5.3M     0  5.3M   0% /run/lock
tmpfs           2.1G     0  2.1G   0% /sys/fs/cgroup
tmpfs           414M     0  414M   0% /run/user/0
```

> ## Snapshot VM -> OS_Ubuntu1804

## 3. Cài đặt Jitsi
### 3.1. Đặt hostname
```
hostnamectl set-hostname jitsimeet
bash
```

### 3.2. Cài đặt OpenJDK Java Runtime Environment (JRE) 8
Enable repo universe nếu chưa được kích hoạt
```
sudo add-apt-repository universe
```

Cài đặt OpenJDK JRE 8:
```
sudo apt install -y openjdk-8-jre-headless
```

Kiểm tra:
```
java -version
```

Cấu hình môi trường JAVA_HOME
```
echo "JAVA_HOME=$(readlink -f /usr/bin/java | sed "s:bin/java::")" | sudo tee -a /etc/profile
source /etc/profile
```

### 3.3. Cài đặt Nginx
```
sudo apt install -y nginx
sudo systemctl start nginx.service
sudo systemctl enable nginx.service
```

### 3.4. Cài đặt Jitsi Meet
Cài Jitsi repo:
```
cd
wget -qO - https://download.jitsi.org/jitsi-key.gpg.key | sudo apt-key add -

sudo sh -c "echo 'deb https://download.jitsi.org stable/' > /etc/apt/sources.list.d/jitsi-stable.list"

sudo apt update -y
```

Cài đặt Jitsi theo phiên bản đã định sẵn ở trên:
```
apt-get install jicofo=1.0-541-1 jitsi-meet=2.0.4384-1 jitsi-meet-prosody=1.0.3969-1 jitsi-meet-turnserver=1.0.3969-1 jitsi-meet-web=1.0.3969-1 jitsi-meet-web-config=1.0.3969-1 jitsi-videobridge2=2.1-164-gfdce823f-1 -y
```

Trong quá trình cài đặt, sẽ được yêu cầu điền hostname. Tại đó, điền IP máy chủ

Sau đó, ta sẽ được hỏi về SSL cert: -> Chọn `Generate a new self-signed certificate (You will later get a chance to obtain a Let’s Encrypt certificate)`


Restart các service:
```
systemctl start prosody
systemctl enable prosody

systemctl start jicofo
systemctl enable jicofo

systemctl start jitsi-videobridge2.service
systemctl enable jitsi-videobridge2

systemctl restart nginx
```

Truy cập bằng IP của máy ảo

<img src="..\images\Screenshot_76.png">

## 4. Cài đặt các tính năng thêm cho Jitsi
### 4.1. Tắt tự động đặt tên phòng
Vào file `/usr/share/jitsi-meet/interface_config.js`, tìm đến dòng `GENERATE_ROOMNAMES_ON_WELCOME_PAGE` sửa giá trị true thành `false`.
```
sed -i 's|GENERATE_ROOMNAMES_ON_WELCOME_PAGE: true,|GENERATE_ROOMNAMES_ON_WELCOME_PAGE: false,|g' /usr/share/jitsi-meet/interface_config.js
```

### 4.2. Yêu cầu người dùng nhập tên trước khi vào phòng họp
Vào file `/etc/jitsi/meet/<domain>-config.js`. Ở đây là IP: `/etc/jitsi/meet/10.10.30.188-config.js`, uncomment dòng sau: `requireDisplayName: true,`
```
sed -i 's|// requireDisplayName: true,|requireDisplayName: true,|g' /etc/jitsi/meet/10.10.30.188-config.js
```

### 4.3. Cấu hình để người dùng mobile truy cập được vào bằng trình duyệt
Vào file `/etc/jitsi/meet/<domain>-config.js`. Ở đây là IP: `/etc/jitsi/meet/10.10.30.188-config.js`, uncomment và sửa dòng `disableDeepLinking: true,`
```
sed -i 's|// disableDeepLinking: false,|disableDeepLinking: true,|g' /etc/jitsi/meet/10.10.30.188-config.js
```

### 4.4. Tắt âm thanh của người vào phòng khi họ mới vào
Vào file `/etc/jitsi/meet/<domain>-config.js`. Ở đây là IP: `/etc/jitsi/meet/10.10.30.188-config.js` uncomment và sửa dòng `startWithAudioMuted`
```
sed -i 's|// startWithAudioMuted: false,|startWithAudioMuted: true,|g' /etc/jitsi/meet/10.10.30.188-config.js
```

### 4.5. Tắt camera của người vào phòng khi họ mới vào
Vào file `/etc/jitsi/meet/<domain>-config.js`. Ở đây là IP: `/etc/jitsi/meet/10.10.30.188-config.js`, uncomment và sửa dòng `startWithVideoMuted`
```
sed -i 's|// startWithVideoMuted: false,|startWithVideoMuted: true,|g' /etc/jitsi/meet/10.10.30.188-config.js
```

### 4.6. Tắt audio level
Vào file `/etc/jitsi/meet/<domain>-config.js`.  Ở đây là IP: `/etc/jitsi/meet/10.10.30.188-config.js`, uncomment và sửa dòng `disableAudioLevels`
```
sed -i 's|// disableAudioLevels: false,|disableAudioLevels: true,|g' /etc/jitsi/meet/10.10.30.188-config.js
```

### 4.7. Tắt làm mờ background video
Vào file `/usr/share/jitsi-meet/interface_config.js`, sửa dòng: `DISABLE_VIDEO_BACKGROUND`
```
sed -i 's|DISABLE_VIDEO_BACKGROUND: false,|DISABLE_VIDEO_BACKGROUND: true,|g' /usr/share/jitsi-meet/interface_config.js
```

Sau đó, xóa `videobackgroundblur` tại mục `TOOLBAR_BUTTONS` để bỏ chức năng làm mờ background đi
```
sed -i "s|'videobackgroundblur',||g" /usr/share/jitsi-meet/interface_config.js
```

### 4.8. Tắt hoạt ảnh feedback
Sửa các dòng ở file `/usr/share/jitsi-meet/interface_config.js`
```
sed -i 's|DISABLE_FOCUS_INDICATOR: false,|DISABLE_FOCUS_INDICATOR: true,|g' /usr/share/jitsi-meet/interface_config.js

sed -i 's|DISABLE_DOMINANT_SPEAKER_INDICATOR: false,|DISABLE_DOMINANT_SPEAKER_INDICATOR: true,|g' /usr/share/jitsi-meet/interface_config.js
```

### 4.9. Cài đặt ngôn ngữ mặc định là tiếng Việt
Đổi ngôn ngữ mặc định thành tiếng Việt:
```
sed -i "s|// defaultLanguage: 'en',|defaultLanguage: 'vi',|g" /etc/jitsi/meet/10.10.30.188-config.js
```

Tải file tiếng việt đã được custom lại tại: https://github.com/cloud365vn/NH-Jitsi/blob/master/lang/main-vi.json

Thay thế nội dung file `/usr/share/jitsi-meet/lang/main-vi.json` bằng file tải về
```
cat file_main-vi.json-custom  > /usr/share/jitsi-meet/lang/main-vi.json

# Xóa file custom
rm -f file_main-vi.json-custom
```