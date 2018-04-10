title: Gitlab 自动备份
toc: true
tags: Git
date: 2018/3/20 19:45:00
update: 2018/3/21 22:16:00

---




## 一、准备工作
### 1.目的
    对代码仓库进行容灾备份
### 2.步骤
* 因为需要使用scp方式进行文件传输，需要对代码仓库服务器(以下简称:**A服务器**)和备份服务器(以下简称:**B服务器**)进行密钥配对，以取消scp传输所需要的密码限制
* A服务器每天定时进行本地备份并通过scp方式将备份发送到B服务器
* B服务器定期清除多余备份，只保留最近10天的备份
* /etc/gitlab/gitlab.rb 配置文件手动备份即可(邮件，端口等配置)

## 二、开始
### 1.密钥配对
#### 1.1.创建并发送密钥
    在A服务器上生成rsa密钥对：
```
ssh-keygen -t rsa
```
    在/root/.ssh下生成私钥id_rsa 和公钥id_rsa.pub 两个文件
    然后在/root/.ssh下复制备份一份id_rsa.pub 命名为 id_rsa.pub.A，以便拷贝到B服务器:
```
cp id_rsa.pub id_rsa.pub.A
```
    使用scp命令进行远程复制，将A机生成的id_rsa.pub.A拷贝到远程服务器B的/root/.ssh目录下:
```
root@ubuntu4146:~/.ssh# scp /root/.ssh/id_rsa.pub.A root@远程服务器ip:/root/.ssh/
```

#### 1.2.导入密钥
    在 B 的/root/.ssh下创建authorized_keys文件，使用如下命令：
```
touch authorized_keys
```
    通过 cat 命令 把id_rsa.pub.A 追写到 authorized_keys 文件中，命令如下：
```
cat id_rsa.pub.A >> authorized_keys
```
    执行如下命令，修改authorized_keys文件的权限：
```
chmod 400 authorized_keys
```
> authorized_keys文件的权限很重要，如果设置为777，那么登录的时候，还是需要提供密码的。

### 2.创建远程备份脚本
    A服务器上创建定期备份脚本auto_backup_to_remote.sh，脚本内容如下
```
#!/bin/bash

# gitlab 机房备份路径
LocalBackDir=/var/opt/gitlab/backups

# 远程备份服务器 gitlab备份文件存放路径
RemoteBackDir=/root/gitlabDataBackup

# 远程备份服务器 登录账户
RemoteUser=root

# 远程备份服务器 IP地址
RemoteIP=（你的远程服务器地址）请自己修改

#当前系统日期
DATE=`date +"%Y-%m-%d"`

#Log存放路径
LogFile=$LocalBackDir/log/$DATE.log

# 查找 本地备份目录下 时间为60分钟之内的，并且后缀为.tar的gitlab备份文件
BACKUPFILE_SEND_TO_REMOTE=$(find /var/opt/gitlab/backups3 -type f -mmin -60  -name '*.tar*')

#新建日志文件
touch $LogFile

#追加日志到日志文件
echo "Gitlab auto backup to remote server, start at  $(date +"%Y-%m-%d %H:%M:%S")" >>  $LogFile
echo "---------------------------------------------------------------------------" >> $LogFile

# 输出日志，打印出每次scp的文件名
echo "---------------------The file to scp to remote server is: $BACKUPFILE_SEND_TO_REMOTE-------------------------------" >> $LogFile


#备份到远程服务器
scp $BACKUPFILE_SEND_TO_REMOTE $RemoteUser@$RemoteIP:$RemoteBackDir

#追加日志到日志文件
echo "---------------------------------------------------------------------------" >> $LogFile
```
    要执行脚本文件，需要修改定时远程备份脚本auto_backup_to_remote.sh的权限
```
chmod 777 auto_backup_to_remote.sh
```

### 3.创建定时备份任务
    编辑/etc/crontab 文件:
```
vim /etc/crontab
```
    添加定时备份和定时发送任务
```
# /etc/crontab: system-wide crontab
# Unlike any other crontab you don't have to run the `crontab'
# command to install the new version when you edit this file
# and files in /etc/cron.d. These files also have username fields,
# that none of the other crontabs do.

SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# m h dom mon dow user  command
17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
25 6    * * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6    * * 7   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6    1 * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
#

# 添加定时备份任务，每天凌晨两点，执行gitlab备份
0  2    * * *   root    /opt/gitlab/bin/gitlab-rake gitlab:backup:create CRON=1

# 添加定时发送任务，每天凌晨三点，执行gitlab备份到远程服务器
0 3    * * *   root   /data/gitlabData/backups/auto_backup_to_remote.sh
```
    编写完 /etc/crontab 文件之后，需要重新启动cron服务
```
#重新加载cron配置文件
sudo /usr/sbin/service cron reload
#重启cron服务
sudo /usr/sbin/service cron restart
```

### 4.定时删除B服务器上的备份
    由于备份文件较大，每天备份一次，不久B服务器上的空间便会被占满，因此我们需要定期清理备份文件
    我们在此规定：

> 备份文件超过10天的都会被删除

#### 4.1创建删除过期备份文件的脚本
    编写脚本auto_remove_old_backup.sh
```
#!/bin/bash

# 远程备份服务器 gitlab备份文件存放路径
GitlabBackDir=/root/gitlabDataBackup

# 查找远程备份路径下，超过10天 且文件后缀为.tar 的 Gitlab备份文件 然后删除
find $GitlabBackDir -type f -mtime +10 -name '*.tar*' -exec rm {} \;
```

    修改auto_remove_old_backup.sh脚本权限为777
```
chmod 777 auto_remove_old_backup.sh
```
#### 4.2 创建定时删除任务
    编辑B服务器上的 /etc/crontab 文件
```
vim /etc/crontab
```
    编写定时任务
```
# /etc/crontab: system-wide crontab
# Unlike any other crontab you don't have to run the `crontab'
# command to install the new version when you edit this file
# and files in /etc/cron.d. These files also have username fields,
# that none of the other crontabs do.

SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# m h dom mon dow user  command
17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
25 6    * * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6    * * 7   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6    1 * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
#

# 添加定时删除任务，每天凌晨4点，执行删除过期的Gitlab备份文件
0  4    * * *   root  /root/gitlabDataBackup/auto_remove_old_backup.sh

```

    同样，编写完 /etc/crontab 文件之后，需要重新启动cron服务
```
#重新加载cron配置文件
sudo /usr/sbin/service cron reload
#重启cron服务
sudo /usr/sbin/service cron restart
```

## 三、结束
    至此从gitlab从本地备份到远程备份结束

> 参考：http://blog.csdn.net/ouyang_peng/article/details/77334215


> https://packages.gitlab.com/gitlab/gitlab-ce/packages/ubuntu/trusty/gitlab-ce_10.4.2-ce.0_amd64.deb
