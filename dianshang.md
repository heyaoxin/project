# Busykid电商平台搭建实施方案

环境准备：4台公有云ECS主机

底层架构：

两台ECS使用LNMP环境充当后端web服务器

两台ECS充当前端，使用nginx反向代理到后端两台web服务器，消除单点故障，提高并发能力，并且采用keepalive技术，保证网络的高可用

[gallery ids="43"]

**实施步骤：**

1.先在公有云上申请HAvip用于绑定两台keepalive作为对外的一个VIP

2.把四台ECS加入到相同安全组，使四台ECS内网互通，注意做完后先测试四台机器连通性

操作参考： 

https://help.aliyun.com/document_detail/25475.html?spm=5176.doc25385.2.1.dRRsk9#section-i35-nll-ngb 

3.部署后端ECS

ECS1：192.168.126.134

ECS2：192.168.126.135

## 搭建lnmp

1.安装mysql

```
yum -y install mariadb-server mariadb #数据库安装
```

数据库启动开机启动

```
systemctl start mariadb
systemctl status mariadb
systemctl enable mariadb
```

数据库初始化

```
mysql_secure_installation
```

2.安装nginx，设置开机启动

```
yum -y install nginx
systemctl start nginx
systemctl enable nginx
```

3.安装php7.2

\#更改源

```
yum install epel-release -y

rpm -Uvh https://mirror.webtatic.com/yum/el7/webtatic-release.rpm
```

\#清除历史版本

```
yum -y remove php*
```

\#安装拓展包

```
yum -y install php72w php72w-cli php72w-fpm php72w-common php72w-devel php27w-mysqlnd
```

\#启动服务

```
systemctl enable php-fpm.service

systemctl start php-fpm.service
```

4.配置nginx

```
cat /etc/nginx/conf.d/taobao.conf
server{
listen 8090;
root /data/taobao;
index index.php index.html index.hml;

tcp_nodelay on;
tcp_nopush on;
sendfile on;
access_log /var/log/nginx/proxy_access.log main;#定义日志
error_log /var/log/nginx/proxy_error.log;  #定义错误日志

#长连接

keepalive_timeout 10s;

keepalive_requests 100;


location /{
try_files $uri $uri/ /index.php?q=$uri&$args;
}
location ~ .php{
fastcgi_pass 127.0.0.1:9000;
fastcgi_index index.php;
fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
include fastcgi_params;
}
}
```

5.上传网站

```
rz [网站压缩包]

#解压 unzip 【】
```

6.修改php-fpm配置

```
vim /etc/php-fpm.d/www.conf

user=nginx

group=nginx
```

7.测试网站是否可行

访问自己IP（公网）

8.两台ECS同样配置即可



**搭建nginx反向代理**

ECS3:192.168.126.132

ECS4:192.168.126.133

ECS3：

1.下载nginx

```
yum -y install nginx
systemctl start nginx
systemctl enable nginx
```

2.配置反向代理和缓存

```
cat /etc/nginx/conf.d/proxy.conf 
upstream websers{
server 192.168.126.134:8090;  #ECS配置一样，使用RR
server 192.168.126.135:8090; 
}
server{
listen 8090;

tcp_nodelay on;
tcp_nopush on;
sendfile on;

#日志分离
access_log /var/log/nginx/proxy_access.log main;#定义登录日志
error_log /var/log/nginx/proxy_error.log;  #定义错误日志


location /{
proxy_pass http://websers; #根据location处理后转发

##缓存设置

proxy_set_header X‐Real‐IP $remote_addr;
proxy_send_timeout 75s; # 默认60s 
proxy_read_timeout 75s; # 默认60s 
proxy_connect_timeout 75; # 默认60s
proxy_cache pxycache; 
proxy_cache_key $request_uri; 
proxy_cache_valid 200 302 301 1h; 
proxy_cache_valid any 1m;

}

}

在主配置文件/etc/nginx/nginx.conf下配置缓存存放目录

http{

........

proxy_cache_path /var/cache/nginx/proxy_cache levels=1:1:1 keys_zone=pxycache:20m max_size=1g;

｝
```

重新加载配置文件

```
nginx -s reload
```

3.测试：通过访问ECS3能访问到两个后端，并且检查代理的缓存，日志是否都正常

ECS4做同样配置即可



**搭建keepalive**

在ECS3,ECS4上做keepalive，因为这两台机器绑定了havip,公网上是通过访问这个havip去访问我们前端的（也就是说着havip就是我们vrrp的VIP）

ECS3：

```
cat /etc/keepalived/keepalived.conf
! Configuration File for keepalived
global_defs {
router_id node0      # 对端backup需要修改
}
vrrp_instance VI_1 {
state MASTER       # node0节点BACKUP
interface ens35
virtual_router_id 10
priority 100             # node1节点小于100,因为node1是backup
advert_int 1
authentication {
auth_type PASS
auth_pass 123456
}
virtual_ipaddress {
x.x.x.x                    #VIP地址（我们申请的HAvip）
}
}
```

ECS4:

```
cat /etc/keepalived/keepalived.conf

! Configuration File for keepalived
global_defs {
router_id node1 
}
vrrp_instance VI_1 {
state BACKUP 
interface ens36
virtual_router_id 10
priority 98           # node1节点小于100,因为node1是backup
advert_int 1
authentication {
auth_type PASS
auth_pass 123456
}
virtual_ipaddress {
x.x.x.x     #vip
}
}
```

\##测试keepalive：

先测试通过访问VIP是否可以访问到后端集群

在把master的keepalive关闭模拟故障测试VIP是否可以漂移到backup

没有问题继续往下：



配置检测nginx负载均衡的健康状况：

```
cat /etc/keepalived/keepalived.conf
! Configuration File for keepalived
global_defs {
router_id node0 # node1修改
}

# 定义script,通过执行脚本检测NGINX负载均衡代理的健康情况，若代理故障就降低自己优先级master降为backup
vrrp_script chk_http_port {
script "/usr/local/src/check_nginx_pid.sh" # 脚本要拥有可执行权限
interval 1 # 设置脚本执行频率 
weight -5 # 优先级-5 ； 注意优先级相减之后 保证一定要小于我们的backup；并且一定要开启抢占模式
}

vrrp_instance VI_1 {
state MASTER # node1节点BACKUP
interface ens35
virtual_router_id 10
priority 100 # node1节点小于100,因为node1是backup
advert_int 1
authentication {
auth_type PASS
auth_pass 123456
}

#调用脚本
track_script {
chk_http_port 
}

virtual_ipaddress {
192.168.31.99
}
}
```

脚本：

```
cat /usr/local/src/check_nginx_pid.sh 
#!/bin/bash
A=`ps -C nginx --no-header |wc -l` #统计nginx进程数 
if [ $A -eq 0 ];then 
/usr/local/nginx/sbin/nginx # 开启nginx服务；会尝试着自己启动，在测试时先注释掉这一行 
if [ `ps -C nginx --no-header |wc -l` -eq 0 ];then 
exit 1 # 如果进程数量为0 非正常退出
else
exit 0 # 如通进程数量不为0 正常退出
fi
else
exit 0
fi
```

测试：关闭master的nginx模拟故障，看master是否可以切换到backup身上

没有问题继续：

**配置数据库主主备份**

在后端ecs1,ecs2分别执行脚本

```
#!/bin/bash
Ip_addr="192.168.126.135" # 修改为对端的node地址
User_pwd="123456"
systemctl stop firewalld
setenforce 0
#yum install mariadb-server -y  #已安装就不需要了
sed -i '/^\[mysqld\]$/a\binlog-ignore = information_schema' /etc/my.cnf.d/server.cnf
sed -i '/^\[mysqld\]$/a\binlog-ignore = mysql' /etc/my.cnf.d/server.cnf
sed -i '/^\[mysqld\]$/a\skip-name-resolve' /etc/my.cnf.d/server.cnf
sed -i '/^\[mysqld\]$/a\auto-increment-increment = 1' /etc/my.cnf.d/server.cnf # 注意node4节点上必须不同
sed -i '/^\[mysqld\]$/a\log-bin = mysql-bin' /etc/my.cnf.d/server.cnf
sed -i '/^\[mysqld\]$/a\auto_increment_offset = 1' /etc/my.cnf.d/server.cnf # 注意node4节点上必须不同
sed -i '/^\[mysqld\]$/a\server-id = 1' /etc/my.cnf.d/server.cnf # 注意node4节点上必须不同
systemctl restart mariadb
mysql -uroot -e "grant replication slave on *.* to 'repuser'@'$Ip_addr' identified by '$User_pwd';"
```

查询ECS1节点master状态：
     MariaDB [(none)]> show master status;

查询ECS2节点master状态
     MariaDB [(none)]> show master status;

在ECS1节点执行连接命令：
     MariaDB [(none)]> change master to master_host='192.168.126.135',master_port=3306,master_user='repuser',master_password='000000',master_log_file='mysql-bin.000003',master_log_pos=407;
     MariaDB [mysql]> start slave;
     在ECS2节点执行连接命令：
     MariaDB [(none)]> change master to master_host='192.168.126.134',master_port=3306,master_user='repuser',master_password='000000',master_log_file='mysql-bin.000003',master_log_pos=402;
     MariaDB [mysql]> start slave;

```
查看从节点状态： show slave status \G;  观察IO和SQL线程是否为YES
```

测试：分别在ECS1/ECS2上创建数据库数据表看是否可以同步，能同步就没问题，继续

现在既然每个数据库都是数据一致的，我们不想老是登录到某一台的数据库进行管理，我们可以怎么做呢？

**nginx代理双主数据库**

那么我们可以用nginx代理数据库的tcp连接，使用VIP去登录数据库进行管理，完成对双主数据库的负载均衡（添加下方黑体字配置即可）

```
cat /etc/nginx/nginx.conf | grep -Ev "^#|^$"
.......
error_page 500 502 503 504 /50x.html;
location = /50x.html {
}
}
}


stream {
upstream cloudsocket {
hash $remote_addr consistent;
# $binary_remote_addr;
server 192.168.126.134:3306 weight=2 max_fails=3 fail_timeout=30s;
server 192.168.126.135:3306 weight=1 max_fails=3 fail_timeout=30s;
}
server {
listen 3306;#数据库服务器监听端口
proxy_connect_timeout 10s;
#proxy_timeout 300s;#设置客户端和代理服务之间的超时时间，如果5分钟内没操作将自动断开。
proxy_pass cloudsocket;
}
}
```

当然，有条件的时候可以再分一组VRRP组专门去代理mysql，此时ECS1为backup，ECS2为master，更好地负载均衡，不用全部都走ECS1，但是没有更多的HAvip就算了



至此，完整的架构已经搭建完毕了！！！！！



**优化：**

1.定时任务定期清理日志( 只保留七天内的日志: )

 \* * * * * find /var/log/nginx/proxy_access.log -atime 7 -delete | -exec rm -rf {}\; 

2.一台机器可以专门放置一些静态资源，然后挂载到各台服务器，方便统一管理所有服务器的静态资源，对电商的上架下架商品更友好，这样子也可以防止其他人对服务器的误操作

3.搭建ftp服务，挂载载网站目录，可以本地编辑网站后上传，不需要登录到服务器，不必要被别人登录

4.搭建zabbix监控工具，实时监测服务器健康状况，若有故障短信通知管理员

zabbix搭建参考： https://www.yuque.com/docs/share/47a6c107-653b-489a-9560-08d795374234?# 

5.在zabbix-server搭建短信服务器，配合zabbix监测，达到故障发送短信的目的

