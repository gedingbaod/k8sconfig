## K8S:

## 基础配置环境：8台

```shell
#IP:192.168.2.100（服务器）
dev.laisontech.com:192.168.2.200
master01:192.168.2.201
master02:192.168.2.202
master03:192.168.2.203

#单独harbor（安装机械硬盘）
harbor.laisontech.com:192.168.2.128
#----------------------------------------------------------

#IP:192.168.2.101（服务器）
node01:192.168.2.204
node02:192.168.2.205
node03:192.168.2.206
node04:192.168.2.207

#jenkins
jenkins.laisontech.com:192.168.2.129
#----------------------------------------------------------

#安装时设置IPV4
subnet 192.168.0.0/16
adress 192.168.2.200
gateway 192.168.0.1
nameserver 8.8.8.8

#镜像加速
https://mirrors.aliyun.com/ubuntu/

#更新 失败请查看：ip地址、网关是否配对正确
sudo apt-get update
sudo apt-get upgrade
```

### 备注：内部修改ip地址

``` shell
#设置静态IP地址
sudo vim /etc/netplan/00-installer-config.yaml
#编辑网络配置文件
network:
  ethernets:
    ens33: #配置的网卡名称,使用ifconfig -a查看得到
      dhcp4: no #dhcp4关闭
      addresses: [192.168.2.200/16] #设置本机IP及掩码
      gateway4: 192.168.0.1 #设置网关
      nameservers:
          addresses: [8.8.8.8] #设置DNS
  version: 2
#使用命令，使静态ip生效
sudo netplan apply
```

### 1.1修改主机名

```shell
sudo vim /etc/hostname
master01

sudo hostnamectl set-hostname master01
master01

#配置/etc/hosts
#IP地址 服务器名称（ip200的hostname改为dev.laisontech.com）
#注释掉::1这一行
#注释掉ipv6
sudo vi /etc/hosts
192.168.2.200 dev.laisontech.com
192.168.2.201 master01
192.168.2.202 master02
192.168.2.203 master03
192.168.2.204 node01
192.168.2.205 node02
192.168.2.206 node03
192.168.2.207 node04
```

### 1.2配置时区

```shell
#localtime
sudo mv /etc/localtime /etc/localtime.utc
#类似创建了快捷方式localtime
sudo ln -s /usr/share/zoneinfo/Asia/Shanghai localtime
#生成
#localtime ->/usr/share/zoneinfo/Asia/Shanghai
#localtime.utc ->/usr/share/zoneinfo/Etc/UTC
```

### 1.3配置内存交换

```shell
sudo vim /etc/default/grub
cgroup_enable=memory swapaccount=1
GRUB_CMDLINE_LINUX="cgroup_enable=memory swapaccount=1"
sudo update-grub
```

### 1.4配置免密

```shell
sudo vi /etc/sudoers
lapis  ALL=(ALL) NOPASSWD:ALL
```

### 1.5重启

```shell
sudo reboot
```



## Docker安装(Ubuntu)

### 1.卸载

```shell
sudo apt-get remove docker docker-engine docker.io containerd runc
```

### 2.安装依赖和镜像库

```shell
sudo apt-get update
```

### 3.安装 apt 依赖包

```shell
sudo apt-get install \
   apt-transport-https \
   ca-certificates \
   curl \
   gnupg-agent \
   software-properties-common -y
```

### 4.添加 Docker 的官方 GPG 密钥

```shell
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```

### 5.设置稳定版仓库

```shell
sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) \
  stable"
```

### 6.安装 Docker

```shell
sudo apt-get update
#安装Docker19.03.14版本：K8S最新版本docker20.10.1不支持
sudo apt-get install docker-ce=19.03.14~ce-0~ubuntu
#安装最新版本Docker
sudo apt-get install docker-ce -y
```

### 7.设置开机自启动并启动 Docker

```shell
sudo systemctl enable docker
sudo systemctl start docker
```

### 8.镜像加速

```shell
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://uyah70su.mirror.aliyuncs.com"],
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
```

### 9.配置网络

```shell
sudo vi /lib/systemd/system/docker.service
ExecStartPost=/sbin/iptables -P FORWARD ACCEPT

sudo systemctl daemon-reload
sudo systemctl restart docker
```

### 10.添加当前用户到 docker 用户组

```shell
# 列出自己的用户组，确认自己在不在 docker 组中
groups
# 没有则新增docker组
sudo groupadd docker
# 把当前用户加入到docker组中
sudo gpasswd -a ${USER} docker 
#更新docker用户组
newgrp docker                 
# 重启docker服务
sudo service docker restart

# 备注 把用户从组中移除
# 模板：gpasswd -d userName groupName
gpasswd -d lapis docker
```



## 安装docker-compose

```shell
#运行以下命令以下载Docker Compose的当前稳定版本
sudo curl -L "https://github.com/docker/compose/releases/download/1.27.4/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
#将可执行权限应用于二进制文件
sudo chmod +x /usr/local/bin/docker-compose
#测试安装
docker-compose --version
```



## 安装Harbor

### 1.解压离线安装包

```shell
#本地下载压缩包，导入Linux中解压
tar -zxf harbor-offline-installer-v2.1.2.tgz
```

### 2.生成证书颁发机构证书

```shell
cd /etc/ssl
#生成CA证书私钥
sudo openssl genrsa -out ca.key 4096
#生成CA证书
sudo openssl req -x509 -new -nodes -sha512 -days 3650 \
 -subj "/C=CN/ST=Zhejiang/L=Hangzhou/O=Harbor/OU=Harbor/CN=harbor.laisontech.com" \
 -key ca.key \
 -out ca.crt
```

![image-20201218181448597](C:\Users\Lucian\AppData\Roaming\Typora\typora-user-images\image-20201218181448597.png)

### 3.生成服务器证书

```shell
#生成私钥
sudo openssl genrsa -out harbor.laisontech.com.key 4096
#生成证书签名请求（CSR）
sudo openssl req -sha512 -new \
    -subj "/C=CN/ST=ZheJiang/L=Hangzhou/O=Harbor/OU=Harbor/CN=harbor.laisontech.com" \
    -key harbor.laisontech.com.key \
    -out harbor.laisontech.com.csr
    
#生成一个x509 v3扩展文件
sudo cat > v3.ext <<-EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1=harbor.laisontech.com
DNS.2=192.168.2.128
DNS.3=harbor
EOF
```

![image-20201218181610828](C:\Users\Lucian\AppData\Roaming\Typora\typora-user-images\image-20201218181610828.png)

![image-20201218182250924](C:\Users\Lucian\AppData\Roaming\Typora\typora-user-images\image-20201218182250924.png)

```shell
#使用该v3.ext文件为您的Harbor主机生成证书
sudo openssl x509 -req -sha512 -days 3650 \
    -extfile v3.ext \
    -CA ca.crt -CAkey ca.key -CAcreateserial \
    -in harbor.laisontech.com.csr \
    -out harbor.laisontech.com.crt
```

![image-20201218182349268](C:\Users\Lucian\AppData\Roaming\Typora\typora-user-images\image-20201218182349268.png)

```shell
#-p是递归创建文件
mkdir -p /data/cert
#将服务器证书和密钥复制到Harbor主机上的cert文件夹中
sudo cp harbor.laisontech.com.crt /data/cert/
sudo cp harbor.laisontech.com.key /data/cert/

#转换yourdomain.com.crt为yourdomain.com.cert，供Docker使用
sudo openssl x509 -inform PEM -in harbor.laisontech.com.crt -out harbor.laisontech.com.cert

sudo cp harbor.laisontech.com.cert /data/cert/
```

![image-20201218182850036](C:\Users\Lucian\AppData\Roaming\Typora\typora-user-images\image-20201218182850036.png)

```shell
sudo mkdir -p /etc/docker/certs.d/harbor.laisontech.com/
cd /etc/docker/certs.d/harbor.laisontech.com/
sudo cp /etc/ssl/ca.crt ./
sudo cp /etc/ssl/harbor.laisontech.com.cert ./
sudo cp /etc/ssl/harbor.laisontech.com.key ./

#备注：
#域名和证书名字必须一致
#放证书的目录名，也叫域名
#tree
#.
#└── harbor.laisontech.com
#    ├── ca.crt
#    ├── harbor.laisontech.com.cert
#    └── harbor.laisontech.com.key
```

![image-20201218183055406](C:\Users\Lucian\AppData\Roaming\Typora\typora-user-images\image-20201218183055406.png)

```shell
#重启Docker
systemctl daemon-reload
systemctl restart docker
```

### 4.重要：添加域名进去

```shell
#需要添加"insecure-registries":["192.168.0.2:8888"]参数到docker的/etc/docker/daemon.json配置文件中，
cd /etc/docker/daemon.json
sudo vi daemon.json
"insecure-registries" : [ "harbor.laisontech.com","192.168.2.128" ],
systemctl daemon-reload
systemctl restart docker
```

### 5.配置harbor.yml

```shell
#修改hostname => harbor.laisontech.com

#修改
#certificate: /etc/ssl/harbor.laisontech.com.cert
#private_key: /etc/ssl/harbor.laisontech.com.key

#harbor_admin_password: Harbor12345
```

![image-20201218183631988](C:\Users\Lucian\AppData\Roaming\Typora\typora-user-images\image-20201218183631988.png)

### 6.安装Harbor

```shell
#安装harbor
sudo ./install.sh
```

![image-20201218183959722](C:\Users\Lucian\AppData\Roaming\Typora\typora-user-images\image-20201218183959722.png)

![image-20201218184013654](C:\Users\Lucian\AppData\Roaming\Typora\typora-user-images\image-20201218184013654.png)

### Harbor起停

```shell
#harbor起停
sudo docker-compose down -v
sudo docker-compose up -d
```

### 进程查看

```shell
#进程查看harbor
ps -ef | grep harbor
#杀死进程
kill -9 [信号数字]
```

### Docker上传镜像到Harbor

```shell
#在项目中标记镜像TAG
docker tag SOURCE_IMAGE[:TAG] harbor.laisontech.com/harbor/REPOSITORY[:TAG]
#推送镜像到当前项目 => harbor
docker push harbor.laisontech.com/harbor/REPOSITORY[:TAG]
```



## Maven私服=>Docker

```shell
#192.168.2.128 maven私服和harbor装在一起
docker run -d -p 8081:8081 --name nexus -v /opt/nexus-data:/nexus-data --restart=always sonatype/nexus3

sudo chmod 777 nexus-data/
```



## Idea => Maven提交到私服

```shell
#1.idea中的pom文件添加信息
    <distributionManagement>
        <repository>
            <id>releases</id>
            <name>releases</name>
            <url>http://192.168.2.128:8081/repository/maven-releases/</url>
        </repository>
        <snapshotRepository>
            <id>snapshots</id>
            <name>snapshots</name>
            <url>http://192.168.2.128:8081/repository/maven-snapshots/</url>
        </snapshotRepository>
    </distributionManagement>
            
#2.本地maven中settings添加路径
   <server>
      <id>releases</id>
      <username>admin</username>
      <password>LaisonTech4u</password>
    </server>
    <server>
      <id>snapshots</id>
      <username>admin</username>
      <password>LaisonTech4u</password>
    </server>
```



## Jenkins部署

```shell
docker run --name blueocean --restart always --user root --privileged=true --dns 8.8.8.8 -d -p 8080:8080 -p 50000:50000 -v /opt/blueocean/home:/var/jenkins_home -v /var/run/docker.sock:/var/run/docker.sock jenkinsci/blueocean

#进入容器查看密码
docker exec -u 0 -it 容器号  /bin/bash
cat /var/jenkins_home/secrets/initialAdminPassword
exit

#官网的
docker run \
  -u root \
  --rm \  
  -d \ 
  -p 8080:8080 \ 
  -p 50000:50000 \ 
  -v opt/blueocean/home:/var/jenkins_home \ 
  -v /var/run/docker.sock:/var/run/docker.sock \ 
  jenkinsci/blueocean 
  
  
docker exec -u 0 -it 容器号  /bin/bash
echo $JAVA_HOME
#/opt/java/openjdk => Jenkins填写JDK路径
#git:/usr/bin/git => Jenkins填写Git路径
#maven直接在jenkins中的全局设置中直接安装
```



## K8S搭建互联

```shell
#master01 master02 master03
#安装kubectl
chmod +x linux-amd64-v1.18.12-kubectl
sudo mv ./linux-amd64-v1.18.12-kubectl /usr/local/bin/kubectl
kubectl version
#安装RKE
chmod +x rke_linux-amd64
sudo mv rke_linux-amd64 /usr/local/bin/rke
rke --version
#安装Helm
tar -zxvf helm-v3.4.2-linux-amd64.tar.gz
sudo mv linux-amd64/helm /usr/local/bin/helm
helm version

#主：master02 实现互联master01和master03
#安装主机配置ssh免密连接
ssh-keygen -t rsa，然后连续回车
ssh-copy-id master01
ssh-copy-id master02
ssh-copy-id master03

#修改配置文件cluster.yml
#增加服务器
#启动rke
rke -d up

#查看是否互联
kubectl get nodes
kubectl get nodes -o wide
--------------------------------------------------------------------

#node之间互联   首先：node本机之间ssh-keygen -t rsa生成公钥密钥id_rsa  id_rsa.pub，然后node01进行ssh-copy-id node02（进行免密连接），node02中的authorized_keys里添加一句。ssh node02测试与node02的连通性


#node之间
#安装时间同步
sudo apt install ntpdate 
sudo ntpdate time1.aliyun.com

#更新ssh配置
sudo vi /etc/ssh/sshd_config
AllowTcpForwarding yes
sudo systemctl restart sshd

#配置docker服务
sudo vi /lib/systemd/system/docker.service
增加ExecStartPost=/sbin/iptables -P FORWARD ACCEPT

#安装配置ssh（node之间可不必互联）
ssh-keygen -t rsa
ssh-copy-id node01
ssh-copy-id node02
ssh-copy-id node03
ssh-copy-id node04

#测试连通性
ssh node01

--------------------------------------------------------------------------
#把node添加到maser02中

ssh-copy-id node01
#修改配置文件cluster.yml
#增加node01
#启动rke
rke -d up
#查看是否互联
kubectl get nodes
kubectl get nodes -o wide

#以此类推node02，node03，node04
```



## Idea => Gitlab

```shell
1.gitlab新建一个项目工程
2.在idea上新建一个项目，完成之后，需要创建一个git仓库； create git repositroy
3.右键项目：Git =》 add commint push
4.push：项目名字+远程仓库url：gitlab项目

详情查看：https://www.cnblogs.com/shenwen/p/9149478.html
```





## Podman安装

```shell
# Ubuntu 20.10 and newer
sudo apt-get -y update
sudo apt-get -y install podman

#重要安装
sudo apt-get update
sudo apt-get install software-properties-common uidmap
sudo add-apt-repository ppa:projectatomic/ppa
sudo apt-get update
sudo apt-get install podman

#静默安装
sudo apt-get update -qq
sudo apt-get install -qq -y software-properties-common uidmap
sudo add-apt-repository -y ppa:projectatomic/ppa
sudo apt-get update -qq
sudo apt-get -qq -y install podman

#迫不得己安装
. /etc/os-release
echo "deb https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/xUbuntu_${VERSION_ID}/ /" | sudo tee /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list
curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/xUbuntu_${VERSION_ID}/Release.key | sudo apt-key add -
sudo apt-get update
sudo apt-get -y upgrade
sudo apt-get -y install podman
```

