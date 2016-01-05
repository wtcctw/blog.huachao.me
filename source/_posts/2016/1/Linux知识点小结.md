title:  "Linux知识点小结"
date: 2016-01-04
updated: 2016-01-05
categories:
- linux
tags:
- linux
---
> Linux一出，谁与争锋

## $1 我的Linux需求
Linux博大精深。我只在此讨论一些我对线上Linux机器维护人员的基本需求，比如装机，加硬盘，配网络。只讨论CentOS 6，或者类似的RHEL，当然Ubuntu也可以此类推，但是一些新特性不予讨论，因为我不懂，比如CentOS 7的xfs不予讨论，并不是说xfs不好，而是以目前我的Linux水平需要更新很多xfs的知识，驾驭需要时间。CentOS 7将ifconfig，netstat等原来常用的命令也干掉了，用ip，lsof替换是更加好的工具，但是大部分的线上机器都应该还没有更新到CentOS 7。下面我们以CentOS 6作为基础，谈我认为最基本的4点。
### $1.1 最小化安装
CentOS有一个minimal版本，相对于标准版去掉了很多Service，比如Network Manager，安装最小版本以后的网络配置是需要admin进行写配置文件的。我个人认为这样是比较好的，因为这样才能知道Linux内核真正关心的是哪些配置文件，直达核心。一些必要的监控工具，完全可以通过yum install来完成。作为线上机器，还是最小化安装，做到能不开的服务就不开，能关掉的端口就关掉，这样既能将宝贵的硬件资源留下来给应用程序，也能够做到更加的安全。
### $1.2 足够安全
除了将能关的端口关掉，能不用的服务关掉以外，安全还需要做到特定的服务只能访问特定的内容。哪怕是root账户，不能访问的文件和文件夹还是不能访问，更加不能操作。开启SELinux以后，能够做到在不修改SELinux的情况下，指定的服务只能访问指定的资源。对于ssh要做到关闭账户密码登录，只能通过秘钥登录，这样在保证秘钥不被盗用的情况下是最安全的。
### $1.3 资源按需调度
我们经常会遇到这样一个问题，假设将磁盘sda挂载到/var目录，但是由于log太多或者上传的文件等等其他因素将硬盘吃光了，再创建一块sdb磁盘就无法挂载到/var目录了，其实Linux自带的lvm已经解决了这个问题，并且CentOS默认就是用lvm来管理磁盘的。我们需要学会如何格式化一块硬盘为lvm，然后挂载到对应目录，在空间被吃光前能够添加一块硬盘就自动扩容。
### $1.4 网络监控
Linux本地要利用好net_filter，也就是iptables，来规划服务哪些网络流量，抛弃哪些网络流量。以及在进行组网的时候需要用router来进行网关的创建，在遇到网络问题的时候通过netstat来查看网络访问异常。网络这块内容很多很杂，各种参数，TCP/IP协议栈等等，但是往往问题还就是出在网络这块，所以要给与高度的关注。

## $2 Linux的理念与基础
小谈几点我对Linux的认识。
### $2.1 Linux的文件系统
Linux将所有的事物都看成文件，这一点人尽皆知。我想说的是，除了传统的ext文件系统，Linux在抽象不同的资源的时候其实有各种不同的文件系统，都是从需求和使用出发，比如proc文件系统就是针对进程的抽象，使得修改对应进程的值就可以直接改变进程的行为。再比如，对于远程ssh登录的pts设备，Linux有对应的devpts文件系统。看下面表哥的type一栏。

file_system                  |           dir|    type|         options|    dump|    pass
-----------------------------|--------------|--------|----------------|--------|--------
/dev/mapper/VolGroup-lv_root |             /|    ext4|        defaults|       1|       1
UUID=xxx                     |         /boot|    ext4|        defaults|       1|       2
/dev/mapper/VolGroup-lv_swap |          swap|    swap|        defaults|       0|       0
tmpfs                        |      /dev/shm|   tmpfs|        defaults|       0|       0
devpts                       |       /dev/pt|  devpts|   gid=5,mod=620|       0|       0
sysfs                        |          /sys|   sysfs|        defaults|       0|       0
proc                         |         /proc|    proc|        defaults|       0|       0

### $2.2 Linux的权限管理
Linux的`-rwxrwxrwx`权限管理也可谓人尽皆知，其实Linux自己也意识到了这样的权限管理所带来的一些局限性。首先rwx的权限管理是基于用户和组的，并且只是大致的分为`owner|group|other`这三类，无法再作更加细粒度的划分。有鉴于此，Linux目前默认是有ACL(Access Control List)管理的，所谓ACL就是能够提供更加细粒度的用户和组管理，比如可以明确哪个user可以有什么样的权限。如下示例
``` bash
getfacl abc
# file: abc
# owner: someone
# group: someone
user::rw-
user:johny:r-x
group::r--
mask::r-x
other::r--
```
而SELinux提供了不基于用户与组的权限管理，SELinux是基于应用程序的，什么样的应用程序可以使用什么资源，对于这些资源这个应用程序能干嘛，这个就是SELinux的管理方式。

### $2.3 Linux上的Service
Linux上的Service组织得非常清晰，`/etc/init.d/`里面包含了所有的Service启动脚本，对应的二进制文件在`/usr/bin 、 /usr/sbin 、 /usr/local/bin`等目录下，一般而言配置文件在`/etc/app_name`下，还有一个chkconfig的工具来管理各个`runlevel`下需要启动的Service。这样的约定俗成使得管理员在配置和使用的时候非常方便。Linux标准的Service都会将log记录到`/var/log/messages`中，使得系统管理员不需要翻阅各种log，直接在`/var/log/messages`中就可以找到绝大部分的log来判断当前系统是否正常。更甚者，`syslogd`被`rsyslogd`替换以后，可以将/var/log/messages中的内容通过UDP发送到远端用专业的log分析工具进行分析。我们需要学习Linux上Service的这些优秀的编程习惯和技巧。

## $3 磁盘
根据$1中的需求，下面是我记录的一些基本的磁盘操作。
- `df -lah` 查看磁盘的使用情况
- `fdisk -l` 查看插入到磁盘驱动器中的硬盘; sd(a,b,c)(1,2,3)，其中a是第一块磁盘，b是第二块磁盘，1，2，3表示磁盘上的主分区，最多4个。用fdisk从磁盘创建分区并且格式化。
- LVM(logical volume manager)，主要就是满足加硬盘就能直接写数据的功能，而不会出现磁盘满了，新的磁盘只能挂载其他目录的情况。lvm有几个概念，VG, PV。将磁盘lvm格式化，创建PV, 创建VG，将创建的PV加入VG，然后在VG中创建lvm，然后就可以动态增加大小了。注意，将磁盘格式化为lvm，但是lv的格式化需要用ext，然后才能mount上去。参考这篇文章[CentOS 6 卷组挂载硬盘教程](http://www.kwx.gd/CentOSApp/Xen-Centos6-Mounted-HardDrive.html)
- `mount -t type(ext4|nfs) /dev/sdxn /path/dir`  来挂载。如果要重启生效，必须将挂载信息写入到`/etc/fstab`
- 磁盘IO效率(IOPS)需要用`vmstat`, `top`等工具来查看。和性能相关的调优和监控留待后续文章详述。

## $4 网络
网络的坑很多，需要把网络搞通没个3，4年很难。下面从网络的配置文件着手，简单理一下网络方面的内容。网络最难的方面应该是如何搭建一个合理的高效的局域网或者城域网，这个需要有专业的网络知识。
### $4.1 配置文件
- `/etc/hosts`私有IP对应主机名
- `/etc/resolv.conf`nameserver DNS的IP
- `/etc/sysconfig/network`其中NETWORKING=要不要有网络，HOSTNAME=主机名，NETWORKING_IPV6=支持ipv6否
- `/etc/sysconfig/network-scripts/ifcfg-xxx`其中DEVICE=网卡代号，BOOTPROTO=是否使用dhcp，HWADDR，IPADDR，NETMASK，ONBOOT，GATEWAY

### $4.2 与网络有关的一些命令
- `router -n`查看路由的命令，特别是要看带G的，表示gateway，而带U的表示up。
- `netstat -anp`查看所有启动的`tcp`,`udp`,`unix stream`的应用程序，以及他们的状态，具体可以参考[TCP/IP,JavaSocket简单分析](https://blog.huachao.me/2015/12/TCP:IP,Java%20Socket%E7%AE%80%E5%8D%95%E5%88%86%E6%9E%90/)一文。

## $5 安全
### $5.1 PAM
PAM只需要简单了解就行，是一个可插拔的认证模块。我的理解是：开发Linux的极客们搞出来的可复用的一个组件。举个例子，现在有一个app，想要验证当前的登录用户是否有权限操作某个目录，那么在PAM里面有现成的模块，app只需要`include`这个模块，给出一个配置文件，就可以了。有一个非常好的关于PAM的视频教程，请看[这里](http://pan.baidu.com/s/1mhgNwhE)
- PAM是应用程序用来进行身份验证的。早期的身份验证和应用程序本身耦合，后来把身份验证单独抽出来，通过PAM来进行管理
- `/etc/pam.d/xxx` 是能用pam来进行管理的应用程序PAM设置，在安装应用程序的时候安装。`/etc/security/mmm`, `/lib/security/pam_mmm`是一套。

### $5.2 SELinux
SELinux也有一个非常好的视频教程，请看[这里](http://pan.baidu.com/s/1mhgNwhE)
- `getenforce`来查看SELinux是否被启用
- `/etc/sysconfig/selinux enforcing`启用SELinux
- SELinux对“运行程序”配置和检查其是否有权限操作“对象”（文件系统），而普通的ACL(rwx)就是根据文件所属owner及其组来判断。SELinux是看可执行文件的type和目录文件的type是否兼容，来决定可执行文件是否能操作资源

### $5.3 防火墙
下面是学习时候的一些摘录。特别一点，要开启内核参数`net.ipv4.ip_forward=1`，在`/etc/sysctl.conf`文件中，用`sysctl -p`来保存。所谓ip_forward指的是内核提供的从一个iface到另外一个iface的IP包转发，比如将IP包从192.168.1.10的eth0转发到10.0.0.123的eth1上。防火墙配置是需要专业技能的。
- tcp_wrapper需要libwrap.so的支持，可执行文件在`ldd bin_file`出来没有`libwrap.so`的，都不能用tcp_wrapper
- iptables是按照规则进行短路判断的，即 满足条件1->执行action1->结束
- iptables-save来更加清晰的查看
- 先删掉全部规则，然后添加，比较简单。添加的时候，先添加策略，再添加细部规则。一般来讲，我们需要关注的是filter这个表的INPUT与OUTPUT
- iptables -A(I) INPUT(OUTPUT,FORWARD) -i(o) iface -p tcp(ump,imp,all) -s (!)source -d dest -j ACCEPT(REJECT,DROP), 还支持的参数 —dport —sport

## $6 工具
[一个好的Linux命令参考网站](http://linuxtools-rst.readthedocs.org/zh_CN/latest/base/index.html)
### $6.1 CPU
- `top` 特别注意load
- `ps aux`和`ps -ef` 特别注意进程状态
- `vmstat 1`表示每秒采集一次
- `sar -u 1` 查看所有cpu相关的运行时间

### $6.2 Memory
- `free`
- `vmstat 1` 注意其中的swap ram block之间的关系
- `sar -r 1` 内存使用率
- `sar -W 1` 查看swap，查询是否由于内存不足产生大量内存交换

### $6.3 IO
- `lsof -i:port` 查询哪个进程占用了这个端口号
- `lsof -u username` 用户打开的文件
- `lsof -p pid` 进程打开的文件

## 杂项
关于安装好系统之后的运行脚本，这边有一个参考
``` bash
#!/bin/bash
#################################################
#   author  huachao
#   date    2015-12-09
#   email   i@huachao.me
#   web     blog.huachao.me
#################################################

flagFile="/root/centos6-init.executed"

precheck(){

    if [[ "$(whoami)" != "root" ]]; then
    echo "please run this script as root ." >&2
    exit 1
    fi

    if [ -f "$flagFile" ]; then
    echo "this script had been executed, please do not execute again!!" >&2
    exit 1
    fi

    echo -e "\033[31m WARNING! THIS SCRIPT WILL \033[0m\n"
    echo -e "\033[31m *1 update the system; \033[0m\n"
    echo -e "\033[31m *2 setup security permissions; \033[0m\n"
    echo -e "\033[31m *3 stop irrelevant services; \033[0m\n"
    echo -e "\033[31m *4 reconfig kernel parameters; \033[0m\n"
    echo -e "\033[31m *5 setup timezone and sync time periodically; \033[0m\n"
    echo -e "\033[31m *6 setup tcp_wrapper and netfilter firewall; \033[0m\n"
    echo -e "\033[31m *7 setup vsftpd; \033[0m\n"
    sleep 5
    
}

yum_update(){
    yum -y update
    #update system at 5:40pm daily
    echo "40 3 * * * root yum -y update && yum clean packages" >> /etc/crontab
}

permission_config(){
    #chattr +i /etc/shadow
    #chattr +i /etc/passwd
}

selinux(){
    sed -i 's/SELINUX=disabled/SELINUX=enforcing/g' /etc/sysconfig/selinux
    setenforce 1
}

stop_services(){
    for server in `chkconfig --list |grep 3:on|awk '{print $1}'`
    do
        chkconfig --level 3 $server off
    done
     
    for server in crond network rsyslog sshd iptables
    do
       chkconfig --level 3 $server on
    done
}

limits_config(){
cat >> /etc/security/limits.conf <<EOF
* soft nproc 65535
* hard nproc 65535
* soft nofile 65535
* hard nofile 65535
EOF
echo "ulimit -SH 65535" >> /etc/rc.local
}

sysctl_config(){
sed -i 's/net.ipv4.tcp_syncookies.*$/net.ipv4.tcp_syncookies = 1/g' /etc/sysctl.conf
sed -i 's/net.ipv4.ip_forward.*$/net.ipv4.ip_forward = 1/g' /etc/sysctl.conf
cat >> /etc/sysctl.conf <<EOF
net.ipv4.tcp_max_syn_backlog = 65536
net.core.netdev_max_backlog =  32768
net.core.somaxconn = 32768
net.core.wmem_default = 8388608
net.core.rmem_default = 8388608
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_timestamps = 0
net.ipv4.tcp_synack_retries = 2
net.ipv4.tcp_syn_retries = 2
net.ipv4.tcp_tw_recycle = 1
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_mem = 94500000 915000000 927000000
net.ipv4.tcp_max_orphans = 3276800
net.ipv4.ip_local_port_range = 1024  65535
EOF
sysctl -p
}

sshd_config(){
    if [ ! -f "/root/.ssh/id_rsa.pub" ]; then
    ssh-keygen -t rsa -P '' -f /root/.ssh/id_rsa
    cat /root/.ssh/id_rsa.pub >> /root/.ssh/authorized_keys
    chmod 600 /root/.ssh/authorized_keys
    fi

    #sed -i '/^#Port/s/#Port 22/Port 65535/g' /etc/ssh/sshd_config
    sed -i '/^#UseDNS/s/#UseDNS no/UseDNS yes/g' /etc/ssh/sshd_config
    #sed -i 's/#PermitRootLogin yes/PermitRootLogin no/g' /etc/ssh/sshd_config
    sed -i 's/#PermitEmptyPasswords yes/PermitEmptyPasswords no/g' /etc/ssh/sshd_config
    sed -i 's/PasswordAuthentication yes/PasswordAuthentication no/g' /etc/ssh/sshd_config
    /etc/init.d/sshd restart
}

time_config(){
    #timezone
    echo "TZ='Asia/Shanghai'; export TZ" >> /etc/profile

    # Update time
    if [! -f "/usr/sbin/ntpdate"]; then
        yum -y install ntpdate
    fi
    
    /usr/sbin/ntpdate pool.ntp.org
    echo "30 3 * * * root (/usr/sbin/ntpdate pool.ntp.org && /sbin/hwclock -w) &> /dev/null" >> /etc/crontab
    /sbin/service crond restart
}

iptables(){
cat > /etc/sysconfig/iptables << EOF
# Firewall configuration written by system-config-securitylevel
# Manual customization of this file is not recommended.
*filter
:INPUT DROP [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
:syn-flood - [0:0]
-A INPUT -i lo -j ACCEPT
-A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 22 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 80 -j ACCEPT
-A INPUT -p icmp -m limit --limit 100/sec --limit-burst 100 -j ACCEPT
-A INPUT -p icmp -m limit --limit 1/s --limit-burst 10 -j ACCEPT
-A INPUT -p tcp -m tcp --tcp-flags FIN,SYN,RST,ACK SYN -j syn-flood
-A INPUT -j REJECT --reject-with icmp-host-prohibited
-A syn-flood -p tcp -m limit --limit 3/sec --limit-burst 6 -j RETURN
-A syn-flood -j REJECT --reject-with icmp-port-unreachable
COMMIT
EOF
/sbin/service iptables restart
source /etc/profile
}

other(){
    # initdefault
    sed -i 's/^id:.*$/id:3:initdefault:/' /etc/inittab
    /sbin/init q
    
    # PS1
    #echo 'PS1="\[\e[32m\][\[\e[35m\]\u\[\e[m\]@\[\e[36m\]\h \[\e[31m\]\w\[\e[32m\]]\[\e[36m\]$\[\e[m\]"' >> /etc/profile
     
    # Wrong password five times locked 180s
    sed -i '4a auth        required      pam_tally2.so deny=5 unlock_time=180' /etc/pam.d/system-auth
}

vsftpd_setup(){
    yum -y install vsftpd
    mv /etc/vsftpd/vsftpd.conf /etc/vsftpd/vsftpd.conf.bak
    touch /etc/vsftpd/chroot_list
    setsebool -P ftp_home_dir=1
cat >> /etc/vsftpd/vsftpd.conf <<EOF
# normal user settings
local_enable=YES
write_enable=YES
local_umask=022
chroot_local_user=YES
chroot_list_enable=YES
chroot_list_file=/etc/vsftpd/chroot_list
local_max_rate=10000000
# anonymous settings
anonymous_enable=YES
no_anon_password=YES
anon_max_rate=1000000
data_connection_timeout=60
idle_session_timeout=600
# ssl settings
#ssl_enable=YES             
#allow_anon_ssl=NO           
#force_local_data_ssl=YES    
#force_local_logins_ssl=YES  
#ssl_tlsv1=YES               
#ssl_sslv2=NO
#ssl_sslv3=NO
#rsa_cert_file=/etc/vsftpd/vsftpd.pem 
# server settings
max_clients=50
max_per_ip=5
use_localtime=YES
dirmessage_enable=YES
xferlog_enable=YES
connect_from_port_20=YES
xferlog_std_format=YES
listen=YES
pam_service_name=vsftpd
tcp_wrappers=YES
#banner_file=/etc/vsftpd/welcome.txt
dual_log_enable=YES
pasv_min_port=65400
pasv_max_port=65410
EOF
    chkconfig --level 3 vsftpd on
    service vsftpd restart
}
 
main(){
    precheck
    
    printf "\033[32m================%40s================\033[0m\n" "updating the system            "
    yum_update

    printf "\033[32m================%40s================\033[0m\n" "re-config permission           "
    permission_config

    printf "\033[32m================%40s================\033[0m\n" "enabling selinux               "
    selinux

    printf "\033[32m================%40s================\033[0m\n" "stopping irrelevant services   "
    stop_services
    
    printf "\033[32m================%40s================\033[0m\n" "/etc/security/limits.config    "
    limits_config
    
    printf "\033[32m================%40s================\033[0m\n" "/etc/sysctl.conf               "
    sysctl_config

    printf "\033[32m================%40s================\033[0m\n" "sshd re-configuring            "
    sshd_config
    
    printf "\033[32m================%40s================\033[0m\n" "configuring time               "
    time_config
    
    printf "\033[32m================%40s================\033[0m\n" "configuring firewall           "
#   iptables
    
    printf "\033[32m================%40s================\033[0m\n" "someother stuff                "
    other

    printf "\033[32m================%40s================\033[0m\n" "done! rebooting                "
    touch "$flagFile"
    sleep 5
    reboot
}

main
```