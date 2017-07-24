---
title: mac run docker
date: 2017-04-24 10:48:57
tags:
---

http://www.cpper.cn/2017/02/01/linux/Docker-Compose/


配置参数    说明
image   从镜像的构建容器
build   直接从pwd的Dockerfile来build，而非通过image选项来pull
links   连接到那些容器。每个占一行，格式为SERVICE[:ALIAS],例如 – db[:database] 如果不写别名 就是一致的 这个别名 会写在hosts里 当作服务器名 这样可以直接用于连接容器服务
external_links  连接到该compose.yaml文件之外的容器中，比如是提供共享或者通用服务的容器服务。格式同links
command 替换默认的command命令
ports   导出端口 主机端口:容器端口
expose  导出端口，但不映射到宿主机的端口上。它仅对links的容器开放。格式直接指定端口号即可。
volumes 加载路径作为卷, 主机路径:容器路径:rw(读写属性) 容器路径必须是绝对路径
volumes_from    加载其他容器 服务所有卷
env_file    从一个文件中导入环境变量，文件的格式为RACK_ENV=development
extends 扩展另一个服务，可以覆盖其中的一些选项 看下面 例1
net 容器的网络模式，可以为”bridge”, “none”, “container:[name or id]”, “host”中的一个。
dns 可以设置一个或多个自定义的DNS地址。
dns_search  可以设置一个或多个DNS的扫描域。
orking_dir, entrypoint, user, hostname, domainname, mem_limit, privileged, restart, stdin_open, tty, cpu_shares，docker run  这些命令都是单行的命令,效果和Dockerfile是一样的 看例2