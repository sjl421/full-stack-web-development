# 常见云服务使用问题汇总

## 如何使用密钥连接主机？

阿里和腾讯差不多，都是在控制台建立密钥对，注意一般服务器不会为你保留私钥，请立即下载并保存好私钥。然后在客户端使用如下命令访问，以 ubuntu 镜像为例，下面这样就可以连接了：

```bash
ssh -i </path/to/private_key> ubuntu@xx.xxx.xx.xxx
```

当然第一次连接会询问是否需要保存并信任远程主机的 footprint，输入 `yes` 即可。

```bash
The authenticity of host 'xxx.xx.xxx.xx (xxx.xx.xxx.xx)' can't be established.
ECDSA key fingerprint is SHA256:xxxxxxxx
Are you sure you want to continue connecting (yes/no)? yes
```

## 重装系统后连接失败，提示 `REMOTE HOST IDENTIFICATION HAS CHANGED! ` ，如何处理？

重装系统后，远程主机 footprint 已经改变，此时应该从客户端删除原先保存的记录

```bash
ssh-keygen -R xxx.xx.xxx.xx
```

## 为何有时候会出现提示密钥权限不对的错误？

密钥的权限太宽泛会导致系统中可以访问密钥的人过多，所以需要设置

```bash
chmod 400 </path/to/private_key>
```

## 如何安装 Docker ？

官方一键安装脚本，速度慢，但是比较靠谱

```bash
curl -sSL https://get.docker.com/ | sh
```

## 如何不用 sudo 使用 docker ？

```bash
sudo usermod -aG docker ubuntu
```

## 设置镜像加速

腾讯云

```bash
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://mirror.ccs.tencentyun.com"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```

## 修改 Ubuntu 软件仓库为国内镜像

```bash
# 备份原文件
mv /etc/apt/sources.list /etc/apt/sources.list.bak

# 修改为阿里云的镜像源
cat > /etc/apt/sources.list << END
deb http://mirrors.aliyun.com/ubuntu/ trusty main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ trusty-security main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ trusty-updates main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ trusty-proposed main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ trusty-backports main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ trusty main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ trusty-security main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ trusty-updates main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ trusty-proposed main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ trusty-backports main restricted universe multiverse
END

# 更新源列表信息
apt-get update
```

## 修改 CentOS 软件仓库为国内镜像

```bash
# 备份原文件
mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.bak

# 下载新的CentOS-Base.repo 到/etc/yum.repos.d/

#CentOS 5
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-5.repo

#CentOS 6
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-6.repo

#CentOS 7
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo

# 生成缓存
yum makecache
```

## 高速安装 docker compose

```bash
curl -L https://get.daocloud.io/docker/compose/releases/download/1.18.0/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
```

## 安装 docker 时遇到 `E: Unable to correct problems, you have held broken packages.`

安装 `libltdl7_2.4.6`

```bash
wget http://launchpadlibrarian.net/236916213/libltdl7_2.4.6-0.1_amd64.deb
```

## 如何在远程终端连接到服务器后，可以在不保持会话的条件下执行命令？

在远程主机上执行

```bash
screen
```

然后即可执行想要执行的命令，退出这个 screen 会话，可以按 `CTRL + A` 然后按 `d` 。
如果恢复会话，可以使用下面的命令进行会话的恢复。

```bash
screen -r
```

## 在云环境部署 Elasticsearch 的容器时发生 `max virtual memory areas vm.max_map_count [65530] likely too low, increase to at least [262144]` 错误

```bash
sudo sysctl -w vm.max_map_count=262144
```
