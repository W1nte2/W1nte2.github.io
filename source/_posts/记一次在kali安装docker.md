---
title: 记一次在kali安装docker
date: 2020-03-16 19:42:49
tags:
---

>在一次项目中，被CT大佬吐槽kali版本旧。怒更最新版本，于是又要重新安装docker。网上靠谱的文档不多，于是记录下来方便查询。

# step.1 安装https协议、CA证书、dirmngr

```
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates
sudo apt-get install dirmngr
```

# step.2 添加GPG密钥并添加更新源
```
sudo curl -fsSL https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/debian/gpg | sudo apt-key add -

sudo echo 'deb https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/debian/ buster stable' | sudo tee /etc/apt/sources.list.d/docker.list
```

# step.3 系统更新以及安装docker
```
sudo apt-get update
sudo apt install docker-ce
```

# step.4 启动docker服务器
```
sudo service docker start
```

# step.5 安装compose
```
sudo apt install docker-compose
```
至此安装完毕。`sodu docker version`可查看docker版本信息。