### 1.服务器系统初始化配置

```shell
#!/bin/bash

# ban root remote
sed -i 's/#PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config

# ssh connect timeout!
if ! grep "TMOUT=300" /etc/profile &>/dev/null; then
echo "export TMOUT=300" >> /etc/profile
fi

# ban selinux
sed -i '/SELINUX/{s/permissive/disabled/}' /etc/selinux/config

#shutdown firewalld for Centos7
if egrep "7.[0-9]" /etc/redhat-release &>/dev/null; then    
systemctl stop firewalld    systemctl disable firewalld
elif egrep "6.[0-9]" /etc/redhat-release &>/dev/null; then    
service iptables stop    
chkconfig iptables off
fi

#set time zone, config time synchronization
ln -s /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
if ! crontab -l |grep ntpdate &>/dev/null ; then    
(echo "* 1 * * * ntpdate Not Found >/dev/null 2>&1";crontab -l) |crontab
fi

#系统内核优化
cat >> /etc/sysctl.conf << EOF
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_tw_buckets = 20480
net.ipv4.tcp_max_syn_backlog = 20480
net.core.netdev_max_backlog = 262144
net.ipv4.tcp_fin_timeout = 20
EOF

# 减少SWAP使用
echo "0" > /proc/sys/vm/swappiness
```

### 2.定时清理文件内容，并记录大小

```shell
#!/bin/bash

#####################################################################
#  Execute the script every hour
#	 Clear the contents of all files in the target directory
#  Take the tmp directory file as an example
#####################################################################
logfile=/tmp/`date +%H-%F`.log
n=`date +%H`
if [ $n -eq 00 ] || [ $n -eq 12 ]
then
# used for as an example 
for i in `find /data/log/ -type f`
do
true > $i
done
else
for i in `find /data/log/ -type f`
do
du -sh $i >> $logfile
done
fi
```

### 3.mysql

#### 3.1 mysql数据库备份

```shell
#!/bin/bash

DATE=$(date +%F_%H-%M-%S)
# your used ip,mysql db name ,username and password
HOST=1.1.1.1
DB=mysql
USER=backup
PASS=123456
# email 
MAIL="ftp@example.com username@example.com"
BACKUP_DIR=/data/db_backup
SQL_FILE=${DB}_full_$DATE.sql
BAK_FILE=${DB}_full_$DATE.zip
cd $BACKUP_DIR
if mysqldump -h$HOST -u$USER -p$PASS --single-transaction --routines --
triggers -B $DB > $SQL_FILE; then
zip $BAK_FILE $SQL_FILE && rm -f $SQL_FILE
if [ ! -s $BAK_FILE ]; then
echo "$DATE 内容" | mail -s "主题" $MAIL
fi
else
echo "$DATE 内容" | mail -s "主题" $MAIL
fi
find $BACKUP_DIR -name '*.zip' -ctime +14 -exec rm {} \;
```

#### 3.2 检测mysql服务

```shell
#!/bin/bash
host=MySQL ip
user=MySQL username
passwd=MySQL password
mysqladmin -h$host -u$user -p$passwd ping &> /dev/null
if [ $? -eq 0 ];then
echo "MySQL is UP"
else
echo "MySQL is down"
fi
```

### 3.3 检测mysql连接数

```shell
#!/bin/bash
#每 3 秒检测一次 MySQL 并发连接数，可以将本脚本设置为开机自启，或指定时间执行
#满足对 MySQL 数据库的监控需求，查看 MySQL 连接是否正常
log_file=/data/mysql/log/mysql_count.log
user=mysql username
passwd=mysql password
while :
do
sleep 2
 count=`mysqladmin -u"$user" -p"$passwd" status | awk '{print $4}'`
 echo "`date +%Y-%m-%d` 并发连接数为：$count" >> $log_file
done
```

### 4.屏蔽访问恶意网站ip

#### 4.1 用TCP连接

```shell
#!/bin/bash
ABNORMAL_IP=$(netstat -an |awk '$4~/:80$/ && $6~/ESTABLISHED/{gsub(/:[0-9]+/,"",$5);{a[$5]++}}END{for(i in a)if(a[i]>100)print i}')
#gsub 是将第五列（客户端 IP）的冒号和端口去掉
for IP in $ABNORMAL_IP; do
if [ $(iptables -vnL |grep -c "$IP") -eq 0 ]; then
iptables -I INPUT -s $IP -j DROP
 fi
done
```

#### 4.2 使用nginx日志

```shell
#!/bin/bash
DATE=$(date +%d/%b/%Y:%H:%M)
ABNORMAL_IP=$(tail -
n5000 access.log |grep $DATE |awk '{a[$1]++}END{for(i in a)if(a[i]>100)print i}')
#使用tail 防止文件过大，读取慢，数字可调整每分钟最大的访问量。awk 不能直接过滤日志，因为包含特殊字符。
for IP in $ABNORMAL_IP; do
if [ $(iptables -vnL |grep -c "$IP") -eq 0 ]; then
iptables -I INPUT -s $IP -j DROP
 fi
done
```

### 5.管理计算机账户

#### 5.1 查看当前计算机所有账户用户名

```shell
#!/bin/bash
echo "方法,指定以:为分隔符，打印/etc/passwd 文件的第 1 列"
awk -F: '{print $1}' /etc/passwd
echo "方法,指定以:为分隔符，打印/etc/passwd 文件的第 1 列"
cut -d: -f1 /etc/passwd
echo "方法使用 sed 的替换功能,将/etc/passwd 文件中第 1 个:后面的所有内容替换为空
(仅显示用户名)"
sed "s/:.*//" /etc/passwd
```

#### 5.2 统计linux系统可以登录计算机的账户数量

```shell
#!/bin/bash
#方法grep：
grep "bash$" /etc/passwd | wc -l 
#方法awk：
awk -F: '/bash$/{x++} END{print x}' /etc/passwd
#更为准确的方法
for shell in `awk -F: '{print $7}' /etc/passwd` 
do
if [[ "`cat /etc/shells`" =~ "$shell" ]]; then 
 fi
done
echo $n
```

