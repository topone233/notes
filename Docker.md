

## 常用命令

```shell
// 查看容器运行日志
docker logs 容器ID

docker rm 容器ID

docker ps [-a]

docker images

docker run
docker start
```



### 安装RabbitMQ

#### 1. 搜索镜像

```shell
docker search rabbitmq
```

#### 2.拉取镜像

```shell
docker pull rabbitmq
```

#### 3.构建容器并运行

- -d 后台运行
- --name 指定容器名
- -p 指定服务运行的端口（5672 应用访问端口，15672 Web端管理页面端口）
- -v 映射目录或文件（如果不存在则创建，无需手动创建）
- -hostname 主机名 （RabbitMQ的一个重要注意事项是它根据所谓的 “节点名称” 存储数据，默认为主机名）
- -e 指定环境变量；（RABBITMQ_DEFAULT_VHOST：默认虚拟机名；RABBITMQ_DEFAULT_USER：默认的用户名；RABBITMQ_DEFAULT_PASS：默认用户名的密码）

```shell
#不指定用户名密码，默认为guest/guest
docker run --name=rabbitmq-server -p 15672:15672 -p 5672:5672 -d rabbitmq:management

#指定用户名密码
docker run --name=rabbitmq-server -p 15672:15672 -p 5672:5672 -e RABBITMQ_DEFAULT_USER=admin -e RABBITMQ_DEFAULT_PASS=admin -d rabbitmq:management
```

#### 4.安装管理界面插件

```shell
# 查看正在运行的容器
docker ps
# 上一步取得rabbitmq容器id
docker exec -it 容器id /bin/bash

# 安装插件
rabbitmq-plugins enable rabbitmq_management

# 退出
exit
```

#### 



## MySQL

-v 参数可以多次使用，每次映射一个目录

-e mysql用户密码

```shell
docker run -d -e MYSQL_ROOT_PASSWORD=admin --name mysql -v /home/admin/data/mysql/my.cnf:/etc/mysql/my.cnf -v /home/admin/data/mysql/data:/var/lib/mysql -p 3306:3306 mysql 
```

