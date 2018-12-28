# Centos7x系统--LAMP环境搭建
---
本文介绍在centos7x系统搭LAMP服务器的步骤：
## 安装Apache ##
 **1.使用yum安装apache，执行**
```shell
Sudo yum install httpd -y
```
**2.安装完成后，输入命令whereis httpd查看安装位置**
```shell
whereis httpd
```
![cmd-result](https://i.loli.net/2018/12/25/5c21a4c809082.png)
**3.输入rpm -qi httpd命令查看Apache的版本信息**
```shell
rpm -qi httpd
```
![cmd-result](https://i.loli.net/2018/12/25/5c21a59548ce5.png)
**4.启动、重启、停止服务命令**
> * 启动Apache服务：systemctl start httpd.service
> * 重启Apache：systemctl restart httpd.service
> * 重启Apache：systemctl restart httpd.service

**5.安装目录介绍**
Apache默认将网站的根目录指向/var/www/html
默认的主配置文件/etc/httpd/conf/httpd.conf
**6.修改默认配置httpd.conf**
执行
```shell
vi /etc/httpd/conf/httpd.conf
```
找到以下内容：
![cmd-result](https://i.loli.net/2018/12/27/5c244194353cb.png)
将此处的AllowOverride None修改为AllowOverride All
**7.开放80端口（针对centos 7以后的版本）**
> * 开启端口：firewall-cmd --zone=public --add-port=80/tcp --permanent
> * 重启防火墙：firewall-cmd --reload
> * 查看状态：firewall-cmd --state
> * 查看80端口是否打开：firewall-cmd--query-port=80/tcp，返回yes表示打开

**8.在Apache启动的情况下，在浏览器输入http://服务器IP，查看是否进入HTTP样本网页，如果能进入如下页面，说明Apache服务器搭建成功。**
![cmd-result](https://i.loli.net/2018/12/27/5c24423ad5bd1.png)

## 安装MySQL ##
**1. 检查系统中是否已安装MySQL**
```shell
Rpm -qa | grep mysql
```
返回空的话，就说嘛没有按照Mysql，如果有的话就全部卸载
```shell
yum -y remove+数据库名称
```
**2. 卸载Mariadb**
注意：在新版本的CentOS7中，默认的数据库已更新为了Mariadb，而非 MySQL，所以执行 yum install mysql 命令只是更新Mariadb数据库，并不会安装 MySQL 。
查看已安装的Mariadb数据库版本：rpm -qa|grep -i mariadb
卸载已安装的Mariadb数据库rpm -qa|grep mariadb|xargs rpm -e --nodeps
再次查看已安装的Mariadb库是否卸载完成
![cmd-result](https://i.loli.net/2018/12/27/5c2446e5c8cff.png)
**3. 安装libaio**
> MySQL依赖libaio，所以先安装libaio
> yum search libaio # 检索相关信息
> yum install libaio # 安装依赖包

![cmd-result](https://i.loli.net/2018/12/27/5c244737d01a1.png)
**4. 下载MySQL Yum Repository**
```shell
wget http://dev.mysql.com/get/mysql-community-release-el7-5.noarch.rpm
```
**5. 添加MySQL Yum Repository**
```shell
yum localinstall mysql-community-release-el7-5.noarch.rpm
```
![cmd-result](https://i.loli.net/2018/12/27/5c2447ac690f7.png)
**6. 验证下是否添加成功**
```shell
yum localinstall mysql-community-release-el7-5.noarch.rpm
```
![cmd-result](https://i.loli.net/2018/12/27/5c2447eeb472d.png)
**7. 选择要启用的MySQL版本**
查看MySQL版本，执行：
```shell
yum repolist all | grep mysql
```
![cmd-result](https://i.loli.net/2018/12/27/5c24488771564.png)
可以看到 5.5， 5.7 版本是默认禁用的，因为现在最新的稳定版是 5.6
```shell
yum repolist enabled | grep mysql //查看当前的启动的MySQL版本
```
**8.　通过Yum来安装MySQL**
```shell
执行 yum install mysql-server 
```
遇到，输入 y 继续，执行完成会提示“完毕！”。此时MySQL 安装完成，它包含了 mysql-community-server、mysql-community-client、mysql-community-common、mysql-community-libs 四个包。
```shell
rpm -qi mysql-community-server.x86_64 0:5.6.24-3.el7
whereis mysql
```
可以看到 MySQL 的安装目录是 /usr/bin/
![cmd-result](https://i.loli.net/2018/12/27/5c244935ad64c.png)
**9.　启动和关闭 MySQL Server**
> * 启动 MySQL Server：systemctl start mysqld
> * 查看 MySQL Server 状态：systemctl status  mysqld
> * 关闭 MySQL Server：systemctl stop mysqld
> * 设置MySQL开机启动：systemctl enable mysqld

![cmd-result](https://i.loli.net/2018/12/27/5c245324ce987.png)
**10.　防火墙设置**
远程访问 MySQL， 需开放默认端口号 3306.
```shell
firewall-cmd --permanent --zone=public --add-port=3306/tcp
firewall-cmd --permanent --zone=public --add-port=3306/udp
```
这样就开放了相应的端口。
```shell
执行 firewall-cmd --reload 
```
![cmd-result](https://i.loli.net/2018/12/27/5c2453b2a93fb.png)
**11.　设置MySQL密码（也可直接进行下一步MySQL安全设置修改root密码）**
mysql5.6 安装完成后，它的 root 用户的密码默认是空的，我们需要及时用 mysql 的 root 用户登录（第一次直接回车，不用输入密码），并修改密码。
```shell
mysql -u root
mysql> use mysql;
mysql> update user set password=PASSWORD("这里输入root用户密码") where User='root';
mysql> flush privileges; 
```
**12.　MySQL安全设置**
服务器启动后，可以执行
```shell
mysql_secure_installation;
```
![cmd-result](https://i.loli.net/2018/12/27/5c245428256ad.png)
此时输入 root 密码（密码为步骤11设置的root账号密码），接下来，为了安全，MySQL 会提示你重置 root 密码，移除其他用户账号，禁用 root 远程登录，移除 test 数据库，重新加载 privilege 表格等，你只需输入 y 继续执行即可。至此，整个 MySQL 安装完成。
## 安装PHP 5.6 ##
**1.　安装epel-release**
```shell
yum -y install epel-release 
```
**2.　安装PHP 5.6**
```shell
rpm -Uvh https://mirror.webtatic.com/yum/el7/webtatic-release.rpm
```
可以使用yum list查看一下可安装的包：
```shell
执行 yum list | grep php
```
安装php插件，执行
```shell
yum install php56w php56w-mysql php56w-gd libjpeg* php56w-ldap php56w-odbc php56w-pear php56w-xml php56w-xmlrpc php56w-mbstring php56w-bcmath php56w-pecl-imagick
```
![cmd-result](https://i.loli.net/2018/12/27/5c2454bfe1126.png)
**3.　验证安装**
终端命令：PHP -v，显示当前PHP版本。
查看已安装的php扩展：rpm -qa|grep php
#### 4.　修改php配置文件
```shell
vi /etc/php.ini
upload_max_size = 512M(最大上传文件大小)
session.use_only_cookies=0（是否仅仅使用cookie在客户端保存会话sessionid，这个选项可以使管理员禁止用户通过URL来传递id）
```
## 安装memcache服务和扩展 ##
**1.　安装memcache服务**
```shell
yum install memcached
```
**2.　设置memcached开机启动**
```shell
chkconfig memcached on
```
**3.　立即启动memcached服务**
```shell
systemctl start memcached
```
**4.　关闭SELinuxsudo setsebool httpd_can_network_connect=1(由于SELinux的安全配置. 通过此命令允许httpd对本机其他服务的访问)**
**5.　安装libmemcached扩展**
```shell
执行 yum -y install libmemcached
```
**6.　安装php-pecl-memcache扩展**
```shell
yum -y install php56w-pecl-memcache
```
**7.　重启Apache服务**
```shell
systemctl restart httpd
```
**8.　phpinfo();查看是否已安装memcache服务**
![cmd-result](https://i.loli.net/2018/12/27/5c24560cef848.png)

