## 容器云练习-基于Docker Compose编排ERP管理系统 

## 一、什么是EPR管理系统,为什么要用ERP系统

1.ERP 就像公司的 **“中央大脑”**，把各个部门（财务、采购、销售、仓库、人事等）的数据和流程全部打通，避免信息孤岛。

2.**没有ERP时**：前台写了个小票，后厨不知道小票内容，只能凭感觉加料，仓库也只能自己记账。当小票丢了、原料没了，这时候也没人知道，会出现信息不对等之类的问题，最后会导致亏钱。

3.**有了ERP后**：当顾客下单后，系统自动扣除库存数量。当库存数量低于警戒线后，自动生成采购单发给供应商。同时销售数据可以实时更新，月底自动计算成本、利润、员工提成。

## 二、资源准备

1.两台linux服务器，这里我用的是CentOS7

2.部署好的k8s集群，在本次实验中使用的是master节点

3.开放8080、6379、9999、3306端口

## 三、实验步骤

**步骤1：获取资源包**

```bash
#从本地服务器下载资源包
curl -O http://192.168.75.10/yum/ERP.tar.gz

#解压资源包
tar -xzvf ERP.tar.gz

#切换至解压后的目录
cd ERP

#查看目录文件
tree
ERP
├── app.jar                #ERP的java可执行包
├── CentOS_7.9.2009.tar    #CentOS7的基本镜像
├── jsh_erp.sql            #ERP的数据库初始化文件
├── nginx                  #nginx相关配置文件
│   ├── app.tar.gz         #ERP的前端页面资源
│   └── nginx.conf         #nginx的代理文件
└── yum                    #需要的安装包
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

步骤2：配置样本源

```bash
cat > local.repo << EOF
[yum]
name=yum
gpgcheck=0
enable=1
baseurl=file:///opt/yum
EOF
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

步骤3：加载基础镜像

```bash
#加载基础镜像
docker load -i CentOS_7.9.2009.tar

#查看是否成功
docker image ls | grep centos
centos   centos7.9.2009   eeb6ee3f44bd   4 years ago    204MB
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

步骤4:构建数据库镜像文件erp-mysql:v1.0

```bash
cat > Dockerfile-mariadb << EOF
#加载原始镜像
FROM centos:centos7.9.2009

#作者名
MAINTAINER 17-60

#配置样本源
RUN rm -rf /etc/yum.repos.d/*
COPY local.repo /etc/yum.repos.d/local.repo
COPY yum /opt/yum

#配置数据库
ENV LC_ALL en_US.UTF-8
COPY init.sh /opt/init.sh
COPY jsh_erp.sql /opt/jsh_erp.sql
RUN yum install -y mariadb mariadb-server
RUN chmod a+x /opt/init.sh
RUN /opt/init.sh
#声明端口
EXPOSE 3306
#开启命令
CMD ["mysqld_safe","--user=root"]
EOF
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

```bash
#创建数据库初始化脚本
cat > init.sh << EOF
#!/bin/bash
mysql_install_db --user=root
mysqld_safe --user=root &
sleep 8
mysqladmin -uroot -p '123456'
mysql -uroot -p123456 -e "grant all on *.* to 'root'@'%' identified by '123456';flush privileges;"
mysql -uroot -p 123456 -e "create databases jsh_erp; use jsh_erp; source /opt/jsh_rep.sql;"
EOF
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

```bash
#创建镜像
docker build -t erp-mysql:v1.0 -f Dockerfile-mariadb .
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

步骤5:构建内存键值数据库erp-redis:v1.0镜像

```bash
#创建Dockerfile_redis
cat > Dockerfile_redis << EOF
FROM centos:centos7.9.2009
MAINTAINER 17-60
RUN rm -rf /etc/yum.repos.d/*
COPY local.repo /etc/yum.repos.d/local.repo
COPY yum /opt/yum
RUN yum install redis -y
RUN sed -i 's/127.0.0.1/0.0.0.0/g' /erc/redis.conf
RUN sed -i 's/protected-mode yes/protected-mode no/g' /etc/redis.conf
RUN echo 'requirepass 123456' >> /etc/redis.conf
EXPOSE 6379
CMD ["/usr/bin/redis-server","/etc/redis.conf"]
EOF
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

```bash
#创建镜像
docker build -t erp-redis:v1.0 -f Dockerfile-redis .
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

步骤6:构建web服务器erp-nginx:v1.0镜像

```bash
cat > Dockerfile-nginx << EOF
FROM centos:centos7.9.2009
MAINTAINER 17-60
RUN rm -rf /etc/yum.repos.d/*
COPY local.repo /etc/yum.repos.d/local.repo
COPY yum /opt/yum
RUN yum install -y nginx
ADD nginx/app.tar.gz /
COPY nginx/nginx.conf /etc/nginx/nginx.conf
EXPOSE 80
CMD ["nginx","-g","daemon off;"]
EOF
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

```bash
#创建镜像
docker build -t erp-nginx:v1.0 -f Dockerfile-nginx .
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

步骤7:构建erp服务erp-server:v1.0镜像

```bash
#创建Dockerfile-erp
cat > Dockerfile-erp << EOF
FROM centos:centos7.9.2009
RUN rm -rf /etc/yum.repos.d/*
COPY local.repo /etc/yum.repos.d/local.repo
COPY yum /opt/yum
RUN yum install -y java-1.8.0-openjdk java-1.8.0-openjdk-devel
COPY app.jar /jshERP_boot/
COPY start.sh /opt/start.sh
RUN chmod a+x /opt/start.sh
EXPOSE 9999
CMD ["/bin/bash","/opt/start.sh"]
EOF
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

```bash
#创建启动脚本
cat > start.sh << EOF
#!/bin/bash
nohup java -jar /jsh_ERP-boot/app.jar
EOF
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

```bash
#创建镜像
docker build -t erp-server:v1.0 -f Dockerfile-erp .
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

步骤8:构建docker-compose编排程序

```bash
#创建docker-compose
cat > docker-compose << EOF
version: '3'
services:
  erp-mysql:
    image: erp-mysql:v1.0
    container_name: erp-mysql
    ports:
    - 3306:3306
    restart: always

  erp-redis:
    image: erp-redis:v1.0
    container_name: erp-redis
    ports:
    - 6379:6379
    restart: always

  erp-server:
    image: erp-server:v1.0
    container_name: erp-server
    environment:
      DB_USERNAME: root
      DB_PASSWORD: 123456
      REDIS_USERNAME: root
      REDIS_PASSWORD: 123456
    links:
    - erp-mysql
    - erp-redis
    ports:
    - 9999:9999
    restart: always

  erp-nginx:
    image: erp-nginx:v1.0
    container_name: erp-nginx
    ports:
    - 8080:80
    links:
    - erp-server:erp-server
    restart: always
EOF
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

```bash
#启动编排程序
docker-compose up -d
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

```bash
#验证是否成功
docker ps | grep erp
f626a5a6843a   erp-nginx:v1.0                       "nginx -g 'daemon of…"   3 hours ago   Up 3 hours            0.0.0.0:8080->80/tcp, :::8080->80/tcp       erp-nginx
93d94d819a25   erp-server:v1.0                      "/bin/bash /opt/star…"   3 hours ago   Up 1 second           0.0.0.0:9999->9999/tcp, :::9999->9999/tcp   erp-server
91cdc9f55ec1   erp-mysql:v1.0                       "mysqld_safe --user=…"   3 hours ago   Up 3 hours            0.0.0.0:3306->3306/tcp, :::3306->3306/tcp   erp-mariadb
b1101ff8c93f   erp-redis:v1.0                       "/usr/bin/redis-serv…"   3 hours ago   Up 3 hours            0.0.0.0:6379->6379/tcp, :::6379->6379/tcp   erp-redis
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

到此实验完成

## 四、最终验证

登录http://<服务器IP>:8080

账号:admin

密码:123456

![img](https://i-blog.csdnimg.cn/direct/167d93afeb21469e8fe427479ed02cb7.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)编辑

![img](https://i-blog.csdnimg.cn/direct/15e1c5f729e9412a8eb956fb9db24a8f.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)编辑

登录成功说明成功部署了

## 五、总结

本次实验通过Docker + docker-compose技术，将jshERP系统从零部署到Centos7的k8s服务上，实现经销管理系统的容器化部署

| 容器名     | 镜像            | 端口    | 技术栈             | 职责                    |
| ---------- | --------------- | ------- | ------------------ | ----------------------- |
| erp-mysql  | erp-mysql:v1.0  | 3306    | Mariadb            | 数据持久化存储          |
| erp-redis  | erp-redis:v1.0  | 6379    | Redis              | 高速缓存加速            |
| erp-nginx  | erp-nginx:v1.0  | 8080-80 | Nginx              | 前端页面展示 + 反向代理 |
| erp-server | erp-server:v1.0 | 9999    | Java + Spring Boot | 后端业务逻辑处理        |