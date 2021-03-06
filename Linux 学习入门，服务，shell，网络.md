[TOC]

# 1 acl访问控制列表（acl策略）

只有Linux有，能够对个别用户，个别组设置独立的权限

| setfacl | -m 修改权限<br>-b 取消acl策略<br>-x 删除指定权限<br>-r 对目录下所有文件与目录都更改 |
| ------- | ------------------------------------------------------------ |
| getfacl | 查看acl权限                                                  |

```shell
setfacl -m u:用户名:权限 文档  # 修改用户的acl策略
setfacl -m g:组名:权限 文档  # 修改组的acl策略	
```



# 2 LDAP网路用户认证

作用：网络用户认证，用户集中管理

集群服务器中，一台LDAP服务器配置用户，其他服务器询问认证

| 客户机连接LDAP服务 | sssd           |
| ------------------ | -------------- |
| 客户机图形配置工具 | authconfig-gtk |
| 默认端口           | 389/TCP        |
| 加密端口           | 636/TCP        |



# 3 NFS 网络文件系统

| 协议         | NFS（2049/TCP） RPC（111/TCP） |
| ------------ | ------------------------------ |
| 文件系统格式 | nfs                            |
| 软件         | nfs-utils                      |
| 服务         | nfs-server                     |
| 配置文件     | /etc/exports                   |

```shell
showmount -e IP	#查看网络分享
mount ip:目录 挂载点 #将网络文件系统临时挂载到本地

#nfs配置文件格式
"共享文件夹绝对路径" 客户机IP（ro/rw） #客户机IP 可以写成*

#nfs永久挂载
IP：目录 挂载点 nfs defaults,_netdev 0 0
```



# 4 NTP 网络时间同步

| 软件包名 | chrony           |
| -------- | ---------------- |
| 服务名   | chronyd          |
| 配置文件 | /etc/chrony.conf |
| 端口     | 123/UDP          |

```shell
#配置文件写法
server IP iburst # iburst 快速同步

systemctl restart chronyd # 重启chronyd服务
systemctl enable chronyd # 设置chronyd服务开机启动
```



# 5 SELinux 强制访问控制系统

针对用户，进程和文件提供了预设的保护策略，以及管理工具，集成于Linux内核（2.6版本及以上）

| 运行模式 | enforing 强制<br>permissive 宽松<br>disabled 关闭 |
| -------- | ------------------------------------------------- |
| 配置文件 | /etc/selinux/config                               |

```shell
setenforce 1/0 # 临时切换至 强制/宽松 模式。宽松模式记录违规，恢复强制模式后处理。
# 任何模式切换至disabled模式，需要修改配置文件后重启系统，反之亦然。
```

```shell
getsebool [-a] [布尔值条款] # 列出目前系统上面的所有布尔值条款设置

setsebool "条款名" on/off # 设置开关 -P 设置为永久
```



# 6 FTP服务

| 服务端软件包名 | vsftpd                  |
| -------------- | ----------------------- |
| 客户端软件包名 | ftp                     |
| 服务名         | vsftpd                  |
| 配置文件       | /etc/vsftpd/vsftpd.conf |
| 默认共享目录   | /var/ftp                |
| 端口           | 21/tcp                  |

```shell
# 添加防火墙策略，让火墙允许ftp服务； --permanent表示永久添加
firewall-cmd --permanent --add-service="ftp"
```



# 7 Firewall防火墙策略管理

作用：隔离、过滤所有<font color=red>入站</font>的请求

防火墙判断规则：

1. 匹配源IP与规则内IP，匹配成功就进入此区域
2. 匹配失败进入默认区域

| 服务名，默认安装并自启 | firewalld                                                    |
| ---------------------- | ------------------------------------------------------------ |
| 命令行管理工具         | firewall-cmd                                                 |
| 图形化管理工具         | firewall-config                                              |
| 运行规则               | runtime（运行时）<br>permanent（永久）                       |
| 预设保护规则集（区域） | public（默认仅允许访问本地的sshd、ping与dhcp三个服务）<br>trusted（允许任何访问）<br>block（明确拒绝所有访问）<br>drop（直接丢弃，节省资源） |

```shell
firewall-cmd --get-default-zone # 查看当前激活区域
firewall-cmd --set-default-zone="区域名" # 设置激活区域，此操作为永久
firewall-cmd --zone="区域名" --list-all # 查看指定区域的详细信息
firewall-cmd --reload # 重载防火墙规则
firewall-cmd --zone="区域名" --add-service="协议名" # 临时增加协议，添加 --permanent 设置为永久，设置后需要重载防火墙规则
firewall-cmd --zone="区域名" --add-source="IP" # 设置IP进入哪个区域

#端口转发
firewall-cmd --zone="区域名" --add-forward-port=port="对外端口"：proto=tcp:toport="内部端口"
```



# 8 配置高级连接

## 配置IPV6

|            | IPV4地址   | IPV6地址    |
| ---------- | ---------- | ----------- |
| 组成       | 32位二进制 | 128位二进制 |
| 分隔符     | .          | :           |
| 表示方式   | 十进制     | 十六进制    |
| 分几个部分 | 4          | 8           |

一张网卡可同时配置IPV4与IPV6，配置方式一样

ping6 可以测试IPV6网络



## 聚合连接（链路聚合）

由多块网卡组成，实现网卡备份，网卡设备的高可用性。

由虚拟网卡管理真实网卡，5秒询问一次。

两种工作方式：

  - ​	轮询式（roundrobin）：流量负载均衡

  - ​	热备份（activebackup）：连接冗余

```shell
#聚合连接配置
#创建虚拟网卡
nmcli connection add com-name team0 type team ifname team0 config '{"runner":{"name":"activebackup"}}' # com-name 后跟配置文件名称 ifname 后跟查询时显示的网卡名
  
#配置IP
nmcli connection modify team0 ipv*.method manual ipv*.addresses "IP/NETMARK" connection.autoconnect yes
  
#添加成员
nmcli connection add com-name team0-p1 type team-slave ifname eth1 master team0 # ifname后跟成员

#删除网卡
nmcli connection delete "网卡名"

#查询虚拟网卡状态
teamdctl "网卡名" state

#nmcli 未识别的网卡可临时设置IP，设置后可显示
ifconfig eth1 "192.168.1.1" netmask "255.255.255.0" up
```

  

# 9 SMB共享

共享文件夹，Linux与Windows平台之间也可用。

| 协议   | SMB             | cifs                 |
| ------ | --------------- | -------------------- |
| 作用   | 验证协议        | 文件系统格式         |
| 软件包 | samba（服务端） | cifs_utils（客户机） |
| 服务   | smb（服务端）   |                      |

```shell
#配置文件
/etc/samba/smb.conf # ; 代表注释
#内容（尾部追加），以下命令，用哪个写哪个
[自定义共享名]
	path = "文件绝对路径"
	public = no/yes # 默认no
	browseable = yes/no # 默认yes
	readonly = yes/no # 默认yes
	write list = "用户名" # 默认无
	valid users =
	hosts allow =
	hosts deny =
```

| 命令    | 选项                                                         |
| ------- | ------------------------------------------------------------ |
| pdbedit | -a 用户名 # 添加smb用户<br/>-L # 查看smb用户 <br/>-x 用户名 # 删除smb |

客户机查看

```shell
# 安装 samba-client
smbclient -L IP/共享名 # 查看共享的目录
smbclient -U 用户名 # IP/共享名 #连接共享
```

客户机临时、永久挂载

```shell
# 安装 cifs_utils 包后，挂载
#临时挂载
mount -o username= ,password= # IP/共享名 "挂载点" # 格式一
mount -o user= ,pass= # IP/共享名 "挂载点" # 格式二

#永久挂载，追加至/etc/fstab
# IP/共享名 "挂载点" cifs defaults，user=,pass=,_netdev 0 0 # _netdev 声明网络设备，系统完全启动后挂载

```

multiuser 多用户模式，可以临时变成权限较大的用户
	配置文件参数中添加 multiuser,sec=ntlmssp # ntlmssp 提供NT局域网管理安全协议

```shell
cifscreds add -u 用户名 IP # 切换smb登陆用户
```

安装步骤：

- ​	1.安装samba
- ​	2.创建专用用户
- ​	3.创建共享目录
- ​	4.修改配置文件
- ​	5.重启服务
- ​	6.修改SElinux，samba的bool
- ​	7.设置acl  
- ​	8.客户机安装cifs_utils，挂载

```shell
getsebool -a | grep samba # 查看SELinux 管理的所有接口，筛选samba相关
setsebool "接口" on # 开启接口
```



# 10 ISCSI

共享一个<font color=red>未格式化</font>的分区或硬盘。服务端提供磁盘，客户端连接并本地使用。

| 软件   | targetcli                                                    |
| ------ | ------------------------------------------------------------ |
| 服务名 | target                                                       |
| 端口   | 3260/TCP                                                     |
| 组成   | backstore # 后端存储，对应到服务端的实际设备，需要一个管理名称<br>lun # 逻辑单元，每一个lun对应一个后端设备，在客户端视为一块虚拟硬盘<br>target # 磁盘组，客户端访问目标，作为一个框架，多个lun组成 |



## 服务端操作

```shell
targetcli # 启动配置软件
/backstores/block create name="后端存储名字" # 创建后端存储
/iscsi/ create iqn.2019-04.com.example:server0 # 创建磁盘组
# 磁盘组 IQN 命名规范： iqn.yyyy-mm.倒序域名：自定义标识
/iscsi/磁盘组名/tpg1/luns create /backstores/block/"后端存储名" # lun与后端存储关联
/iscsi/磁盘组名/tpg1/acls create iqn.2019-04.com.example:desktop0 # 创建acls用户，用于远程连接，用户名必须iqn规范
/iscsi/磁盘组名/tpg1/portals create IP # 启动监听IP与默认端口
exit # 退出时自动创建配置文件

systemctl restart target # 启动target服务
systemctl enabled target # 开机自动启动target服务
```



## 客户端操作

| 软件     | iscsi-initiator-utils          |
| -------- | ------------------------------ |
| 配置文件 | /etc/iscsi/initiatorname.iscsi |
| 服务     | iscsi、iscsid                  |

```shell
#修改配置文件
InitiatorName=iqn.2019-04.com.example:desktop0 # 填写acls用户名，不能有任何空格

systemctl restart iscsid # 重启iscsid服务，无法开机自启。刷新acls配置名称

#发现iscsi磁盘方法一
iscsiadm -m discovery -t st -p "IP"
#发现iscsi磁盘方法二，利用 man iscsiadm 查找范例
iscsiadm --mode discoverydb --type sendtargets --portal "IP" --discover

systemctl restart iscsi # 重启iscsi服务，连接iscsi磁盘，并可以使用。

#开机启动iscsi磁盘
systemctl enabled iscsi
#修改配置文件 /var/lib/iscsi/nodes/磁盘组名/IP:端口/default
node.conn[0].startup=automatic # 设置开机启动
```



# 11 Mariadb 基础

常见数据库：oracle、mysql、mariadb、sql-server

| 软件包 | mariadb-server |
| ------ | -------------- |
| 服务   | mariadb        |
| 端口   | 3306/TCP       |

```mysql
mysqladmin -u root password '新密码' # 第一次设置密码
mysqladmin -u root password '旧密码' '新密码'  # 修改密码

# 以下命令进入mysql执行，不能自动补全。
SHOW DATABASES； # 显示所有数据库
CREATE DATABASE "数据库名"; # 创建数据库
DROP DATABASE "数据库名"; # 删除数据库
DESC "表名"; # 查询表结构

# 数据库授权
GRANT "权限列表" on "数据库名"."表名" to "用户名"@"客户地址" identified by "密码";
# 权限列表：all、select、insert、updata、delete
flush privileges; # 刷新配置
```



# 12 httpd 服务

| web框架  | web 基于B/S 架构 Browser/Serve                       |
| -------- | ---------------------------------------------------- |
| 常见软件 | hpptd（Apache）、nginx、tomcat、Tengine（nginx优化） |
| 端口     | 80/TCP                                               |

```shell
# httpd 默认主页 /var/www/html/index.html
# httpd 主配置文件 /etc/httpd/conf/httpd.conf
Listen # 监听端口
ServerName # 本站服务的域名
DocumentRoot # 网页根目录
DirectoryIndex # 默认主页文件

hpptd -t # 自检配置文件
```



## 虚拟主机

一台服务器可以提供多个域名web页面。可基于三种方式区分：IP、域名、端口。

```shell
# 副配置文件：设置权限和虚拟主机 /etc/httpd/conf.d/*.conf
# 虚拟主机格式
Listen "端口"
<VirtualHost "IP":"端口"> # "IP" 可以写 *
	ServerName 域名
	DocunmentRoot 网站根目录
</irtualHost>

# 访问控制
<Directory "目录">
	Require all denied/granted # 全部拒绝或允许
	Require IP
</Directory>
```

若网站根目录不在 /var/www 下，需要处理<font color=red>SELinux</font>和<font color=red>访问控制</font>权限

```shell
# 处理SELinux
semanage fcontext -l # 查看所有上下文
ls -Zd 目录 # 查看目录上下文
chcon -R --reference=模板目录 新目录 # 循环复制模板目录的上下文到新目录
```



## 动态网站（wsgi）

需要软件：mod_wsgi

```shell
<VirtualHost "IP":"端口"> # "IP" 可以写 *
	ServerName 域名
	DocunmentRoot 网站根目录
	WSGIScriptAlias / /var/www/html/wsgi/index.wsgi #打开根目录，自动调转
</irtualHost>
```



## 更改任意端口

```shell
semanage port -l # 查看所有SELinux默认端口
semange port -a -t http_port_t -p tcp 8909 # 开放httpd监听8909端口
```



## 安全的WEB

| 软件 | mod_ssl                       |
| ---- | ----------------------------- |
| 协议 | https（安全的超文本传输协议） |

```shell
/etc/pki/tls/certs/*.crt # 证书存放位置
/etc/pki/tls/private/*.key # 私钥存放位置
/etc/httpd/conf.d/ssl.conf # 配置ssl
Listen 443 https
.. ..
<VirtualHost _default_:443>
DocumentRoot "/var/www/html"                                     # 网页目录
ServerName server0.example.com:443                              # 站点的域名
.. ..
SSLCertificateFile /etc/pki/tls/certs/server0.crt                  # 网站证书
.. ..
SSLCertificateKeyFile /etc/pki/tls/private/server0.key             # 网站私钥
.. .. 
SSLCACertificateFile /etc/pki/tls/certs/example-ca.crt             # 根证书
```



# 13 postfix基础邮件服务

| 软件包 | postfix |
| ------ | ------- |
| 服务   | postfix |
| 端口   | 25/TCP  |

```shell
# vim  /etc/postfix/main.cf
.. ..
inet_interfaces = all                         #监听接口
mydomain = example.com                          #邮件域,默认补全的域名
myhostname = example.com                          #本服务器主机名

mail -s "标题" -r "发件人" "收件人" # 回车后写正文，最后一行以单个 . 结尾
mail -u "用户名" # 查看指定用户邮件
```



# 14 swap分区、parted 分区工具

以分区充当swap分区

parted 分区工具，支持GPT分区模式，最多128个主分区，18EB空间。

自动改变硬盘文件系统，分区空间有误差

```shell
mktable gpt # 指定分区模式
mkpart # 文件系统只记录
unit GB # 设置primt单位
```

```shell
# 制作swap分区
mkswap "目录" # 格式化生成文件系统
swapon "目录" # 启用swap分区
swapon  -s # 查看swap分区，权限为优先级，数值越大，优先级越高
swapoff "目录" # 停用swap分区

#开机启动
/dev/* swap swap defaults 0 0
swapon -a # 检测开机启动
```

# 15 DNS

正向解析：域名变IP	反向解析：IP变域名

结构：根域、一级DNS服务器、二级DNS服务器、三级DNS服务器

所有域名必须以点结尾 点称为根域

FQDN 完全合格主机名，格式 站点名.域名.

IANA 互联网数字分配机构

域名解析步骤：

1. ​	查看/etc/hosts
2. ​	查看/etc/resolve.conf（存放DNS服务器IP）
3. ​	给DNS服务器发送请求
4. ​	DNS服务器递归查询
5. ​	DNS服务器与其它服务器迭代查询
6. ​	返回结果

| 软件包             | bind（域名服务）、bind-chroot（虚拟根支持，安全） |
| ------------------ | ------------------------------------------------- |
| 服务名             | named                                             |
| 端口               | 53/UDP                                            |
| 主配置文件         | /etc/named.conf（设置负责解析的域名）             |
| 运行时的虚拟根环境 | /var/named/chroot/                                |
| 地址库文件夹       | /var/named                                        |

```shell
vim  /etc/named.conf                             # 建立新配置
options {
	recursion yes/no;								# 开启/关闭递归查询
    directory  "/var/named";                          # 地址库默认存放位置
    forwarders { IP;IP; }；					#启动DNS缓存，IP为DNS服务器IP
};
zone  "zz.cn" {                                  # 定义正向DNS区域
    type  master;                                     # 主区域
    file  "zz.cn.zone";                             # 自定义地址库文件名
};

cp  -p  named.localhost  zz.cn.zone      #参考范本建地址库文件，需要特别注意 -p

vim  tedu.cn.zone                          # 修订地址库记录
$TTL 1D                                          # 文件开头部分可保持不改
@   IN SOA  @ rname.invalid. (
                    0   ; serial
                    1D  ; refresh
                    1H  ; retry
                    1W  ; expire
                    3H )    ; minimum
@       NS  svr7.zz.cn.                       # 本区域DNS服务器的FQDN
svr7    A   192.168.4.7                       # 为DNS主机提供A记录
pc207   A   192.168.4.207                     # 其他正向地址记录.. ..
www     A   192.168.4.100                     # A 代表IPV4  AAAA 代表IPV6
www     A   192.168.4.110                     # 配置DNS轮询
www     A   192.168.4.120
*       A   119.75.217.56                     # 配置多对一的泛域名解析
```



## 配置DNS子域授权



```shell
# 子域域名.            IN    NS      子DNS的FQDN.
# 子DNS的FQDN.       IN    A        子DNS的IP地址

bj.zz.cn.         NS       pc207.bj.zz.cn.             #子区域及子DNS主机名
pc207.bj.zz.cn.   A       192.168.5.207                  #子DNS的IP地址
```



## 配置Split分离解析

```shell
# 每个区域的个数必须一致，域名必须一致
vim  /etc/named.conf
options {
        directory  "/var/named";
};
acl "mylan" {                                      # 名为mylan的列表
        192.168.4.207; 192.168.7.0/24;
};

view "mylan" {
    match-clients { mylan; };                      # 检查客户机地址是否匹配此列表
    zone "zz.cn" IN {
        type master;
        file "zz.cn.zone.lan";
    };
};

view "other" {
    match-clients { any; };                          # 匹配任意客户机地址
    zone "zz.cn" IN {
        type master;
        file "zz.cn.zone.other";
    };
};
```



# 16 RADIO

| 模式                | 磁盘数    | 优、缺点                         |
| ------------------- | --------- | -------------------------------- |
| 0 条带模式          | 大于等于2 | 没有容错、I/O性能高              |
| 1 镜像模式          | 大于等于2 | 有备份                           |
| 5 高性价比模式      | 大于等于3 | 利用1块磁盘存储校验，可坏1块磁盘 |
| 6 高性价比/可靠模式 | 大于等于4 | 利用2块磁盘存储校验，可坏2块磁盘 |
| 1+0 或 0+1          | 大于等于4 |                                  |



# 17 进程管理

| 命令   | 选项                                                         |
| ------ | ------------------------------------------------------------ |
| pstree | -a # 显示完整的命令行信息<br>-p # 列出对应的PID号，systemd（所有进程的父进程，上帝进程） |
| ps     | aux # a 所有进程（当前终端） u 以用户格式输出 x 当前用户在所有终端下的进程<br>-elf # e 所有进程（系统内）l 长格式显示 f 完整的进程信息（PPID为父进程号） |
| top    | -d 动态进程刷新秒数<br>-U 用户名<br>交互命令<br>? # 查看帮助<br>P、M # 根据%CPU、%MEM 降序排列<br/>T # 根据消耗时间降序排列<br/>K # 杀死指定的进程<br/>q # 退出top |
| pgrep  | -l 进程名 # 检索指定名称进程<br/>-U 用户名 # 检索指定用户进程<br/>-t 终端号 # 检索终端进程<br/>-x # 精确匹配完整的进程名 |

图形命令行终端pts/0 # 0代表第一个，依次排序

## 后台进程

| 命令 &   | 程序运行状态放入后台                           |
| -------- | ---------------------------------------------- |
| Ctrl+z   | 暂停并转入后台                                 |
| jobs     | 列出当前用户当前终端的后台任务<br>-l 显示PID号 |
| fg 编号  | 启动指定编号的后台任务，并恢复至前端           |
| bg  编号 | 启动指定编号的后台任务                         |



## 杀死进程

| kill  [-9]  PID       | 杀死指定PID值的进程          |
| --------------------- | ---------------------------- |
| killall  [-9]  进程名 | 杀死指定名称的所有进程       |
| pkill                 | 根据指定的名称或条件杀死进程 |

# 18 系统日志

| 目录              | 作用                             |
| ----------------- | -------------------------------- |
| /var/log/messages | 记录内核消息、各种服务的公共消息 |
| /var/log/dmesg    | 记录系统启动过程的各种消息       |
| /var/log/cron     | 记录与cron计划任务相关的消息     |
| /var/log/maillog  | 记录邮件收发相关的消息           |
| /var/log/secure   | 记录与访问限制相关的安全消息     |
| /var/log/lastlog  | 最近用户的登陆行为               |
| /var/log/wtmp     | 成功的登陆/注销                  |
| /var/log/btmp     | 失败的登陆                       |
| /var/log/utmp     | 当前登录的用户                   |

```shell
tailf /var/log/messages # 将输出文件的最后10行，然后等待文件增长
# -n 指定显示文件最后的行数(默认显示最后10行)
```



## 日志消息级别

| EMERG（紧急）   | 级别0，系统不可用的情况         |
| --------------- | ------------------------------- |
| ALERT（警报）   | 级别1，必须马上采取措施的情况   |
| CRIT（严重）    | 级别2，严重情形                 |
| ERR（错误）     | 级别3，出现错误                 |
| WARNING（警告） | 级别4，值得警告的情形           |
| NOTICE（注意）  | 级别5，普通但值得引起注意的事件 |
| INFO（信息）    | 级别6，一般信息                 |
| DEBUG（调试）   | 级别7，程序/服务调试消息        |



# 19 用户登陆查询

查询结果简单到详细：users、who、w

last 查看最近登陆成功的信息

lastb  查看最近登陆失败的信息



# 20 PXE批量装机

| 所需服务与工具                      | 作用                    |
| ----------------------------------- | ----------------------- |
| DHCP                                | 分配IP、定位引导文件    |
| TFTP                                | 提供引导下载            |
| HTTP                                | 提供yum源、自动应答文件 |
| system-config-kickstart（图形工具） | 生成自动应答文件        |



## DHCP 服务器

| 软件包   | dhcp                 |
| -------- | -------------------- |
| 服务名   | dhcpd                |
| 配置文件 | /etc/dhcp/dhcpd.conf |

整个服务过程以广播进行，先到先得，一个网络中，只能有一个DHCP服务器。

```shell
# vim 末行模式 ：r 文件路径  # 读取文件到当前光标之下
# vim  /etc/dhcp/dhcpd.conf
subnet 192.168.4.0 netmask 255.255.255.0 { # 声明网段
     range  192.168.4.10 192.168.4.200; # IP范围
     option domain-name-servers DNSIP；
     option routers 网关IP；
     next-server  192.168.4.7;  # 引导服务IP
     filename  "pxelinux.0"; # 引导文件、二进制文件。由syslinux软件提供（redhat7独有）
}

# pxelinux.0 位置 /usr/share/syslinux/pxelinux.0
# pxelinux.0 内容 /var/lib/tftpboot/pxelinux.cfg/default
```



## TFTP 服务

| 软件包       | tftp-server       |
| ------------ | ----------------- |
| 服务名       | tftp              |
| 端口         | 69/TCP            |
| 默认共享路径 | /var/lib/tftpboot |

需从光盘内复制

- ​	isolinux.cfg（菜单配置，/var/lib/tftpboot/pxelinux.cfg/default）
- ​	vesamenu.c32（提供图形支持）
- ​	splash.png （背景图片）
- ​	vmlinuz（内核）
- ​	initrd.img（初始化文件）

```shell
vim  /var/lib/tftpboot/pxelinux.cfg/default
default vesamenu.c32                             # 默认交给图形模块处理
timeout 600                                      # 选择限时为60秒（单位1/10秒）
.. ..
menu title  PXE  Installation  Server             # 启动菜单标题信息
.. ..
label  linux                                  # 菜单项标签
    menu  label  ^Install Red Hat Enterprise Linux 7 
    menu  default                              # 默认启动方式
    kernel  vmlinuz                   		   # 内核的位置
    append  initrd=rhel7/initrd.img  ks=http：# IP/应答文件
```

​	

# 21 rsync 同步

- rsync  [选项...]  本地目录1  本地目录2  （把目录1同步到目录2下）
- rsync  [选项...]  本地目录1/  本地目录2 （把目录1下的内容同步到目录2下）
- 可以利用ssh远程同步

| 命令  | 选项                                                         |
| ----- | ------------------------------------------------------------ |
| rsync | -n # 测试同步过程，不做实际修改<br>-a # 归档模式<br/>-v # 显示详细操作信息<br/>-z：传输过程中启用压缩/解压(超过1G使用)<br/>--delete：删除目标文件夹内多余的文档 |



# 22 inotifywait 事件监控

| 软件包 | inotify-tools                                                |
| ------ | ------------------------------------------------------------ |
| 程序名 | inotifywait                                                  |
| 格式   | inotifywait  [选项]  目标文件夹                              |
| 选项   | -m # 持续监控（捕获一个事件后不退出）<br>-r # 递归监控、包括子目录及文件<br/>-q # 减少屏幕输出信息<br/>-e # 指定监视的 modify、move、create、delete、attrib 等事件类别  <br/>-qq # 不输出屏幕信息 |



## 实时同步

```shell
ssh-kengen # 生成公钥与私钥
ssh-copy-id root@IP # 将公钥给对方，私钥登陆公钥

while /目录/inotifywait -qqr /目录/
do
	rsync -av 目录 root@IP：目录
done
```



# 23 网络

## 网络拓扑结构

| 点对点 | 广域网                                                       |
| ------ | ------------------------------------------------------------ |
| 星型   | 易实现，易网络扩展，易故障排查。中心节点压力大，组网成本较高 |
| 网状   | 冗余性、容错性和可靠性高。组网成本高                         |



## OSI 七层模型（理论模型）

| 层级       | 作用                                             | 代表                                                         |
| ---------- | ------------------------------------------------ | ------------------------------------------------------------ |
| 应用层     | 网络服务与最终用户的一个接口                     | Telnet、FTP、HTTP、SNMP等                                    |
| 表示层     | 数据的表示、安全、压缩                           | 格式有，JPEG、ASCll、DECOIC、加密格式等                      |
| 会话层     | 建立、管理、终止会话                             | 对应主机进程，指本地主机与远程主机正在进行的会话             |
| 传输层     | 定义传输数据的协议端口号，以及流控和差错校验     | 防火墙、端口。协议有：TCP UDP，数据包一旦离开网卡即进入网络传输层 |
| 网络层     | 进行逻辑地址寻址，实现不同网络之间的路径选择     | 路由器。协议有：CMP IGMP IP（IPV4 IPV6） ARP RARP            |
| 数据链路层 | 建立逻辑连接、进行硬件地址寻址、差错校验等功能。 | 交换机                                                       |
| 物理层     | 建立、维护、断开物理连接。                       | 物理设备                                                     |



## TCP/IP 五层模型（实际应用）

| 应用层 | 数据 | 计算机 |
| ------ | ---------------------------- | ------------------------- |
| 传输层     | 数据段  | 防火墙 |
| 网络层   | 数据包 | 路由器            |
| 数据链路层 | 数据帧 | 交换机                                                       |
| 物理层     | 比特流                   | 物理设备                                                     |

## 568B 线序

白橙 橙 自绿 蓝 白蓝 绿 白棕 棕

## 数据链路层

- 学习
- 广播
- 转发
- 更新

VLAN（Virtual Local Area Network）的中文名为"虚拟局域网"，在计算机网络中，一个二层网络可以被划分为多个不同的广播域，一个广播域对应了一个特定的用户组，默认情况下这些不同的广播域是相互隔离的

VLAN作用：广播控制，增加安全性，提高带宽利用率，降低延迟

VLAN最多4096个：0～4095

TRUNK：跨交换机的同VLAN通信

以太网通道：多个物理接口绑定为一个虚拟接口，增加冗余性



## 网络层

实现两个端系统之间的数据透明传送，具体功能包括寻址和路由选择、连接的建立、保持和终止等

ICMP：控制报文协议。它是TCP/IP协议簇的一个子协议，用于在IP主机、路由器之间传递控制消息

路由器拒绝广播

### OSPF：动态路由协议，开放式最短路径优先



## 传输层

找到应用，端口共有65536个：0～65535

- TCP：传输控制协议，可靠，面向连接，传输效率低
- UDP：用户数据协议，不可靠，无连接，传输效率高

TCP信号：

- SYN 想与对方建立连接
- FIN 想与对方断开连接
- ACK 确认

### ACL 访问控制列表

- 标准访问控制列表：默认不写明的都拒绝，基于源IP地址过滤数据包，列表号1～99
- 扩展访问控制列表：基于源IP、目的IP、指定协议和端口过滤数据包，列表号100～199

反掩码中：

- 0 严格匹配
- 1 不匹配

### NAT 网络地址转换

使用本地地址的主机在和外界通信时，都要在NAT路由器上将其本地地址转换成全球IP地址，才能和因特网连接。

### PAT 端口地址转换 

PAT普遍应用于接入设备中，它可以将中小型的网络隐藏在一个合法的IP地址后面。PAT与动态地址NAT不同，它将内部连接映射到外部网络中的一个单独的IP地址上，同时在该地址上加上一个由NAT设备选定的TCP端口号。

### STP 生成树协议

逻辑上断开环路，防止广播风暴。当线路故障时，阻塞的接口自动激活，恢复通信，起备用线路的作用。

### HSRP 热备份路由器协议

实现HSRP的条件是系统中有多台路由器，它们组成一个“热备份组”，这个组形成一个虚拟路由器。HSRP是cisco平台一种特有的技术，是cisco的私有协议。



# 24 SHELL

/etc/shells 解释器文件

```shell
source	"脚本文件" # 使用用户默认解释器运行脚本
bash "脚本文件" # 使用指定解释器运行脚本，会新开解释器进程
```



## 环境变量

| PWD      | 当前工作目录                                            |
| -------- | ------------------------------------------------------- |
| PATH     | 指定命令的搜索路径                                      |
| USER     | 当前登录的用户名                                        |
| UID      | 当前登录的用户名的ID                                    |
| SHELL    | 当前用户用的是哪种shell                                 |
| HOME     | 用户的主工作目录                                        |
| HOSTNAME | 主机的名称                                              |
| PS1      | 第一级Shell命令提示符，root用户是#，普通用户是$         |
| PS2      | 第二级命令提示符，默认是“>”                             |
| RANDOM   | 随机数变量。每次引用这个变量会得到一个0~32767的随机数。 |



## 内部变量

| $#   | 命令行参数或位置参数的数量                         |
| ---- | -------------------------------------------------- |
| $?   | 最近一次执行的命令或shell脚本的出口状态，0代表成功 |
| $!   | 上一个命令的PID                                    |
| $$   | shell脚本的进程ID                                  |
| $*   | 所有的位置参数，有双引号时，一个整体               |
| $@   | 所有的位置参数，有双引号时，单独个体               |
| $0   | 脚本文件名                                         |
| $n   | $1表示第一个参数,$2表示第二个参数                  |

```shell
stty -echo # 关闭终端显示
stty echo # 打开终端显示
export a=20 # 设置全句变量

`命令` $(命令) # 两个都是执行命令，获取返回值
```



## 数值运算

整数运算的三种方式

```shell
expr $x + $y
expr $x % $y # 取模，求余

$[x+y]

$((x+y))
```

自增，自减

```shell
$[i++] $[i--] $[i+=2] $[i-=8] $[i/=2]
let i-=7;echo $i # ; 两边各自看作一行命令
```

小数运算

```shell
echo 'scale=4;12.34+5.678' | bc # scale=N 限制小数位的长度
```



## 条件测试

使用 "test 表达式" 或 [表达式]

| 符号      | 作用                                      |
| --------- | ----------------------------------------- |
| ==        | 比较两个字符串是否相等                    |
| !=        | 比较两个字符串是否不相等                  |
| A&&B      | A执行成功，就执行B                        |
| A\|\|B    | A执行失败，就执行B                        |
| A;B       | A执行完，就执行B                          |
| A&&B\|\|C | A执行成功，就执行B，A或B执行失败，就执行C |
| -z        | 检查变量的值是否未设置（空值）            |
| -n        | 测试变量是否不为空                        |
| -eq       | 比较两个整数是否相等                      |
| -ne       | 比较两个整数是否不相等                    |
| -gt       | 比较前面的整数是否大于后面的整数          |
| -ge       | 比较前面的整数是否大于或等于后面的整数    |
| -lt       | 比较前面的整数是否小于后面的整数          |
| -le       | 比较前面的整数是否小于或等于后面的整数    |
| -e        | 判断对象是否存在（不管是目录还是文件）    |
| -d        | 判断对象是否为目录（存在且是目录）        |
| -f        | 判断对象是否为文件（存在且是文件）        |
| -r        | 判断对象是否可读                          |
| -w        | 判断对象是否可写                          |
| -x        | 判断对象是否具有可执行权限                |



## for 循环

```shell
# 格式一
for 变量名 in 值列表
do
	命令
done

# 格式二
for((i=1;i<=10;i+=2))
do
	命令
done
```



## while 循环

```shell
while 条件 # 条件位置使用 : 或 true 表示无限循环 
do
	命令
done
```



## if 条件

```shell
# 单分支
if "条件";then
	命令
else
	命令
fi

# 多分支
if "条件";then
	命令
elif "条件";then
	命令
else
	命令
fi
```



## case 流程控制

```shell
case  变量  in
模式1)
	命令序列1 ;;
模式2)
	命令序列2 ;;
	.. ..
*)
	默认命令序列
esac
```



## 函数

```shell
# 格式一
function  函数名 {
	命令序列
	.. ..
}

# 格式二
函数名() {
	命令序列
	.. ..
}

#调用函数 函数名 位置参数1 位置参数2

#生成1到100的两种方式
{1..100}
seq 100
```



## 中断控制

```shell
break # 中断循环体
continue # 直接进入下一次循环
exit # 退出脚步，产生 $? 的值，默认为0
```



## 字符串处理

字符串截取

```shell
# 字符串处理的三种方式
${"变量名":"起始位置":"长度"} # 起始位置从0开始，可省略
expr substr "${变量名}" "起始位置" "长度" # 起始位置从1开始
echo ${变量名} | cut -b "起始位置"-"结束位置" # 起始位置从1开始
```

字符串替换

```shell
${"变量名"/old/new} # 只替换匹配的第一个
${"变量名"# old/new} # 替换匹配的全部
```

字符串掐头

```shell
${"变量名"#*"关键词"} # 从左向右，最短匹配，删除之前所有
${"变量名"##*"关键词"} # 从左向右，最长匹配，删除之前所有
```

字符串去尾

```shell
${"变量名"%"关键词"*} # 从右向左，最短匹配，删除之前所有
${"变量名"%%"关键词"*} # 从右向左，最长匹配，删除之前所有
```

字符串初值的处理（默认值）

```shell
${"变量名":-"值"} # 若变量没有值，就写入值
```



## 数组

```shell
# 格式一
数组名=(值1 值2 ... 值n) # 空格分离

# 格式二
数组名[下标]=值 # 下标从0开始

# 输出值
${数组名[下标]} # 单个
${数组名[@]} # 全部
${#数组名[@]} # 数组值个数
${数组名[@]:起始下标:个数} # 输出连续多个
```



# 25 expect 预期交互

安装 expect 包

```shell
# 格式
expect << EOF
spawn ssh 192.168.4.5                               # 创建交互式进程
expect "password:" { send "123456\r" }              # 自动发送密码
expect "#"          { send "touch /tmp.txt\r" }     # 发送命令
expect "#"          { send "exit\r" }
EOF

# 如果不希望ssh时出现yes/no的提示，远程时使用如下选项:
# ssh -o StrictHostKeyChecking=no root@IP
```



# 26 正则表达式

<center>基本列表</center>

| 符号      | 作用                               |
| --------- | ---------------------------------- |
| ^         | 匹配行首                           |
| $         | 匹配行尾                           |
| []        | 集合，匹配集合中任意单个字符       |
| [^]       | 对集合取反，在查找前对规则取反     |
| .         | 任意单个字符                       |
| *         | 匹配前一字符任意次数，不能单独使用 |
| \\{n,m\\} | 匹配前一字符n次到m次               |
| \\{n\\}   | 匹配前一字符n次                    |
| \\{n,\\}  | 匹配前一字符n次以上                |
| \\(\\)    | 保留                               |

<center>扩展列表</center>

| +     | 匹配前一字符至少一次 |
| ----- | -------------------- |
| ?     | 匹配前一字符至多一次 |
| {n,m} | 匹配前一字符n次到m次 |
| ()    | 保留,组合为整体      |
| \\b   | 单词边界             |

- grep 支持基本列表

- grep -E 或 egrep 支持扩展列表

  ```shell
  grep/egrep -i # 忽略大小写
  grep/egrep -v # 取反
  ```

  
# 27 sed文本处理工具

```shell
# 格式一
sed  [选项]  '条件指令'  文件

# 格式二
前置命令 | sed  [选项]  '条件指令'
```



| 选项 | 作用                                                         |
| ---- | ------------------------------------------------------------ |
| -n   | 屏蔽默认输出，默认sed会输出读取文档的全部内容                |
| -r   | 让sed支持扩展正则                                            |
| -i   | sed直接修改源文件，默认sed只是通过内存临时修改文件，源文件无影响 |



| 常见指令 | 作用 |
| -------- | ---- |
| p        | 显示 |
| d        | 删除 |
| s        | 替换 |

```shell
# p指令案例集锦
sed -n 'p' /etc/passwd # 打印全部
sed -n '3p' /etc/passwd # 打印第3行
sed  -n '3,6p' /etc/passwd # 输出3到6行
sed -n '3p;5p' /etc/passwd # 打印第3和5行
sed -n '3,+10p' /etc/passwd # 打印第3以及后面的10行
sed -n '1~2p' /etc/passwd # 打印奇数行
sed -n '/root/p' /etc/passwd # 打印包含root的行
sed -n '/bash$/p' /etc/passwd # 打印bash结尾的行

# d指令案例集锦
sed  '3,5d' a.txt             # 删除第3~5行
sed  '/xml/d' a.txt            # 删除所有包含xml的行
sed  '/xml/!d' a.txt         # 删除不包含xml的行，!符号表示取反
sed  '/^install/d' a.txt    # 删除以install开头的行
sed  '$d' a.txt                # 删除文件的最后一行
sed  '/^$/d' a.txt            # 删除所有空行

# s指令案例集锦
sed 's/xml/XML/'  a.txt        # 将每行中第一个xml替换为XML
sed 's/xml/XML/3' a.txt        # 将每行中的第3个xml替换为XML
sed 's/xml/XML/g' a.txt        # 将所有的xml都替换为XML
sed 's/xml# g'     a.txt       # 将所有的xml都删除（替换为空串）
sed 's#/bin/bash#/sbin/sh#' a.txt  # 将/bin/bash替换为/sbin/sh
sed '4,7s/^/#/'   a.txt        # 将第4~7行注释掉（行首加#号）
sed 's/^#an/an/'  a.txt        # 解除以#an开头的行的注释（去除行首的#号）
sed -r 's/^(.)(.*)(.)$/\3\2\1/' a.txt # 将文件中每行的第一个、倒数第1个字符互换
sed -r 's/([A-Z])/[\1]/g' nssw.txt # 为文件中每个大写字母添加括号
```



| 指令 | 作用                   |
| ---- | ---------------------- |
| i    | 在指定的行之前插入文本 |
| a    | 在指定的行之后追加文本 |
| c    | 替换指定的行           |
| r    | 读入文档，有-i才会保存 |
| w    | 保存到文档             |

```shell
sed  '2a XX'   a.txt            # 在第二行后一行，追加XX
sed  '2i XX'   a.txt            # 在第二行前一行，插入XX
sed  '2c XX'   a.txt            # 将第二行替换为XX
# 可以用\n换行
sed '2r m.txt' a.txt            # 在第2行下方插入m.txt
sed '1,4r m.txt' a.txt          # 1到4行保存到m.txt
```

  

复制、剪切

模式空间

- 存放当前处理的行，将处理结果输出
- 若当前行不符合处理条件，原样输出
- 处理完当前行再读入下一行来处理

保持空间

- 类似与“剪贴板”
- 默认存放一个换行符

| 复制 | H：模式空间--[追加]--保持空间<br>h：模式空间--[覆盖]--保持空间 |
| :--- | ------------------------------------------------------------ |
| 粘贴 | G：保持空间--[追加]--模式空间<br/>g：保持空间--[覆盖]--保持空间 |

```shell
sed '1h;1d;$G' a.txt # 把第1行剪切到文末
sed '1h;2,3H;$G' a.txt # 1～3行复制到文末
```



# 28 awk 数据处理引擎

格式：awk [选项] '[条件]{指令}' 文件

```shell
awk '{print $1,$3}' test.txt        # 打印文档第1列和第3列

# 选项 -F 可指定分隔符
# 输出passwd文件中以分号分隔的第1、7个字段
awk -F: '{print $1,$7}' /etc/passwd

# 以“:”或“/”分隔，输出第1、10个字段
awk -F [:/] '{print $1,$10}' /etc/passwd

awk -F: '{print $1,"的解释器:",$7}' /etc/passwd
root 的解释器: /bin/bash
bin 的解释器: /sbin/nologin
… …
```

<center>awk常用内置变量</content>

| $0     | 文本当前行的全部内容       |
| ------ | -------------------------- |
| $n(>0) | 文本的第n列                |
| NR     | 文件当前行的行号           |
| NF     | 文件当前行的列数（有几列） |



- BEGIN{ }		行前处理，读取文件内容前执行，指令执行1次 
- { }			逐行处理，读取文件过程中执行，指令执行n次 
- END{ }		行后处理，读取文件结束后执行，指令执行1次 

```shell
# 统计系统中使用bash作为登录Shell的用户总个数
awk 'BEGIN{x=0}/bash$/{x++} END{print x}' /etc/passwd
```

awk 处理条件：正则表达式、数值/字符串比较、逻辑比较、运算符

正则： ~ 代表匹配、!~ 代表不匹配		

```shell
awk -F: '/bash$/{print}' /etc/passwd # 输出其中以bash结尾的完整记录
awk -F: '/root/' /etc/passwd # 输出包含root的行数据
awk -F: '/^(root|adm)/{print $1,$3}' /etc/passwd # 输出root或adm账户的用户名和UID信息
awk -F: '$1~/root/' /etc/passwd # 输出账户名称包含root的基本信息
awk -F: '$7!~/nologin$/{print $1,$7}' /etc/passwd # 输出其中登录Shell不以nologin结尾（对第7个字段做!~反向匹配）的用户名、登录Shell信息
awk -F: 'NR==3{print}' /etc/passwd # 输出第3行（行号NR等于3）的用户记录
awk -F: '$3>=1000{print $1,$3}' /etc/passwd # 输出账户UID大于等于1000的账户名称和UID信息
awk -F: '$3<10{print $1,$3}' /etc/passwd # 输出账户UID小于10的账户名称和UID信息
awk -F: '$3>10 && $3<20' /etc/passwd # 输出账户UID大于10并且小于20的账户信息
awk -F: '$3>1000 || $3<10' /etc/passwd # 输出账户UID大于1000或者账户UID小于10的账户信息
```

​		

## awk流程控制

```shell
# 单分支
awk -F: '{if($7~/bash$/){i++}}END{print i}'  /etc/passwd # 统计/etc/passwd文件中登录Shell是“/bin/bash”的用户个数
 
# 双分支
awk -F: '{if($3<=1000){i++}else{j++}}END{print i,j}' /etc/passwd # 分别统计/etc/passwd文件中UID小于或等于1000、UID大于1000的用户个数
```



## awk数组

定义数组的格式：数组名[下标]=元素值

调用数组的格式：数组名[下标]，下标是任意字符串

遍历数组的用法：for(变量 in 数组名){print  数组名[变量]}。

```shell
 awk 'BEGIN{a[0]=0;a[1]=11;a[2]=22; for(i in a){print i,a[i]}}'
 awk  '{ip[$1]++} END{for(i in ip) {print i,ip[i]}}' /var/log/httpd/access_log | sort -nr
```



# 29 Nginx服务器

<center>web服务器软件</center>

| Unix 与 Linux | Apache、Nginx、Tenginx、Lighttpd、Tomcat、IBMWebSphere、Jboss |
| ------------- | ------------------------------------------------------------ |
| Windows       | IIS                                                          |

```shell
# 安装Nginx
yum -y install gcc pcre-devel openssl-devel      # 安装依赖包
useradd -s /sbin/nologin nginx 
./configure   \
> --prefix=/usr/local/nginx   \               # 指定安装路径
> --user=nginx   \                            # 指定用户
> --group=nginx  \                            # 指定组
> --with-http_ssl_module                      # 开启SSL加密功能

curl IP # 命令行浏览器

/usr/local/nginx/sbin/nginx                    # 启动服务
/usr/local/nginx/sbin/nginx -s stop            # 关闭服务
/usr/local/nginx/sbin/nginx -s reload          # 重新加载配置文件
/usr/local/nginx/sbin/nginx -V                 # 查看软件信息
ln -s /usr/local/nginx/sbin/nginx /sbin/       # 方便后期使用

netstat  -anptl # 查看全部端口 
```



## 升级Nginx服务器

```shell
./configure   \ # 将需要的功能源码放入objs/src内
> --prefix=/usr/local/nginx   \ 
> --user=nginx   \ 
> --group=nginx  \ 
> --with-http_ssl_module

make # 将objs/src内源码编译成程序，如objs/nginx

mv /usr/local/nginx/sbin/nginx /usr/local/nginx/sbin/nginxold # 备份旧程序
cp objs/nginx  /usr/local/nginx/sbin/         # 拷贝新版本
make upgrade                            # 升级
```



## 用户认证

```shell
vim  /usr/local/nginx/conf/nginx.conf # 打开配置文件
user nginx; # 进程所有者
worker_processes  1;  #  启动进程数量，与CPU一致
error_log  logs/error.log; # 错误日志文件
pid        logs/nginx.pid; # PID号文件

events {
    worker_connections  1024; # 单个进程最大并发量
}

http{
    server { # 定义虚拟主机 可基于IP 端口 域名
            listen       80;	# 监听端口，可以写成 IP：端口
            server_name  localhost; # 监听的域名
            auth_basic "Input Password:";                        # 认证提示符
            auth_basic_user_file "/usr/local/nginx/pass";        # 认证密码文件
            location / { # 匹配url，支持正则(前加~)，可以多个
                root   html; # 网站根目录
                index  index.html index.htm; # 默认主页左边优先级高
            }
     }
     server{
        listen 80;
        server_name www.xyz.com;
        location / { 
            root   www;                                
            index  index.html index.htm;
        }
     }
}

# 生成密码文件，创建用户及密码
yum -y install  httpd-tools
htpasswd -c /usr/local/nginx/pass   tom        # 创建密码文件
htpasswd  /usr/local/nginx/pass   jerry        # 追加用户，不使用-c选项

/usr/local/nginx/sbin/nginx -s reload   # 重新加载配置文件
```



## SSL虚拟主机

```shell
# 安装Nginx时必须使用--with-http_ssl_module参数，启用加密模块
vim  /usr/local/nginx/conf/nginx.conf
… …    
server {
        listen       443 ssl;
        server_name            www.c.com;
        ssl_certificate      cert.pem;         # 这里是证书文件 /usr/local/nginx/conf
        ssl_certificate_key  cert.key;         # 这里是私钥文件
        ssl_session_cache    shared:SSL:1m;
        ssl_session_timeout  5m;
        ssl_ciphers  HIGH:!aNULL:!MD5;
        ssl_prefer_server_ciphers  on;
        location / {
            root   html;
            index  index.html index.htm;
        }
    }
```



## 部署LNMP环境

```shell
yum -y install gcc openssl-devel pcre-devel # pcre-devel 正则依赖
yum -y install   mariadb   mariadb-server   mariadb-devel # 安装数据库
yum -y  install  php   php-mysql # php 为解释器 php-mysql 连接mysql插件
yum -y  install php-fpm-5.4.16-42.el7.x86_64.rpm # php-fpm php服务

# 查看php-fpm配置文件
vim /etc/php-fpm.d/www.conf
[www]
listen = 127.0.0.1:9000            # PHP端口号
pm.max_children = 32                # 最大进程数量
pm.start_servers = 15                # 最小进程数量
pm.min_spare_servers = 5            # 最少需要几个空闲着的进程
pm.max_spare_servers = 32           # 最多允许几个进程处于空闲状态

# 修改Nginx配置文件并启动服务
vim /usr/local/nginx/conf/nginx.conf
location / {
            root   html;
            index  index.php  index.html   index.htm;
#设置默认首页为index.php，当用户在浏览器地址栏中只写域名或IP，不说访问什么页面时，服务器会把默认首页index.php返回给用户
        }
 location  ~  \.php$  { # 正则匹配
            root           html;
            fastcgi_pass   127.0.0.1:9000;    #将请求转发给本机9000端口，PHP解释器
            fastcgi_index  index.php;
            #fastcgi_param   SCRIPT_FILENAME  $document_root$fastcgi_script_name;
            include        fastcgi.conf;
        }
```



## 地址重写

```shell
# 修改配置文件(访问a.html重定向到b.html)
 vim /usr/local/nginx/conf/nginx.conf
.. ..
server {
        listen       80;
        server_name  localhost;
		rewrite /a.html  /b.html (redirect); # 最后加redirect,url显示新地址
        location / {
            root   html;
        index  index.html index.htm;
        }
}

# 访问192.168.4.5的请求重定向至www.zz.cn
vim /usr/local/nginx/conf/nginx.conf
.. ..
server {
        listen       80;
        server_name  localhost;
        rewrite ^/ http:# www.zz.cn/;
        location / {
            root   html;
        	index  index.html index.htm;
        }
}

# 访问192.168.4.5/下面子页面，重定向至www.zz.cn/下相同的页面
vim /usr/local/nginx/conf/nginx.conf
.. ..
server {
        listen       80;
        server_name  localhost;
        rewrite ^/(.*)$ http:# www.zz.cn/$1;
        location / {
            root   html;
        	index  index.html index.htm;
        }
        # 这里，~符号代表正则匹配，*符号代表不区分大小写
        if ($http_user_agent ~* firefox) {            # 识别客户端firefox浏览器
        	rewrite ^(.*)$ /firefox/$1;
        }
}
```

地址重写格式 rewrite 旧地址  新地址 [选项];

- last        不再读其他rewrite，只在同location有用，
- break       不再读其他语句，结束判断
- redirect    临时重定向
- permament   永久重定向



## Nginx反向代理

Nginx采用轮询的方式调用后端Web服务器

```shell
vim /usr/local/nginx/conf/nginx.conf
.. ..
http {
.. ..
# 使用upstream定义后端服务器集群，集群名称任意(如webserver)
# 使用server定义集群中的具体服务器和端口
upstream webserver {
				# 通过ip_hash设置调度规则为：相同客户端访问相同服务器
                ip_hash;
                server 192.168.2.100 weight=1 max_fails=1 fail_timeout=30;
                server 192.168.2.200 weight=2 max_fails=2 fail_timeout=30;
                server 192.168.2.101 down;
                # weight设置服务器权重值，默认值为1
                # max_fails设置最大失败次数
                # fail_timeout设置失败超时时间，单位为秒
                # down标记服务器已关机，不参与集群调度
        }
.. ..
server {
        listen        80;
        server_name  localhost;
        location / {
			# 通过proxy_pass将用户的请求转发给webserver集群
            proxy_pass http:# webserver;
        }
}
```

### Nginx的TCP/UDP调度器

```shell
yum –y install gcc pcre-devel openssl-devel        # 安装依赖包
tar  -xf   nginx-1.12.2.tar.gz
cd  nginx-1.12.2
./configure   \
> --with-http_ssl_module   \                             # 开启SSL加密功能
> --with-stream                                       # 开启4层反向代理功能
make && make install           # 编译并安装

vim /usr/local/nginx/conf/nginx.conf
stream {
            upstream backend {
               server 192.168.2.100:22;            # 后端SSH服务器的IP和端口
               server 192.168.2.200:22;
			}
            server {
                listen 12345;                    # Nginx监听的端口
                proxy_connect_timeout 10s;
                proxy_timeout 30s;
                 proxy_pass backend;
            }
}
http {
.. ..
}
```



## Nginx常见问题处理

### 自定义报错页面

```shell
vim /usr/local/nginx/conf/nginx.conf
.. ..
error_page   404  /40x.html;    # 自定义错误页面
.. ..
```



### 查看服务器状态信息

```shell
./configure   \
> --with-http_ssl_module                        # 开启SSL加密功能
> --with-stream                                 # 开启TCP/UDP代理模块
> --with-http_stub_status_module                # 开启status状态页面

# 修改Nginx配置文件，定义状态页面
vim /usr/local/nginx/conf/nginx.conf
… …
location /status {
                stub_status on;
                 #allow IP地址;
                 #deny IP地址;
        }
… …

# 查看状态页面信息
curl  http:# 192.168.4.5/status
Active connections: 1 
server accepts handled requests
 10 10 3 
Reading: 0 Writing: 1 Waiting: 0
# Active connections：当前活动的连接数量。
# Accepts：已经接受客户端的连接总数量。
# Handled：已经处理客户端的连接总数量。（一般与accepts一致，除非服务器限制了连接数量）。
# Requests：客户端发送的请求数量。
# Reading：当前服务器正在读取客户端请求头的数量。
# Writing：当前服务器正在写响应信息的数量。
# Waiting：当前多少客户端在等待服务器的响应。
```



### 优化Nginx并发量

```shell
# 优化Nginx并发量
vim /usr/local/nginx/conf/nginx.conf
.. ..
worker_processes  2;                    # 与CPU核心数量一致
events {
	worker_connections 65535;        # 每个worker最大并发连接数
use epoll;
}
.. ..

# 优化Linux内核参数（最大文件数量）
ulimit -a                        # 查看所有属性值
ulimit -Hn 100000                # 设置硬限制（临时规则）
ulimit -Sn 100000                # 设置软限制（临时规则）
vim /etc/security/limits.conf # 手动写入
    .. ..
*               soft    nofile            100000
*               hard    nofile            100000
#该配置文件分4列，分别如下：
#用户或组    硬限制或软限制    需要限制的项目   限制的值
```



### 优化Nginx数据包头缓存

```shell
vim /usr/local/nginx/conf/nginx.conf
.. ..
http {
    client_header_buffer_size    1k;        # 默认请求包头信息的缓存    
    large_client_header_buffers  4 4k;      # 大请求包头部信息的缓存个数与容量 一个请求16k
    .. ..
}
```



### 浏览器本地缓存静态数据

```shell
vim /usr/local/nginx/conf/nginx.conf
server {
        listen       80;
        server_name  localhost;
        location / {
            root   html;
            index  index.html index.htm;
        }
        location ~* \.(jpg|jpeg|gif|png|css|js|ico|xml)$ {
            expires        30d;            # 定义客户端缓存时间为30天
        }
}
```



### 日志切割

```shell
# 每周5的03点03分自动执行脚本完成日志切割工作
vim /usr/local/nginx/logbak.sh
#!/bin/bash
date=`date +%Y%m%d`
logpath=/usr/local/nginx/logs
mv $logpath/access.log $logpath/access-$date.log
mv $logpath/error.log $logpath/error-$date.log
kill -USR1 $(cat $logpath/nginx.pid)

crontab -e
03 03 * * 5  /usr/local/nginx/logbak.sh
```

<center>KILL 信号</center>

| 字符型号 | 对应数字 | 作用                             |
| -------- | -------- | -------------------------------- |
| HUP      | 1        | 终端断线，重新加载配置，平滑升级 |
| INT      | 2        | 中断（同Ctrl+c）                 |
| QUIT     | 3        | 退出（同Ctrl+\）                 |
| TERM     | 15       | 终止（无信号选项时的默认值）     |
| KILL     | 9        | 强制终止                         |
| CONT     | 18       | 继续（与STOP相反，fg/bg命令）    |
| STOP     | 19       | 暂停（同Ctrl+z）                 |

以上除了KILL信号，程序都可以忽略。



### 对页面进行压缩处理

```shell
vim /usr/local/nginx/conf/nginx.conf
http {
.. ..
gzip on;                            # 开启压缩
gzip_min_length 1000;               # 小文件不压缩
gzip_comp_level 4;                  # 压缩比率
gzip_types text/plain text/css application/json application/x-javascript text/xml application/xml application/xml+rss text/javascript;
                                    # 对特定文件压缩，类型参考mime.types
.. ..
}
```



### 服务器内存缓存

```shell
http { 
    open_file_cache          max=2000  inactive=20s;
            open_file_cache_valid    60s;
            open_file_cache_min_uses 5;
            open_file_cache_errors   off;
    # 设置服务器最大缓存2000个文件句柄，关闭20秒内无请求的文件句柄
    # 文件句柄的有效时间是60秒，60秒后过期
    # 只有访问次数超过5次会被缓存
    # 缓存报错不报警
} 
```



# 30 memcached服务

memcached是高性能的分布式缓存服务器，用来集中缓存数据库查询结果，减少数据库访问次数，以提高动态Web应用的响应速度。

## 构建memcached服务

```shell
yum -y  install   memcached

vim /usr/lib/systemd/system/memcached.service
ExecStart=/usr/bin/memcached -u $USER -p $PORT -m $CACHESIZE -c $MAXCONN $OPTIONS

vim /etc/sysconfig/memcached
PORT="11211"
USER="memcached"
MAXCONN="1024"
CACHESIZE="64"
OPTIONS=""
```

验证时需要客户端主机安装telnet，远程memcached来验证服务器的功能：

```shell
telnet  192.168.4.5  11211
Trying 192.168.4.5...
……
## 提示：0表示不压缩，180为数据缓存时间，3为需要存储的数据字节数量。
set name 0 180 3                # 定义变量，变量名称为name
plj                            # 输入变量的值，值为plj                
STORED
get name                        # 获取变量的值
VALUE name 0 3                 # 输出结果
plj
END
##提示：0表示不压缩，180为数据缓存时间，3为需要存储的数据字节数量。
add myname 0 180 10            # 新建，myname不存在则添加，存在则报错
set myname 0 180 10            # 添加或替换变量
replace myname 0 180 10        # 替换，如果myname不存在则报错
get myname                    # 读取变量
append myname 0 180 10        # 向变量中追加数据
delete myname                    # 删除变量
stats                        # 查看状态
flush_all                        # 清空所有
quit                            # 退出登录         
```


## LNMP+memcached

使用PHP来操作memcached，注意必须要为PHP安装memcache扩展（php-pecl-memcache），否则PHP无法解析连接memcached的指令。



## PHP Session 共享

```shell
/var/lib/php/session/            # 查看服务器本地的Session信息

vim  /etc/php-fpm.d/www.conf            # 修改该配置文件的两个参数
# 文件的最后2行
修改前效果如下:
php_value[session.save_handler] = files
php_value[session.save_path] = /var/lib/php/session
# 原始文件，默认定义Sessoin会话信息本地计算机（默认在/var/lib/php/session）
修改后效果如下:
php_value[session.save_handler] = memcache
php_value[session.save_path] = "tcp:# 192.168.2.5:11211"
# 定义Session信息存储在公共的memcached服务器上，主机参数中为memcache（没有d）
# 通过path参数定义公共的memcached服务器在哪（服务器的IP和端口）
```



## PHP -FPM 错误日志文件

```
/var/log/php-fpm/www-error.log
```



# 31 Tomcat

| 软件环境   | openjdk                           |
| ---------- | --------------------------------- |
| 软件包     | tomcat                            |
| 端口       | 8080                              |
| 主配置文件 | /usr/local/tomcat/conf/server.xml |
| Java SE    | 标准版                            |
| Java EE    | 企业版                            |
| Java ME    | 移动版                            |

部署Tomcat服务器软件

```shell
yum -y install  java-1.8.0-openjdk                # 安装JDK
yum -y install java-1.8.0-openjdk-headless        # 安装JDK
java -version                                    # 查看JAVA版本

ls /usr/local/tomcat
bin/                                            # 主程序目录
lib/                                            # 库文件目录
logs/                                          # 日志目录  
temp/                                         # 临时目录
work/                                        # 自动编译目录jsp代码转换servlet
conf/                                        # 配置文件目录
webapps/                                        # 页面目录

# 启动服务
/usr/local/tomcat/bin/startup.sh

netstat -nutlp |grep java        # 查看java监听的端口
tcp        0      0 :::8080              :::*                LISTEN      2778/java           
tcp        0      0 ::ffff:127.0.0.1:8005     :::*         LISTEN       2778/java  
```



## 虚拟主机

```shell
cat /usr/local/tomcat/conf/server.xml
<Server>
   <Service>
     <Connector port=8080 /> # 监听端口 虚拟主机通用
     <Connector port=8009 />
     <Engine name="Catalina" defaultHost="localhost"> # defaultHost 定义访问ip时 默认打开的Host
    <Host name="www.a.com" appBase="webapps/a" unpackWARS="true" autoDeploy="true"> # 每个Host是一个虚拟主机
    	<Context path="" docBase="base"/>
    	<Context path="/test" docBase="/var/www/html/" /> # 当用户访问http:# www.a.com/test打开/var/www/html目录下的页面
    </Host>
    <Host name="www.b.com" appBase="webapps/b" unpackWARS="true" autoDeploy="true">
</Host>
… …

# appBase 定义基础目录，默认项目文件夹是ROOT
# docBase 定义首页目录，默认项目文件夹是ROOT
# path 匹配url
```



## SSL 加密站点

```shell
vim /usr/local/tomcat/conf/server.xml
… …
<Connector port="8443" protocol="org.apache.coyote.http11.Http11NioProtocol"
maxThreads="150" SSLEnabled="true" scheme="https" secure="true"
keystoreFile="/usr/local/tomcat/keystore" keystorePass="123456" clientAuth="false" sslProtocol="TLS" />
# 端口 8443
# 备注，默认这段Connector被注释掉了，打开注释，添加密钥信息即可
```



## 配置Tomcat日志

```shell
 vim /usr/local/tomcat/conf/server.xml
.. ..
<Host name="www.a.com" appBase="a" unpackWARS="true" autoDeploy="true">
<Context path="/test" docBase="/var/www/html/" />
# 从默认localhost虚拟主机中把Valve这段复制过来，适当修改下即可
<Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
               prefix=" a_access" suffix=".txt" # prefix 文件名 suffix 扩展名
               pattern="%h %l %u %t &quot;%r&quot; %s %b" />
</Host>
<Host name="www.b.com" appBase="b" unpackWARS="true" autoDeploy="true">
<Context path="" docBase="base" />
<Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
               prefix=" b_access" suffix=".txt"
               pattern="%h %l %u %t &quot;%r&quot; %s %b" />
</Host>
.. ..
```



# 32 Varnish缓存服务器

- 使用Varnish加速后端Web服务 
- 代理服务器可以将远程的Web服务器页面缓存在本地 
- 远程Web服务器对客户端用户是透明的 
- 利用缓存机制提高网站的响应速度 

## 部署Varnish缓存服务器

```shell
yum -y install gcc readline-devel    # 安装软件依赖包
yum -y install ncurses-devel         # 安装软件依赖包
yum -y install pcre-devel            # 安装软件依赖包
yum -y install python-docutils-0.11-0.2.20130715svn7687.el7.noarch.rpm # 安装软件依赖包
useradd -s /sbin/nologin varnish                # 创建账户
tar -xf varnish-5.2.1.tar.gz
cd varnish-5.2.1
./configure
make && make install

# 复制启动脚本及配置文件
cp  etc/example.vcl   /usr/local/etc/default.vcl # 可以复制到任意地点

# 修改代理配置文件
vim  /usr/local/etc/default.vcl
backend default {
     .host = "192.168.2.100";
     .port = "80";
 }
 
# 启动服务
varnishd  -f /usr/local/etc/default.vcl
# varnishd命令的其他选项说明如下：
# varnishd –s malloc,128M        定义varnish使用内存作为缓存，空间为128M
# varnishd –s file,/var/lib/varnish_storage.bin,1G 定义varnish使用文件作为缓存

# 查看varnish日志
varnishlog                        # varnish日志
varnishncsa                    # 访问日志

# 更新缓存数据，在后台web服务器更新页面内容后，用户访问代理服务器看到的还是之前的数据，说明缓存中的数据过期了需要更新（默认也会自动更新，但非实时更新）。
varnishadm  
varnish> ban req.url ~ .*
# 清空缓存数据，支持正则表达式
```

