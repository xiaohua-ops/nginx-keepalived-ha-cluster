## 一、环境准备

### 1.1 硬件要求

配置项最低要求推荐配置CPU2 核4 核及以上内存2G4G 及以上硬盘20G40G 及以上网络公网 / 内网可通固定 IP 地址

### 1.2 软件版本约束

软件版本要求说明CentOS Stream9非 Stream 版 CentOS 已停止官方维护Nginx1.24.0 稳定版编译安装，支持 HTTPS、状态监控模块MySQL8.0.42二进制包安装，兼容商城业务PHP7.4.33CRMEB 商城强制要求 7.1~7.4 版本，高版本不兼容CRMEBv5.4+开源版，Apache-2.0 协议可免费商用

## 二、系统初始化配置

**所有操作均使用 root 账号执行**

### 2.1 配置固定 IP 地址

编辑网卡配置（以 `ens160.nmconnection` 为例，替换为你的实际网卡名）

```Plain
vim /etc/NetworkManager/system-connections/ens160.nmconnection
```

1.写入以下 IPv4 配置，根据实际网络环境修改地址、网关、DNS

```Shell
[ipv4]
method=manual
addresses=192.168.88.101/24
gateway=192.168.88.2
dns=8.8.8.8;114.114.114.114;

[ipv6]
method=ignore
```

2.重载网卡

```Plain
nmcli c reload
nmcli c up ens160
```

### 2.2 配置主机名与 hosts 解析

```Plain
# 设置主机名
hostnamectl set-hostname web01.itcast.cn && bash

# 配置本地hosts解析 cat >> /etc/hosts << EOF
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.88.101   web01 web01.itcast.cn
EOF
```

### 2.3 关闭防火墙与 SELinux

```Plain
# 停止并禁用防火墙
systemctl stop firewalld
systemctl disable firewalld

# 临时关闭SELinux 
setenforce 0
# 永久关闭SELinux
sed -i '/SELINUX=enforcing/cSELINUX=disabled' /etc/selinux/config

# 验证SELinux状态
getenforce
```

### 2.4 配置清华大学 yum 镜像源（官方文档）

CentOS Stream 9 官方源访问速度慢，替换为国内清华镜像源

#### 一键脚本替换

1.安装 Perl 依赖

```Plain
dnf install perl perl-autodie -y
```

2.创建镜像替换脚本

```Plain
vim update_mirror.pl
```

3.以下脚本内容

```Bash
#!/usr/bin/perl
use strict;
use warnings;
use autodie;

my $mirrors = 'https://mirrors.tuna.tsinghua.edu.cn/centos-stream';

if (@ARGV < 1) {
    die "Usage: $0 <filename1> <filename2> ...\n";
}

while (my $filename = shift @ARGV) {
    my $backup_filename = $filename . '.bak';
    rename $filename, $backup_filename;

    open my $input, "<", $backup_filename;
    open my $output, ">", $filename;

    while (<$input>) {
        s/^metalink/# metalink/;

        if (m/^name/) {
            my (undef, $repo, $arch) = split /-/;
            $repo =~ s/^\s+|\s+$//g;
            ($arch = defined $arch ? lc($arch) : '') =~ s/^\s+|\s+$//g;

            if ($repo =~ /^Extras/) {
                $_ .= "baseurl=${mirrors}/SIGs/\$releasever-stream/extras" . ($arch eq 'source' ? "/${arch}/" : "/\$basearch/") . "extras-common\n";
            } else {
                $_ .= "baseurl=${mirrors}/\$releasever-stream/$repo" . ($arch eq 'source' ? "/" : "/\$basearch/") . ($arch ne '' ? "${arch}/tree/" : "os") . "\n";
            }
        }

        print $output $_;
    }
}
```

4.执行脚本替换源并更新缓存

```Plain
perl ./update_mirror.pl /etc/yum.repos.d/centos*.repo
dnf clean all && dnf makecache
```

### 2.5 系统时间同步

```Plain
# 安装chrony时间同步服务
dnf install chrony -y
# 启动并设置开机自启
systemctl enable chronyd --now
# 验证同步状态
chronyc tracking
```

## 三、LNMP 环境编译安装

### 3.1 Nginx 编译安装

1.安装编译依赖库

```Plain
dnf -y install pcre-devel zlib-devel openssl-devel gcc gcc-c++ make
```

2.创建运行用户（禁止登录）

```Plain
useradd -r -s /sbin/nologin www
```

3.下载并解压 Nginx 源码包

```Plain
cd /usr/local/src
wget http://nginx.org/download/nginx-1.24.0.tar.gz
tar xvf nginx-1.24.0.tar.gz
cd nginx-1.24.0
```

4.编译配置

```Plain
./configure --prefix=/usr/local/nginx --user=www --group=www
```

5.编译并安装（`-j$(nproc)` 自动匹配 CPU 核心数加速编译）

```Plain
make -j$(nproc) && make install
```

6.配置环境变量

```Plain
echo 'export PATH=$PATH:/usr/local/nginx/sbin' >> /etc/profile
source /etc/profile

# 验证安装
nginx -v
```

7.配置 systemd 系统服务

```Plain
vim /usr/lib/systemd/system/nginx.service
```

写入以下内容：

```JavaScript
[Unit]
Description=Nginx Web Server
After=network.target

[Service]
Type=forking
PIDFile=/usr/local/nginx/logs/nginx.pid
ExecStart=/usr/local/nginx/sbin/nginx
ExecReload=/usr/local/nginx/sbin/nginx -s reload
ExecStop=/usr/local/nginx/sbin/nginx -s quit
PrivateTmp=true
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
```

8.启动 Nginx 并设置开机自启

```Plain
systemctl daemon-reload
systemctl enable nginx --now
# 验证服务状态
systemctl status nginx
# 验证端口监听
ss -naltp | grep nginx
```

浏览器访问服务器 IP，出现 `Welcome to nginx!` 即为安装成功。

### 3.2 MySQL 8.0 二进制安装

1.依赖安装

```Shell
# 安装依赖
dnf install libaio-devel numactl-devel -y
```

2.创建运行用户

```Plain
useradd -r -s /sbin/nologin mysql
```

3.下载并解压 MySQL 二进制包

```Plain
cd /usr/local/src
wget https://cdn.mysql.com/Downloads/MySQL-8.0/mysql-8.0.42-linux-glibc2.28-x86_64.tar.xz
tar -xf mysql-8.0.42-linux-glibc2.28-x86_64.tar.xz

# 移动到安装目录
mv mysql-8.0.42-linux-glibc2.28-x86_64 /usr/local/mysql
```

4.创建数据目录与日志目录

```Plain
mkdir -p /usr/local/mysql/data /usr/local/mysql/logs
chown -R mysql:mysql /usr/local/mysql
chmod 750 /usr/local/mysql/data
```

5.配置 my.cnf 主配置文件

```Plain
vim /etc/my.cnf
```

写入以下内容：

```JavaScript
[mysqld]
basedir=/usr/local/mysql
datadir=/usr/local/mysql/data
socket=/tmp/mysql.sock
port=3306

log-error=/usr/local/mysql/logs/error.log
log-bin=/usr/local/mysql/data/binlog
server-id=10
character_set_server=utf8mb4

gtid-mode=on
log-slave-updates=1
enforce-gtid-consistency=1
sql_mode=NO_ENGINE_SUBSTITUTION

[mysql]
socket=/tmp/mysql.sock
```

6.数据库初始化

```Bash
cd /usr/local/mysql
./bin/mysqld --initialize --user=mysql --basedir=/usr/local/mysql

# 查看初始化生成的临时root密码（保存好，后续登录需要）
grep password /usr/local/mysql/logs/error.log | awk '{print $NF}'
```

7.配置 systemd 系统服务

```Plain
vim /usr/lib/systemd/system/mysqld.service
```

写入以下内容：

```JavaScript
[Unit]
Description=MySQL Server
After=network.target

[Service]
User=mysql
Group=mysql
Type=forking
ExecStart=/usr/local/mysql/bin/mysqld --daemonize --pid-file=/usr/local/mysql/data/mysqld.pid
ExecStop=/usr/local/mysql/bin/mysqladmin --defaults-file=/etc/my.cnf shutdown
TimeoutSec=600
PIDFile=/usr/local/mysql/data/mysqld.pid
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

8.启动 MySQL 并设置开机自启

```Plain
systemctl daemon-reload
systemctl enable mysqld --now
# 验证服务状态
systemctl status mysqld
# 验证端口监听
ss -naltp | grep mysqld
```

9.配置环境变量与修改 root 密码

```Plain
# 添加环境变量
echo 'export PATH=$PATH:/usr/local/mysql/bin' >> /etc/profile
source /etc/profile

# 登录MySQL（输入上面保存的临时密码）
mysql -uroot -p
# 登录后修改root密码（替换为你的自定义密码）
mysql> set password ='123456';
mysql> flush privileges;
mysql> exit;
```

### 3.3 PHP 7.4 安装（适配 CRMEB 商城）

CRMEB 商城仅支持 PHP 7.1~7.4，使用 Remi 源安装，简化依赖管理。

1.安装 EPEL 与 Remi 仓库

```Plain
# 安装EPEL仓库
dnf install epel-release -y
# 安装Remi仓库
dnf install -y https://rpms.remirepo.net/enterprise/remi-release-9.rpm

# 启用PHP 7.4模块
dnf module enable php:remi-7.4 -y
```

2.安装 PHP 7.4 及商城所需扩展

```Plain
dnf install -y php php-cli php-fpm php-mysqlnd php-xml php-mbstring php-json php-common php-opcache php-bcmath php-gd php-curl php-fileinfo php-redis
```

3.验证安装

```Plain
php -v
```

4.配置 PHP-FPM

```Plain
vim /etc/php-fpm.d/www.conf
```

修改以下核心配置（适配 Nginx 运行用户）：

```Shell
# 第23-26行，修改运行用户和组
user = www
group = www

# 第38行，修改监听地址和端口
listen = 127.0.0.1:9000

# 进程管理模式，生产环境推荐static
pm = static
pm.max_children = 50
```

5.配置 systemd 服务与启动

```Plain
# 授权目录权限
chown -R www:www /var/lib/php
chown -R www:www /var/log/php-fpm

# 重载系统服务
systemctl daemon-reload

# 启动php-fpm并设置开机自启
systemctl enable php-fpm --now
# 验证服务状态
systemctl status php-fpm
# 验证端口监听
ss -naltp | grep 9000
```

## 四、Nginx 与 PHP 联动配置

### 4.1 核心配置修改

1.备份原始 Nginx 配置

```Plain
cp /usr/local/nginx/conf/nginx.conf /usr/local/nginx/conf/nginx.conf.bak
```

2.编辑 Nginx 主配置文件

```Plain
vim /usr/local/nginx/conf/nginx.conf
```

替换为以下精简优化配置：

```Markdown
worker_processes  auto;
worker_rlimit_nofile 65535;

events {
    worker_connections  65536;
    use epoll;
    multi_accept on;
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  logs/access.log  main;
    error_log  logs/error.log  warn;

    sendfile        on;
    tcp_nopush      on;
    tcp_nodelay     on;

    keepalive_timeout  65;
    client_max_body_size 50M;

    server {
        listen       80;
        server_name  localhost;
        root   html;

        location / {
            index  index.html index.htm index.php;
        }

        # PHP解析核心配置
        location ~ \.php$ {
            fastcgi_pass   127.0.0.1:9000;
            fastcgi_index  index.php;
            fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
            include        fastcgi_params;
        }

        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }
}
```

3.验证配置并重载 Nginx

```Plain
nginx -t
nginx -s reload
```

### 4.2 环境验证

1.创建 PHP 测试文件

```Plain
vim /usr/local/nginx/html/demo.php
```

写入以下内容：

```Plain
<?phpphpinfo();?>
```

浏览器访问 `http://服务器IP/demo.php`，出现 PHP 信息页面即为 LNMP 环境搭建成功。

## 五、CRMEB 开源商城部署

### 5.1 源码下载与目录配置

1.安装 Git 工具

```Plain
dnf install git -y
```

2.创建项目目录并克隆源码

```Plain
mkdir -p /data
cd /data
git clone https://gitee.com/ZhongBangKeJi/CRMEB.git
```

3.配置目录权限

```Plain
chown -R www:www /data/CRMEB
chmod -R 755 /data/CRMEB
```

### 5.2 Nginx 商城站点配置

1.修改 Nginx 配置文件

```Plain
vim /usr/local/nginx/conf/nginx.conf
```

替换 server 模块为以下商城专属配置：

```Bash
server {
    listen       80;
    server_name  localhost;
    # 商城项目根目录
    root /data/CRMEB/crmeb/public;

    location / {
        index  index.html index.php;
        # 商城伪静态规则
        if (!-e $request_filename) {
            rewrite  ^(.*)$  /index.php?s=$1  last;
            break;
        }
    }

    # PHP解析配置
    location ~ \.php$ {
        fastcgi_pass   127.0.0.1:9000;
        fastcgi_index  index.php;
        fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
        include        fastcgi_params;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   html;
    }
}
```

2.验证配置并重载 Nginx

```Plain
nginx -t
nginx -s reload
```

### 5.3 数据库创建

```Plain
# 登录MySQL
mysql -uroot -p
# 创建商城数据库
mysql> create database crmeb default character set utf8mb4 collate utf8mb4_unicode_ci;
# 创建专用数据库用户（生产环境推荐，替换为自定义密码）
mysql> create user 'crmeb'@'localhost' identified by 'Crmeb@123456';
mysql> grant all privileges on crmeb.* to 'crmeb'@'localhost';
mysql> flush privileges;
mysql> exit;
```

# 集群整体架构部署

# 一、102、103 节点环境初始化（集群的基础，必须先做）

所有节点必须保持环境一致，否则会出现各种奇怪的兼容问题，以下命令 102、103 节点全部执行一遍，101 节点之前已经做过，不用重复执行。

## 基础环境配置（3 节点必须完全一致）

### 配置清华大学 yum 镜像源（官方文档）

CentOS Stream 9 官方源访问速度慢，替换为国内清华镜像源

#### 一键脚本替换

1.安装 Perl 依赖

```Plain
dnf install perl perl-autodie -y
```

2.创建镜像替换脚本

```Plain
vim update_mirror.pl
```

3.写入以下脚本内容

```Bash
#!/usr/bin/perl
use strict;
use warnings;
use autodie;

my $mirrors = 'https://mirrors.tuna.tsinghua.edu.cn/centos-stream';

if (@ARGV < 1) {
    die "Usage: $0 <filename1> <filename2> ...\n";
}

while (my $filename = shift @ARGV) {
    my $backup_filename = $filename . '.bak';
    rename $filename, $backup_filename;

    open my $input, "<", $backup_filename;
    open my $output, ">", $filename;

    while (<$input>) {
        s/^metalink/# metalink/;

        if (m/^name/) {
            my (undef, $repo, $arch) = split /-/;
            $repo =~ s/^\s+|\s+$//g;
            ($arch = defined $arch ? lc($arch) : '') =~ s/^\s+|\s+$//g;

            if ($repo =~ /^Extras/) {
                $_ .= "baseurl=${mirrors}/SIGs/\$releasever-stream/extras" . ($arch eq 'source' ? "/${arch}/" : "/\$basearch/") . "extras-common\n";
            } else {
                $_ .= "baseurl=${mirrors}/\$releasever-stream/$repo" . ($arch eq 'source' ? "/" : "/\$basearch/") . ($arch ne '' ? "${arch}/tree/" : "os") . "\n";
            }
        }

        print $output $_;
    }
}
```

4.执行脚本替换源并更新缓存

```Plain
perl ./update_mirror.pl /etc/yum.repos.d/centos*.repo
dnf clean all && dnf makecache
```

系统初始化设置

```Plain
# 永久关闭SELinux，避免权限拦截
setenforce 0
sed -i 's/^SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
getenforce  
# 验证：输出Disabled即为成功
# 防火墙配置：放通集群所需端口，先关闭后续可精细化配置
systemctl stop firewalld
systemctl disable firewalld

# 配置时间同步（时间不同步会导致主从同步失败、集群脑裂）
dnf install -y chrony
systemctl enable --now chronyd
chronyc tracking  
# 验证：输出同步状态即为成功
# 安装基础依赖包
dnf install -y wget curl vim net-tools gcc make cmake lsof unzip zip bzip2 tar git tree

# 设置主机名，配置hosts（3节点分别执行，对应自己的IP）
# 101节点执行：
hostnamectl set-hostname node1
# 102节点执行：
hostnamectl set-hostname node2
# 103节点执行：
hostnamectl set-hostname node3

# 【所有节点都执行】配置hosts，添加3台节点的解析
cat >> /etc/hosts << EOF
192.168.88.101 node1
192.168.88.102 node2
192.168.88.103 node3
EOF
```

## 配置 3 节点 SSH 免密登录

```Plain
# 主节点节点生成SSH密钥，一路回车即可
ssh-keygen -t rsa

# 主节点101，把公钥分发到102、103和自己
ssh-copy-id root@node1
ssh-copy-id root@node2
ssh-copy-id root@node3

# 验证：101节点执行以下命令，不用输密码就能登录即为成功
ssh node2 hostname
ssh node3 hostname
```

# 二、分层集群部署步骤（从纯 LNMP 开始，从零搭建）

## 第一层：接入层 Keepalived+Nginx 一主二备集群

### 101 节点（已有 Nginx）安装 Keepalived，配置主节点

```Plain
# 101节点执行：安装Keepalived
dnf install -y keepalived
cp /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf.bak
# 清空Keepalived配置文件
> /etc/keepalived/keepalived.conf

# 写入主节点配置
cat >> /etc/keepalived/keepalived.conf << EOF
! 全局配置
global_defs {
    router_id LVS_MASTER
    script_user root
    enable_script_security
}

! Nginx健康检查脚本
vrrp_script chk_nginx {
    script "/etc/keepalived/chk_nginx.sh"
    interval 2
    weight -20
    fall 3
    rise 2
}

! VRRP实例配置
vrrp_instance VI_1 {
    state MASTER
    interface ens33  # 用ip addr查看你的网卡名，一般是ens33/eth0
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.88.200/24  # VIP，后续域名解析到这个IP
    }
    track_script {
        chk_nginx
    }
}
EOF

# 创建健康检查脚本
cat >> /etc/keepalived/chk_nginx.sh << EOF
#!/bin/bash
if ! ps -ef | grep nginx | grep -v grep > /dev/null
then
    systemctl restart nginx
    sleep 2
    if ! ps -ef | grep nginx | grep -v grep > /dev/null
    then
        systemctl stop keepalived
    fi
fi
EOF
# 给脚本加执行权限
chmod +x /etc/keepalived/chk_nginx.sh

# 启动Keepalived
systemctl enable --now keepalived

# 验证：查看VIP绑定成功
ip addr | grep 192.168.88.200
```

### 102、103 节点安装 Nginx+Keepalived，配置备节点

```Plain
# 102、103节点执行：安装Nginx
dnf install -y nginx
systemctl enable --now nginx

# 【关键】从101同步Nginx配置、商城代码到102、103
# 101节点执行：
rsync -avz /etc/nginx/ root@node2:/etc/nginx/
rsync -avz /etc/nginx/ root@node3:/etc/nginx/
rsync -avz /usr/share/nginx/html/ root@node2:/usr/share/nginx/html/
rsync -avz /usr/share/nginx/html/ root@node3:/usr/share/nginx/html/

# 102、103节点重启Nginx
systemctl restart nginx

# 102、103节点安装Keepalived
dnf install -y keepalived
cp /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf.bak
# 清空Keepalived配置文件
> /etc/keepalived/keepalived.conf
```

#### 102 节点（BACKUP1）配置

```Plain
# 102节点写入配置
cat >> /etc/keepalived/keepalived.conf << EOF
! 全局配置
global_defs {
    router_id LVS_BACKUP1
    script_user root
    enable_script_security
}

! Nginx健康检查脚本
vrrp_script chk_nginx {
    script "/etc/keepalived/chk_nginx.sh"
    interval 2
    weight -20
    fall 3
    rise 2
}

! VRRP实例配置
vrrp_instance VI_1 {
    state BACKUP
    interface ens33
    virtual_router_id 51
    priority 90
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.88.200/24
    }
    track_script {
        chk_nginx
    }
}
EOF
```

#### 103 节点（BACKUP2）配置

```Plain
# 103节点写入配置
cat >> /etc/keepalived/keepalived.conf << EOF
! 全局配置
global_defs {
    router_id LVS_BACKUP2
    script_user root
    enable_script_security
}

! Nginx健康检查脚本
vrrp_script chk_nginx {
    script "/etc/keepalived/chk_nginx.sh"
    interval 2
    weight -20
    fall 3
    rise 2
}

! VRRP实例配置
vrrp_instance VI_1 {
    state BACKUP
    interface ens33
    virtual_router_id 51
    priority 80
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.88.200/24
    }
    track_script {
        chk_nginx
    }
}
EOF
```

#### 同步脚本，启动服务

```Plain
# 101节点同步健康检查脚本到102、103
rsync -avz /etc/keepalived/chk_nginx.sh root@node2:/etc/keepalived/
rsync -avz /etc/keepalived/chk_nginx.sh root@node3:/etc/keepalived/

# 102、103节点给脚本加执行权限
chmod +x /etc/keepalived/chk_nginx.sh

# 102、103节点启动Keepalived
systemctl enable --now keepalived
```

### 接入层验证

```Plain
# 1. 访问VIP，能正常打开商城即为成功
curl 192.168.88.200

# 2. 故障模拟：停止101的Keepalived+Nginx
systemctl stop keepalived nginx

# 3. 验证：VIP自动漂移到102，访问VIP依然正常# 102节点执行：
ip addr | grep 192.168.88.200

# 4. 恢复101节点
systemctl start keepalived nginx
```

## 第二层：业务层 PHP-FPM 负载均衡集群

### 102、103 节点安装 PHP-FPM，和 101 版本一致

```Plain
# 102、103节点执行：先查看101的PHP版本，对应安装
# 101节点执行：php -v，比如php8.1
dnf install -y php php-fpm php-mysqlnd php-redis php-gd php-curl php-mbstring php-xml php-bcmath

# 101节点同步PHP配置到102、103
rsync -avz /etc/php.ini root@node2:/etc/
rsync -avz /etc/php.ini root@node3:/etc/
rsync -avz /etc/php-fpm.d/ root@node2:/etc/php-fpm.d/
rsync -avz /etc/php-fpm.d/ root@node3:/etc/php-fpm.d/

# 102、103节点启动PHP-FPM
systemctl enable --now php-fpm
```

### 修改 Nginx 配置，配置 PHP 负载均衡

```Plain
# 所有节点（101、102、103）都修改Nginx配置
vim /etc/nginx/conf.d/shop.conf  # 你的商城配置文件

# 在server块上方添加PHP上游集群
upstream php_cluster {
    server 192.168.88.101:9000;
    server 192.168.88.102:9000;
    server 192.168.88.103:9000;
}

# 把fastcgi_pass改成php_cluster
location ~ \.php$ {
    fastcgi_pass   php_cluster;
    fastcgi_index  index.php;
    fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
    include        fastcgi_params;
}

# 所有节点重启Nginx
systemctl restart nginx
```

## 第三层：缓存层 Redis 一主二从 + 3 节点哨兵集群

### 所有节点安装 Redis

```Plain
# 101、102、103节点都执行：安装Redis
dnf install -y redis
cp /etc/redis/redis.conf /etc/redis/redis.conf.bak

# 修改基础配置，所有节点一致
sed -i 's/^bind 127.0.0.1/bind 0.0.0.0/g' /etc/redis/redis.conf
sed -i 's/^protected-mode yes/protected-mode no/g' /etc/redis/redis.conf
sed -i 's/^daemonize no/daemonize yes/g' /etc/redis/redis.conf
sed -i 's/^# requirepass foobared/requirepass Redis@6666/g' /etc/redis/redis.conf
sed -i 's/^appendonly no/appendonly yes/g' /etc/redis/redis.conf
```

### 101 节点配置主节点，102、103 配置从节点

```Plain
# 102、103节点执行：添加从节点同步配置
echo "replicaof 192.168.88.101 6379" >> /etc/redis/redis.conf
echo "masterauth Redis@6666" >> /etc/redis/redis.conf

# 所有节点启动Redis
systemctl enable --now redis

# 101节点验证主从同步成功
redis-cli -a Redis@6666 info replication
# 输出 connected_slaves:2 即为成功
```

### 3 节点哨兵集群部署

```Plain
# 所有节点配置哨兵，配置文件完全一致cat >> /etc/redis/sentinel.conf << EOF
port 26379
daemonize yes
logfile "/var/log/redis/sentinel.log"
dir "/var/lib/redis"
sentinel monitor mymaster 192.168.88.101 6379 2
sentinel auth-pass mymaster Redis@6666
sentinel down-after-milliseconds mymaster 3000
sentinel failover-timeout mymaster 10000
sentinel parallel-syncs mymaster 1
EOF

# 所有节点给配置文件加权限
chown redis:redis /etc/redis/sentinel.conf
chmod 600 /etc/redis/sentinel.conf

# 所有节点启动哨兵
redis-sentinel /etc/redis/sentinel.conf

# 验证哨兵集群
redis-cli -p 26379 info sentinel
# 输出 num_other_sentinels=2 即为成功
```

### 配置 PHP Session 存储到 Redis

```Plain
# 所有节点修改PHP配置
vim /etc/php.ini
# 修改以下2行
session.save_handler = redis
session.save_path = "tcp://192.168.88.101:6379?auth=Redis@6666"

# 所有节点重启PHP-FPM
systemctl restart php-fpm
```

## 第四层：数据库层 MySQL 一主二从集群

### 101 节点（已有 MySQL）配置主库

```Plain
# 101节点修改my.cnf，开启binlog
cp /etc/my.cnf /etc/my.cnf.bak
cat >> /etc/my.cnf << EOF
# 主库配置
server-id = 1
log_bin = mysql-bin
binlog_format = ROW
expire_logs_days = 7
binlog_ignore_db = mysql
binlog_ignore_db = information_schema
binlog_ignore_db = performance_schema
binlog_ignore_db = sys
EOF

# 重启MySQL
systemctl restart mysqld

# 创建主从同步账号
mysql -uroot -p你的MySQL密码 -e "
CREATE USER 'repl'@'%' IDENTIFIED WITH mysql_native_password BY 'Repl@6666';
GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'repl'@'%';
FLUSH PRIVILEGES;
"

# 锁表，备份全量数据
mysql -uroot -p你的MySQL密码 -e "FLUSH TABLES WITH READ LOCK;"
# 查看主库binlog位置，记录File和Position
mysql -uroot -p你的MySQL密码 -e "show master status;"
# 备份数据
mysqldump -uroot -p你的MySQL密码 --all-databases --single-transaction > /root/master_mysql_backup.sql
# 解锁
mysql -uroot -p你的MySQL密码 -e "UNLOCK TABLES;"
```

### 102、103 节点安装 MySQL，配置从库

```Plain
# 102、103节点安装MySQL
dnf install -y mysql-server mysql
systemctl enable --now mysqld

# 配置从库my.cnf
vim /etc/my.cnf
# 102节点server-id=2，103节点server-id=3
cat >> /etc/my.cnf << EOF
[mysqld]
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
user=mysql
server-id=2  # 102写2，103写3
relay_log=relay-bin
read_only=1
log_bin=mysql-slave-bin
binlog_format=ROW
expire_logs_days=7
replicate_ignore_db=mysql
replicate_ignore_db=information_schema
replicate_ignore_db=performance_schema
replicate_ignore_db=sys

[mysql]
socket=/var/lib/mysql/mysql.sock

[client]
socket=/var/lib/mysql/mysql.sock
EOF

# 102、103节点重启MySQL
systemctl restart mysqld

# 101节点同步备份文件到102、103
scp /root/master_mysql_backup.sql root@node2:/root/
scp /root/master_mysql_backup.sql root@node3:/root/

# 102、103节点导入备份
mysql -uroot -p你的MySQL密码 < /root/master_mysql_backup.sql
```

### 102、103 节点配置主从同步

```Plain
# 102、103节点执行，替换MASTER_LOG_FILE和MASTER_LOG_POS为你记录的主库信息
mysql -uroot -p你的MySQL密码 -e "
CHANGE MASTER TO
MASTER_HOST='192.168.88.101',
MASTER_PORT=3306,
MASTER_USER='repl',
MASTER_PASSWORD='Repl@6666',
MASTER_LOG_FILE='mysql-bin.000001',
MASTER_LOG_POS=156,
MASTER_AUTO_POSITION=0;
START SLAVE;

"# 验证主从同步成功
mysql -uroot -p你的MySQL密码 -e "show slave status\G;"
# 核心：Slave_IO_Running=Yes 和 Slave_SQL_Running=Yes
```

## 第五层：Kafka 3 节点 Kraft 集群

```Plain
# 所有节点安装JDK17
dnf install -y java-17-openjdk java-17-openjdk-devel
java -version

# 所有节点下载Kafka
cd /usr/local
wget https://mirrors.aliyun.com/apache/kafka/3.7.0/kafka_2.13-3.7.0.tgz
tar -zxvf kafka_2.13-3.7.0.tgz
mv kafka_2.13-3.7.0 kafka
cd kafka
mkdir -p data
```

### 生成集群 ID

```Plain
# 101节点执行，记录生成的ID
KAFKA_CLUSTER_ID="$(bin/kafka-storage.sh random-uuid)"
echo $KAFKA_CLUSTER_ID
```

### 每个节点配置 server.properties

#### 101 节点（node.id=1）

```Plain
cat > config/kraft/server.properties << EOF
node.id=1
controller.quorum.voters=1@192.168.88.101:9093,2@192.168.88.102:9093,3@192.168.88.103:9093
listeners=PLAINTEXT://:9092,CONTROLLER://:9093
advertised.listeners=PLAINTEXT://192.168.88.101:9092
inter.broker.listener.name=PLAINTEXT
controller.listener.names=CONTROLLER
log.dirs=/usr/local/kafka/data
num.network.threads=3
num.io.threads=8
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600
num.partitions=3
default.replication.factor=3
min.insync.replicas=2
log.retention.hours=168
log.segment.bytes=1073741824
log.retention.check.interval.ms=300000
EOF
```

#### 102 节点（node.id=2）

```Plain
cat > config/kraft/server.properties << EOF
node.id=2
controller.quorum.voters=1@192.168.88.101:9093,2@192.168.88.102:9093,3@192.168.88.103:9093
listeners=PLAINTEXT://:9092,CONTROLLER://:9093
advertised.listeners=PLAINTEXT://192.168.88.102:9092
inter.broker.listener.name=PLAINTEXT
controller.listener.names=CONTROLLER
log.dirs=/usr/local/kafka/data
num.network.threads=3
num.io.threads=8
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600
num.partitions=3
default.replication.factor=3
min.insync.replicas=2
log.retention.hours=168
log.segment.bytes=1073741824
log.retention.check.interval.ms=300000
EOF
```

#### 103 节点（node.id=3）

```Plain
cat > config/kraft/server.properties << EOF
node.id=3
controller.quorum.voters=1@192.168.88.101:9093,2@192.168.88.102:9093,3@192.168.88.103:9093
listeners=PLAINTEXT://:9092,CONTROLLER://:9093
advertised.listeners=PLAINTEXT://192.168.88.103:9092
inter.broker.listener.name=PLAINTEXT
controller.listener.names=CONTROLLER
log.dirs=/usr/local/kafka/data
num.network.threads=3
num.io.threads=8
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600
num.partitions=3
default.replication.factor=3
min.insync.replicas=2
log.retention.hours=168
log.segment.bytes=1073741824
log.retention.check.interval.ms=300000
EOF
```

### 启动集群

```Plain
# 所有节点格式化，替换成你的集群ID
bin/kafka-storage.sh format -t 你的集群ID -c config/kraft/server.properties

# 所有节点后台启动
nohup bin/kafka-server-start.sh config/kraft/server.properties > logs/server.log 2>&1 &

# 验证集群
bin/kafka-topics.sh --create --topic order_notify --bootstrap-server 192.168.88.101:9092 --partitions 3 --replication-factor 3
bin/kafka-topics.sh --list --bootstrap-server 192.168.88.101:9092
```

## 第六层：Ceph 3 节点分布式存储集群

### 前置要求：3 台节点都新增一块空硬盘（如 /dev/sdb，至少 20G）

```Plain
# 所有节点验证硬盘
lsblk | grep sd
```

### 所有节点安装 Docker

```Plain
dnf install -y dnf-plugins-core
dnf config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
dnf install -y docker-ce docker-ce-cli containerd.io
systemctl enable --now docker
docker -v
```

### 101 节点引导创建集群

```Plain
cd /root
curl --silent --remote-name --location https://github.com/ceph/ceph/raw/reef/src/cephadm/cephadm
chmod +x cephadm
./cephadm add-repo --release reef
./cephadm install
cephadm bootstrap --mon-ip 192.168.88.101
# 保存输出的Dashboard地址、账号密码
```

### 加入 102、103 节点

```Plain
# 101节点分发公钥
ssh-copy-id -f -i /etc/ceph/ceph.pub root@node2
ssh-copy-id -f -i /etc/ceph/ceph.pub root@node3

# 加入集群
ceph orch host add node2
ceph orch host add node3

# 验证节点
ceph orch host ls
```

### 部署服务

```Plain
# 部署3个MON
ceph orch apply mon 3

# 部署MGR
ceph orch apply mgr 2

# 自动发现空硬盘创建OSD
ceph orch apply osd --all-available-devices

# 部署RGW
ceph orch apply rgw myrgw --placement="3 node1 node2 node3"

# 创建商城用户
radosgw-admin user create --uid=shop_user --display-name="商城静态资源用户"

# 保存access_key和secret_key# 验证集群健康
ceph -s
# 输出 health: HEALTH_OK
```

## 第七层：Zabbix 全节点监控

### 101 节点安装 Zabbix Server

```Bash
# 101节点安装Zabbix源
rpm -Uvh https://repo.zabbix.com/zabbix/7.0/centos/9/x86_64/zabbix-release-7.0-2.el9.noarch.rpm
dnf clean all

# 安装Zabbix组件
dnf install -y zabbix-server-mysql zabbix-web-mysql zabbix-nginx-conf zabbix-sql-scripts zabbix-agent

# 创建Zabbix数据库
mysql -uroot -p你的MySQL密码 -e "
CREATE DATABASE zabbix CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;
CREATE USER 'zabbix'@'localhost' IDENTIFIED BY 'Zabbix@6666';
GRANT ALL PRIVILEGES ON zabbix.* TO 'zabbix'@'localhost';
FLUSH PRIVILEGES;
"

# 导入初始数据
zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz | mysql -uzabbix -pZabbix@6666 zabbix

# 配置Zabbix Server
vim /etc/zabbix/zabbix_server.conf
# 修改
DBPassword=Zabbix@6666

# 配置Nginx
vim /etc/nginx/conf.d/zabbix.conf
# 取消注释，修改
listen 8080;
server_name 192.168.88.101;

# 启动服务
systemctl enable --now zabbix-server zabbix-agent nginx php-fpm

# 访问 http://192.168.88.101:8080，默认账号Admin，密码zabbix
```

### 102、103 节点安装 Zabbix Agent

```Plain
# 102、103节点安装源
rpm -Uvh https://repo.zabbix.com/zabbix/7.0/centos/9/x86_64/zabbix-release-7.0-2.el9.noarch.rpm
dnf clean all

# 安装Agent
dnf install -y zabbix-agent

# 修改配置
sed -i 's/^Server=127.0.0.1/Server=192.168.88.101/g' /etc/zabbix/zabbix_agentd.conf
sed -i 's/^ServerActive=127.0.0.1/ServerActive=192.168.88.101/g' /etc/zabbix/zabbix_agentd.conf
sed -i 's/^Hostname=Zabbix server/Hostname=node2/g' /etc/zabbix/zabbix_agentd.conf  # 103写node3# 启动Agent

systemctl enable --now zabbix-agent
```

# 四、最终验证

访问 VIP `http://192.168.88.200`，商城全流程正常，即为部署成功。

## 许可证与声明

- CRMEB 商城系统基于开源协议，可免费商用
- 本文档遵循 MIT 协议，可自由修改、分发，转载请注明出处
- 本文档仅用于学习和技术交流，生产环境部署请做好安全加固和数据备份



