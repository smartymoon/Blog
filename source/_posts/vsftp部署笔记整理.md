---
title: vsftp部署笔记整理
date: 2019-07-16 10:20:00
tags: linux
---

公司需要一个公共的文件管理系统，这里采用ftp。服务器为阿里云ecs, 系统为 ubuntu。在搭建过程中遇到很多坑，因此做一个笔记。vsftp 安装后就可以使用，默认使用系统用户登陆，同时配置了最低权限，这应该是出于安全的考虑，我们这里需要搭建一个多用户，多权限，多目录的环境，需要做很多额外的配置。
- ECS 配置
- 虚拟用户配置
- 用户权限配置
- 客户端配置

#### 1. ECS 配置
阿里云 ECS 有安全组功能，限制了服务器对外开放的接口。而 ftp 服务的端口为 21，这个需要我们特别打开，但是仅打开21是不够用的，这个21仅仅是用于传输命令用的，当我们需要上传下载的时候，实际上是使用的其它端口，这里我们手动指定为 8000 到 9000，配置方法后面会有。因此在 ECS 中我们要打开两组端口：21，8000-9000。

#### 2. 虚拟用户配置
此配置是服务搭建过程中的难点，步骤多。给第一个员工开 linux 的系统用户是不方便的，我们采用虚拟用户，所有的虚拟用户的帐户密码配置在一个文件中，我们在系统中新增一个传统用户，用于代理这些虚拟用户，也就是说虚拟用户登陆后实际上登录的都是这一个代理用户 `vuser`。
```
useradd -m -s /usr/sbin/nologin vuser
```
`-m` 表示创建家目录，虚拟用户的默认fpt根目录就是这个目录。  
`nologin` 表示这个用户不能直接登录，因此也不需要设置密码。

安装 vsftpd, 这个没什么好说的
```
apt install vsftpd
```
安装成功后，其配置文件于 `/etc/vsftpd.conf`, 这个文件很重要，后续要做很多更改。

接下来创建一个文件夹，用于存放我们的自定义文件， 首先创建的是虚拟用户账户文件 `users.txt`
```
mkdir /etc/vsftp
vim /etc/vsftp/users.txt
```
`users.txt` 的文件格式很二，一行用户名一行密码
```
// users.txt 内容
zhangsan
123
lisi
123
wangwu
123
```
接下来需要把这个文件变成 users.db 使用的工具是 `db_load`, 此命令存在于 `db-util` 这个包中
```
apt install db-util
# 转换命令如下，会生成 users.db, 我们做用户有登陆是用它做验证
db_load -T -t hash -f /etc/vsftp/users.txt /etc/vsftp/users.db
```
在 `vsftpd.conf` 中有这样一个配置， `pam_service_name=vsftpd`
它的意思是，在做用户验证时，我们使用的是 `/etc/pam.d/vsftpd` 这个文件,
首先禁用系统用户的fpt登陆，我们把它里面的内容全部注释掉。为了让虚拟用户登陆，我们把刚刚创建的 `users.db` 配置到这里。
```
// 注意 users 后面我们要省略掉 .db 
auth required pam_userdb.so db=/etc/vsftp/users
account required pam_userdb.so db=/etc/vsftp/users
```
此时，虚拟用户准备完毕, 我们开始配置 `/etc/vsftpd.conf`
```
guest_enable=YES // 开启虚拟用户功能
guest_username=vuser // 代理的真实用户，是我们上面刚刚创建的
pasv_enable=YES // 开启 ftp 被动模式
pasv_addr_resolve=YES // 欺骗地址？不大清楚

// 当 ftp 传输文件时，会告诉客户端，你来我某个端口的获取文件  
// 阿里云的 ECS 现在全部使用专有网络，意味着服务器并不知道自己的公网IP 
// 些时，它会把自己的内网IP加端口告诉客户端，客户端当然访问不到 ECS 的 
// 内网IP，所以我们要手动把 ECS 的外网IP写在这里
pasv_address=你的.服务器.外网IP
pasv_min_port=8000 // 传输文件用端口范围，最小
pasv_max_port=9000 // 传输文件用端口范围，最大
listen=YES // 开启监听
listen_ipv6 // 关系IPV6监听，不然会默认使用它，出错
anonymous_enable=NO // 关闭匿名用户登陆
write_enable=YES // 可写
allow_writeabel_chroots=YES // 允许在可写文件夹中改变目录
```
此时我们的虚拟用户就可以登录了

#### 3. 用户权限配置
此时用户仅仅是能登录了，什么权限还没有，基本上什么都不能做。这些操作包括：上传，下载，创建目录，删除文件，指定根目录。我们有两个地方配置这些权限。  
1. `/etc/vsftpd.conf` 对所有虚拟用户生效
2. `/etc/vsftp/users_config` 我们自己创建的文件夹，以虚拟用户名作为文件名，权限配置方法同 `vsftpd.conf`

首先，我们在 `vsftpd.conf` 中指定虚拟用户配置文件夹的位置,让 vsftpd 知道去哪里找
```
user_config_dir=/etc/vsftp/users_config
```
ftp 权限配置表如下：
```
anon_mkdir_wirte_enable=YES // 允许创建目录
anon_other_write_enable // 允许删除操作
anon_upload_enable // 允许上传
anon_world_readable_only=NO // 允许下载
local_root=/home/vuser/zhangsan // 根目录
```
要注意的是，操作能成功有两个条件要同时满足。  
1. ftp 权限满足
2. 代理系统用户 `vuser` 拥有相关权限，也就是常说的 `rwx`

#### 4. 客户端配置
我这里使用的是 `winscp`, 要把会话的编码从 `utf8 自动`，改成 `utf8 开启动`， 不然会出现乱码的问题 
