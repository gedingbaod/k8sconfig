
#卸载原有docker
sudo apt-get remove docker docker-engine docker-ce docker.io
dpkg -l | grep docker
dpkg -l |grep ^rc|awk '{print $2}' |sudo xargs dpkg -P
sudo apt-get autoremove docker-ce-*
sudo rm -rf /etc/systemd/system/docker.service.d
docker --version



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


#安装主机配置ssh免密连接
ssh-keygen -t rsa，然后连续回车
ssh-copy-id master01
ssh-copy-id smaster02
ssh-copy-id node01

#安装时间同步
sudo apt install ntpdate 
sudo ntpdate time1.aliyun.com

#更新ssh配置
sudo vi /etc/ssh/sshd_config
AllowTcpForwarding yes
sudo systemctl restart sshd




#配置docker服务
sudo vi b/systemd/system/docker.service
增加ExecStartPost=/sbin/iptables -P FORWARD ACCEPT


#修改配置文件cluster.yml
增加服务器

#启动rke
rke -d up --update-only



