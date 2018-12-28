﻿---
layout: post
title: "nginx＋php负载均衡集群环境中的session共享方案梳理"
subtitle: "负载均衡、session方案"
author: "other"
header-img: "img/post-bg-css.jpg"
header-mask: 0.4
tags:
  - nginx
  - php
  - session
---
>在网站使用nginx+php做负载均衡情况下，同一个IP访问同一个页面会被分配到不同的服务器上，如果session不同步的话，就会出现很多问题，比如说最常见的登录状态。下面罗列几种nginx负载均衡中session同步的方式。

####1. 不使用session，换用cookie
<span style="font-size:14px">session是存放在服务器端的，cookie是存放在客户端的，我们可以把用户访问页面产生的session放到cookie里面，就是以cookie为中转站。你访问web服务器A，产生了session然后把它放到cookie里面，当你的请求被分配到B服务器时，服务器B先判断服务器有没有这个session，如果没有，再去看看客户端的cookie里面有没有这个session，如果也没有，说明session真的不存在，如果cookie里面有，就把cookie里面的sessoin同步到服务器B，这样就可以实现session的同步了。<br>
说明：这种方法实现起来简单，方便，也不会加大数据库的负担，但是如果客户端把cookie禁掉了的话，那么session就无从同步了，这样会给网站带来损失；cookie的安全性不高，虽然它已经加了密，但是还是可以伪造的。</span>
####2. session存在数据库（MySQL）中
<span style="font-size:14px">PHP可以配置将session保存在数据库中，这种方法是把存放session的表和其他数据库表放在一起，如果mysql也做了集群的话，每个mysql节点都要有这张表，并且这张session表数据表要实时同步。
但是要注意的是：<br>
用数据库来同步session，会加大数据库的IO，增加数据库的负担。而且数据库读写速度较慢，不利于session的适时同步。</span>
####3. session存在memcache或者redis中
<span style="font-size:14px">memcache可以做分布式，php配置文件中设置存储方式为memcache，这样php自己会建立一个session集群，将session数据存储在memcache中。
特别说明：
以这种方式来同步session，不会加大数据库的负担，并且安全性比用cookie大大的提高，把session放到内存里面，比从文件中读取要快很多。但是memcache把内存分成很多种规格的存储块，有块就有大小，这种方式也就决定了，memcache不能完全利用内存，会产生内存碎片，如果存储块不足，还会产生内存溢出。</span>
####4. 采用nginx中的ip_hash机制
<span style="font-size:14px">nginx中的ip_hash技术能够将某个ip的请求定向到同一台后端web机器中，这样一来这个ip下的某个客户端和某个后端web机器就能建立起稳固的session。
也就是说，ip_hash机制能够让某一客户机在相当长的一段时间内只访问固定的后端的某台真实的Web服务器，这样会话就会得以保持，我们在网站页面进行login的时候，
就不会在后面的web服务器之间跳来跳去了，自然也不会出现登陆一次后网站又提醒你没有登陆需要重新登陆的情况；<br>
ip_hash是在upstream配置中定义的：</span>

```shell
upstream nginx.example.com {
   server 192.168.74.235:80;
   server 192.168.74.236:80;
   ip_hash;
}
server {
   listen 80;
   location / {
      proxy_pass
      http://nginx.example.com;
    }
}
```
<span style="font-size:14px;color:blue">ip_hash是容易理解的，但是因为仅仅能用ip这个因子来分配后端web，因此ip_hash是有缺陷的，不能在一些情况下使用：</span>

 - nginx不是最前端的服务器
ip_hash要求nginx一定是最前端的服务器，否则nginx得不到正确ip，就不能根据ip作hash。譬如使用的是squid为最前端，那么nginx取ip时只能得到squid的服务器ip地址，用这个地址来作分流是肯定错乱的。
 - nginx的后端还有其它方式的负载均衡
假如nginx后端又有其它负载均衡，将请求又通过另外的方式分流了，那么某个客户端的请求肯定不能定位到同一台session应用服务器上。这么算起来，nginx后端只能直接指向应用服务器，或者再搭一个squid，然后指向应用服务器。最好的办法是用 location作一次分流，将需要session的部分请求通过ip_hash分流，剩下的走其它后端去。
```shell
----------------------------顺便说一下之前线上用过的nginx负载均衡中的session共享处理方案----------------------------
用的就是上面第三站方式，将session存放在memcached里面。
 
公司的一些网站页面（LNMP框架）涉及到登陆需求（有sessionID），用到了memcache缓存服务，将php的sessionID缓存到memcache里面。
将sessionID放在memcache里后，会加快页面访问速度，页面访问飞快！
 
如果memcache里面存放的只是sessionID，而没有其他业务，那么memcache服务器的内存消耗就不大！
 
首先保障php扩展模块里要支持memcached功能（即一定要安装php的memcached扩展模块）
[root@huanqiu vhosts]# /Data/app/php5.5.1/bin/php -m
[PHP Modules]
..........
memcached
...........
 
遇到问题：
在迁移网站业务的过程中（迁移后使用的是新的memcache机器）
由于php.ini和代码中的memcache连接信息没有及时修改或者没有完全修改过来，导致迁移后的页面访问速度有点缓慢，有点卡！
最后仔细排查，把所有有关memcache连接信息的配置都改过来了，迁移后的页面访问速度就正常了！
 
1）首先部署三台memcache服务器，主机名分别是memcache1.server ，memcache2.server ，memcache3.server ，启动相应的端口。
注意，不用业务应用到的memcache服务端口不能冲突。
比如：业务A用到memcache1-3.server服务器的11021,11022,11023端口，业务B就用到了memcache1-3.server的11031,11032,11033端口
 
部署memcache集群服务
yum安装即可，部署三台memcache1，memcache2，memcache3
启动相应缓存端口
[root@memcache2 ~]# ps -ef|grep memcache
root      6139     1  0 May30 ?        00:00:05 /usr/bin/memcached -d -m 512 -p 11021 -u root -c 4096 -P  /var/lib/memcache/logs/memcached_11021.pid
root      6184     1  0 May30 ?        00:00:05 /usr/bin/memcached -d -m 512 -p 11022  -u root -c 4096 -P /var/lib/memcache/logs/memcached_11022.pid
root      6198     1  0 May30 ?        00:00:05 /usr/bin/memcached -d -m 512 -p 11023 -u root -c 4096 -P  /var/lib/memcache/logs/memcached_11023.pid
root      6214     1  0 May30 ?        00:00:05 /usr/bin/memcached -d -m 512 -p 11031 -u root -c 4096 -P  /var/lib/memcache/logs/memcached_11031.pid
root      6229     1  0 May30 ?        00:00:05 /usr/bin/memcached -d -m 512 -p 11032 -u root -c 4096 -P  /var/lib/memcache/logs/memcached_11032.pid
root      6244     1  0 May30 ?        00:00:05 /usr/bin/memcached -d -m 512 -p 11033 -u root -c 4096 -P  /var/lib/memcache/logs/memcached_11033.pid
 
将上面的程序添加到开机启动/etc/rc.local里面
 
2）在业务机器上应用memcache缓存
     a）比如业务A
     首先在相应的业务服务器上的/etc/hosts里设置主机映射（如果能ping通memcache机器的内网，就用内网）
     #vim /etc/hosts
      192.168.1.23  memcache1.server 
      192.168.1.24  memcache2.server
      192.168.1.25  memcache3 .server 
 
     首先在php的php.ini里面设置memcache缓存
    #vim /Data/app/php/etc/php.ini
       ...............
       [Session]
       session.save_handler = memcached
       session.save_path = "memcache1.server :11021,memcache2.server :11022,memcache3.server :11023"
     
     然后重启php服务
  
     最后在相应的代码程序里使用memcache缓存，比如：
     # vim  main.php
     $config['params']['erp_host']   = 'http://www.xqshijie.com';
                //以下是memcache配置，把相应的参数都换成相应环境下的
                        $config['components']['cache']['class'] = 'system.caching.CMemCache';
                        $config['components']['cache']['useMemcached'] = 'true';
                        $config['components']['cache']['keyPrefix'] = '';
                        $config['components']['cache']['hashKey'] = false;
                        $config['components']['cache']['serializer'] = false;
                        $config['components']['cache']['servers'][0]['host'] = 'memcache1.server';
                        $config['components']['cache']['servers'][0]['port'] = 11021;
                        $config['components']['cache']['servers'][0]['weight'] = 10;
...........................
 
 
     b）业务B
     在相应业务服务器的/etc/hosts里设置主机映射（如果能ping通memcache机器的内网，就用内网）
     #vim /etc/hosts
      192.168.1.23  memcache1.server 
      192.168.1.24  memcache2.server
      192.168.1.25  memcache3 .server 
 
     首先在php的php.ini里面设置memcache缓存
    #vim /Data/app/php/etc/php.ini
       ...............
       [Session]
       session.save_handler = memcached
       session.save_path = "memcache1.server :11031,memcache2.server :11032,memcache3.server :11033"
     
     然后重启php服务
  
     最后在相应的代码程序里使用memcache缓存，比如：
     # vim  main.php
     $config['params']['erp_host']   = 'http://erp.fangfull.com';
                //以下是memcache配置，把相应的参数都换成相应环境下的
                        $config['components']['cache']['class'] = 'system.caching.CMemCache';
                        $config['components']['cache']['useMemcached'] = 'true';
                        $config['components']['cache']['keyPrefix'] = '';
                        $config['components']['cache']['hashKey'] = false;
                        $config['components']['cache']['serializer'] = false;
                        $config['components']['cache']['servers'][0]['host'] = 'memcache1.server';
                        $config['components']['cache']['servers'][0]['port'] = 11031;
                        $config['components']['cache']['servers'][0]['weight'] = 10;
.............................................
 
--------------------------------------------------------------------------------------------------
清理memcache缓存的方法：
1）
默认memcache会监听11221端口，如果想清空服务器上memecache的缓存，大家一般使用的是：
telnet localhost 11211
flush_all
 
2）同样也可以使用：
echo "flush_all" | nc localhost 11211
使用flush_all 后并不是删除memcache上的key，而是置为过期
```
<span style="color:red">------------------------------------php.ini中关于session属性的相关设置-------------------------------------</span>
1）
session.use_cookies：是否在客户端用 cookie 来存放会话 ID，1是开启 ，0是关闭
若session.use_cookies = 1,
sessionid在客户端采用的存储方式，置1代表使用cookie记录客户端的sessionid，同时，\$_COOKIE变量里才会有$_COOKIE[‘PHPSESSIONID’]这个元素存在
一般脚本语言都会原生支持“session机制”，如PHP程序配置：
<br>设置php.ini的session.use_trans_sid = 1，PHP自动在URL里传递session id
<br>设置php.ini的session.use_cookies = 1，使用cookie在客户端保存session id
2）
session.auto start：
将php.ini中的如下选项配置修改即可：
session.auto_start=0
修改成
sessioin.auto_start=1
开启session.auto_start
优点在于，任何时候都不会因忘记执行session_start()或session_start()在程序里的位置不对，而导致错误；
缺点在于，如果你使用的是第三方代码，则必须删去其中的全部 session_start()，否则将不能得到正确的结果。
3）
session的内容存在文件里的话,文件在哪儿?
如果不指定, Linux下默认在 "/tmp"目录。
线上在php.ini配置文件了做了指定，session内容存放在memcache缓存里。
默认session内容是存储在文件里的，即session.save_handler = files
但是我们线上是设置将session内容保存到memcache里的

线上环境下的配置：
[Session]
; Handler used to store/retrieve data.
; http://php.net/session.save-handler
;session.save_handler = files
session.save_handler = memcached
session.save_path = "memcache1.huanqiu.com:11311,memcache1.huanqiu.com:11312,memcache2.huanqiu.com:11311,memcache2.huanqiu.com:11312"
4）
session的生命周期的设置
a）session的默认生命周期是多久?
答:关闭浏览器就失效
原因:因为session_id存在于cookie,而默认情况,cookie关闭浏览器即失败.
b）如何设置session生命周期为30分钟呢?
在php.ini文件里设置session.cookie_lifetime = 1800
线上生产环境下设置的是7天，生命周期是一周
; Lifetime in seconds of cookie or, if 0, until browser is restarted.
; http://php.net/session.cookie-lifetime
session.cookie_lifetime = 604800
5）
session的名字
; Name of the session (used as cookie name).
; http://php.net/session.name
session.name = PHPSESSID
<span style="color:red">------------------------------------session与cookie的简单区别-------------------------------------</span>
session和cookie本质上确实是两个东西，但cookie同时也是session id的载体，cookie保存session id。
1）cookie数据存放在客户的浏览器上，session数据放在服务器上。
session保存在服务器端与浏览器设置无关，cookie在客户端并受浏览器设置限制。
cookie是在你的电脑上保存的,session是在服务器上的. 也就是说你换一个电脑你的cookie就不起作用了, 而session只要你的浏览器不关就还能访问到. 通常的都是两者结合着用的. cookie的话你自己就可以通过对浏览器的设置禁用掉.这样就不起作用了

2）cookie不是很安全，别人可以分析存放在本地的cookie并进行cookie欺骗，考虑到安全应当使用session。
session是服务器端缓存，cookie是客户端缓存。
cookie机制采用的是在客户端保持状态的方案，而session机制采用的是在服务器端保持状态的方案

3）session会在一定时间内保存在服务器上。当访问增多，会比较占用你服务器的性能，考虑到减轻服务器性能方面，应当使用cookie。
session是服务器保持客户端状态信息的方案，一般是保存在服务器中的一块内存中，session超时时间在服务器端进行设置。
cookie是客户端保持用户信息的方案，一般是文件形式保存，cookie清空时间是在客户端浏览器设置。
从开发角度说，session信息可以通过技术方案写到客户端保存，cookie中的用户信息，也可以在用户访问该网站时，通过技术手段自动更新用户的session信息。

4）单个cookie保存的数据不能超过4K，很多浏览器都限制一个站点最多保存20个cookie。

5）建议：将登陆信息等重要信息存放为session；其他信息如果需要保留，可以放在cookie中
<span style="color:red">------------------------------------开启session功能-------------------------------------</span>
开启session功能是很重要的，比如下面一个场景：某个网站程序在测试服务器上调试，首页是ok的，但一到后台去登录就登录不进去，起初怀疑是rewrite规则没有写对，后排查就是因为session功能没有打开引起的！
那么session应该如何开启？
1）编辑php.ini配置文件
session.save_path=文件夹路径      指向任意一个有写权限的目录就行了.
register_globals = On           打开全局变量,如果不打开,你就这样用$_SESSION['sessioname'];但是我本人从来没成功过.
2）重启php服务即可（如果是lamp模式，就重启apache）
<span style="color:red">------------------------------------看一个linux下Session丢失的案例分析-------------------------------------</span>
由于各种原因需要进行代码迁移，迁移后重新搭建php环境，运行代码。最后在登录页面时发现后台不能访问，会直接返回到登录页面，接着对代码进行测试，没有报任何错误，最后排查是因为跳转时session丢失造成的！那么session如何会丢失呢？
发现造成这个原因有这几种：
a）session存储路径（目录）不存在，自然就无法生成session临时文件
b）session存储路径下有没有权限，如果没有，也就不可能存储session数据
c）能正常存session数据，但session存入后被清空

尝试解决的措施：
a）在项目根目录下创建phpinfo.php文件，在文件中写入phpinfo(),运行此文件，查看页面，就可以找到session的存储路径，
b）在服务器上查找session存储路径是否存在，不存在创建存储目录，并分配权限，如果有session存储路径，就查看其是否有权限，没有就分配权限，
c）是否是第三个原因，可在phpinfo.php页面中查找date.timezone是否设置不对，然后在php.ini配置文件中找到date.timezone进行配置


----------
需要清楚知道的：
1）上面在php.ini文件里将session.save_handler修改为memcached，即表示将php的session信息存放到memcache里（前提是安装了memcached扩展），然后在session.save_path处配置连接memcache信息。如：
session.save_handler = memcached
session.save_path = "memcache1.huanqiu.com:11311,memcache1.huanqiu.com:11312,memcache2.huanqiu.com:11311,memcache2.huanqiu.com:11312"

注意：
带d的memcached扩展，则session.save_path配置连接的时候不需要加tcp://
如果是不带d的memcache扩展，则session.save_path配置连接的时候需要加tcp://

2）如果将session.save_handler修改为redis，即表示将php的session信息存放到redis里（前提是安装了php的phpredis扩展），然后在session.save_path处配置redis的connect 地址。如下：
session.save_handler = redis 
session.save_path = "tcp://127.0.0.1:6379"


----------


 




