---
title: codimd搭建之mysql
date: 2020-08-19 23:23:22
tags:
---
>今天需要临时搭建一个md共享文档，记录一下搭建过程中的坑。<br>

[codimd](https://github.com/codimd/container)

1.Install docker and docker-compose, "Docker for Windows" or "Docker for Mac"
2.Run git clone https://github.com/codimd/container.git codimd-container
3.Change to the directory codimd-container directory
4.Run docker-compose up in your terminal
5.Wait until see the log HTTP Server listening at port 3000, it will take few minutes based on your internet connection.
6.Open http://127.0.0.1:3000
先上docker-compose.yml文件：
```yml
version: "3"
services:
  codimd:
    image: nabo.codimd.dev/hackmdio/hackmd:2.0.1
    environment:
      - CMD_DB_URL=mysql://root:passwd@{you_ip}:3306/codimd
      - CMD_USECDN=false
    ports:
      - "3000:3000"
    volumes:
      - upload-data:/home/codimd/app/public/uploads
    restart: always
volumes:
  upload-data: {}
```
此时是用docker搭建codimd,而且mysql数据库也是docker容器，所以you_ip应该是vps的公网地址而不是127.0.0.1。同时，3306端口要放开所有ip访问。