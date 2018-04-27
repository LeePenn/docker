# docker
Deploy docker and harbor

## 基于自签发证书的HTTPS安装Notary&Clair

### 环境

| NAME | INFO |
| ---------- | -------- |
| CentOS | 7.4 |
| docker | 1.13.1 |
| docker-compose | 1.9.0 |
| IP | 111.231.112.132 |

### 证书

如已有证书，可跳过本段

### 1.CA

* 创建并进入证书目录
 ```
 mkdir -p /data/cert
 cd /data/cert
 ```
* 生成CA Key
 ```
 openssl genrsa -out ca.key 3072
 ```
* 生成CA Pem
 ```
 openssl req -new -x509 -days 1095 -key ca.key -out ca.pem
 ```
 根据提示输入相关内容

 ```
Country Name (2 letter code) [XX]:CN
State or Province Name (full name) []:Shanghai
Locality Name (eg, city) [Default City]:Shanghai
Organization Name (eg, company) [Default Company Ltd]:
Organizational Unit Name (eg, section) []:
Common Name (eg, your name or your server's hostname) []:
Email Address []:
 ```
### 2.域名证书

* 生成Key
```
openssl genrsa -out your.harbor.domain.key 3072
```
* 生成证书请求
```
openssl req -new -key your.harbor.domain.key -out your.harbor.domain.csr
```
根据提示输入相关内容
```
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.

Country Name (2 letter code) [XX]:CN
State or Province Name (full name) []:Shanghai
Locality Name (eg, city) [Default City]:Shanghai
Organization Name (eg, company) [Default Company Ltd]:
Organizational Unit Name (eg, section) []:
Common Name (eg, your name or your server's hostname) []:img.linge.io
Email Address []:

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
```
* 签发证书
```
openssl x509 -req -in your.harbor.domain.csr -CA ca.pem -CAkey ca.key -CAcreateserial -out your.harbor.domain.pem -days 1095
```
* 查看证书信息
```
openssl x509 -noout -text -in your.harbor.domain.pem
```
* 信任CA证书
```
cp ca.pem /etc/pki/ca-trust/source/anchors/
update-ca-trust enable
update-ca-trust extract
```
复制ca.pem到/etc/pki/ca-trust/source/anchors/
执行update-ca-trust enable
执行update-ca-trust extract
如Docker已启动，则重启
在其他节点上重复上述命令，以便导入并信任此CA证书

# Docker

## 1.安装Docker&docker-compose

* 使用yum安装
升级
```
yum update
yum install -y docker.io docker-compose
```
需先安装EPEL源
```
rpm -ivh https://mirrors.tuna.tsinghua.edu.cn/centos/7/extras/x86_64/Packages/epel-release-7-9.noarch.rpm
```
## 2.启动Docker

* Start & Enable Docker
```
systemctl start docker
systemctl enable docker
```

# 安装Harbor

## 1.下载离线安装包

* 从官网下载
```
curl -O https://github.com/vmware/harbor/releases/download/v1.2.2/harbor-offline-installer-v1.2.2.tgz
```
* 从镜像站点下载
```
curl -O http://harbor.orientsoft.cn/harbor-1.2.2/harbor-offline-installer-v1.2.2.tgz
```

## 2.安装

* 解压后进入harbor目录
```
tar zxf harbor-offline-installer-v1.2.2.tgz
cd harbor
```
* 修改harbor.cfg文件
```
hostname = img.linge.io
ui_url_protocol = https
db_password = root123
max_job_workers = 3
customize_crt = on
ssl_cert = /data/cert/img.linge.io.pem
ssl_cert_key = /data/cert/img.linge.io.key
secretkey_path = /data
harbor_admin_password = Harbor12345
self_registration = off
token_expiration = 30
project_creation_restriction = everyone
verify_remote_cert = on
```
E-Mail & 认证设置可以后期在Harbor中直接修改

* 安装Harbor
```
./install.sh --with-notary --with-clair
```
命令完成后，相关容器即已启动，可以通过`docker ps -a`查看

* 确认容器运行状态
```
docker ps -a

CONTAINER ID        IMAGE                                     COMMAND                  CREATED             STATUS              PORTS                                                              NAMES
8aeced18c424        vmware/nginx-photon:1.11.13               "nginx -g 'daemon ..."   50 minutes ago      Up 50 minutes       0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp, 0.0.0.0:4443->4443/tcp   nginx
5b26bd765b8d        vmware/harbor-jobservice:v1.2.2           "/harbor/harbor_jo..."   50 minutes ago      Up 50 minutes                                                                          harbor-jobservice
11137c0fdfcf        vmware/notary-photon:server-0.5.0         "/usr/bin/env sh -..."   50 minutes ago      Up 50 minutes                                                                          notary-server
5ffa8915e63b        vmware/harbor-ui:v1.2.2                   "/harbor/harbor_ui"      50 minutes ago      Up 50 minutes                                                                          harbor-ui
8ea3c5eddc92        vmware/clair:v2.0.1-photon                "/clair2.0.1/clair..."   50 minutes ago      Up 49 minutes       6060-6061/tcp                                                      clair
8979001fb52c        vmware/notary-photon:signer-0.5.0         "/usr/bin/env sh -..."   50 minutes ago      Up 50 minutes                                                                          notary-signer
c794d00f02d1        vmware/registry:2.6.2-photon              "/entrypoint.sh se..."   50 minutes ago      Up 50 minutes       5000/tcp                                                           registry
a748399c3ace        vmware/harbor-db:v1.2.2                   "docker-entrypoint..."   50 minutes ago      Up 50 minutes       3306/tcp                                                           harbor-db
931b51bdbbe1        vmware/postgresql:9.6.4-photon            "/entrypoint.sh po..."   50 minutes ago      Up 50 minutes       5432/tcp                                                           clair-db
e89258795849        vmware/harbor-adminserver:v1.2.2          "/harbor/harbor_ad..."   50 minutes ago      Up 50 minutes                                                                          harbor-adminserver
3acfb83cd5ba        vmware/harbor-notary-db:mariadb-10.1.10   "/docker-entrypoin..."   50 minutes ago      Up 50 minutes       3306/tcp                                                           notary-db
db40b9cf7618        vmware/harbor-log:v1.2.2                  "/bin/sh -c 'crond..."   50 minutes ago      Up 50 minutes       127.0.0.1:1514->514/tcp                                            harbor-log
```
如有Exited状态容器，使用`docker start CONTAINER ID`来启动即可

## 3.使用

添加域名到/etc/hosts
* 在命令行使用
docker login
```
docker login -u admin -p Harbor12345 your.harbor.domain
```
* 使用浏览器访问
https://your.harbor.domain
输入用户名／密码即可
