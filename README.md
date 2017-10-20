基于mailman,Postfix,Apache的Mailinglist系统搭建

目标实现Mailinglist邮件订阅转发等功能

OpenSSL Postfix Dovecot Apache mailman

1.安装部署邮件服务器Postfix

生成ssl证书
1、生成证书的脚本代码

以hostname为命名生成证书，运行脚本后需输入四次相同密码(密码须包含数字和字母)

==========
#!/bin/sh
rm -rf $(hostname).*
 
openssl genrsa -des3 -out $(hostname).key 1024
 
SUBJECT="/C=US/ST=Mars/L=iTranswarp/O=iTranswarp/OU=iTranswarp/CN=$(hostname)"
 
openssl req -new -subj $SUBJECT -key $(hostname).key -out $(hostname).csr
 
mv $(hostname).key $(hostname).origin.key
 
openssl rsa -in $(hostname).origin.key -out $(hostname).key
 
openssl x509 -req -days 3650 -in $(hostname).csr -signkey $(hostname).key -out $(hostname).crt
 
cp $(hostname).crt /etc/pki/tls/certs/$(hostname).crt
cp $(hostname).key /etc/pki/tls/certs/$(hostname).key
 
echo "the key path:/etc/pki/tls/certs/$(hostname).key"
echo "the crt path:/etc/pki/tls/certs/$(hostname).crt"
 
rm -rf $(hostname).*
===================


2、Postfix安装及配置
yum -y install postfix

#vi /etc/postfix/main.cf

以下是postconf -n　输出结果，请根据自己需要定制

=======================================
alias_database = hash:/etc/aliases
alias_maps = hash:/etc/aliases, hash:/etc/mailman/aliases
command_directory = /usr/sbin
config_directory = /etc/postfix
daemon_directory = /usr/libexec/postfix
data_directory = /var/lib/postfix
debug_peer_level = 2
debugger_command = PATH=/bin:/usr/bin:/usr/local/bin:/usr/X11R6/bin ddd $daemon_directory/$process_name $process_id & sleep 5
home_mailbox = Maildir/
html_directory = no
inet_interfaces = all
inet_protocols = all
mail_owner = postfix
mailbox_size_limit = 1073741824
mailq_path = /usr/bin/mailq.postfix
manpage_directory = /usr/share/man
message_size_limit = 10485760
mydestination = $myhostname,$mydomain
mydomain = mailinglist.xxx.com
myhostname = mailinglist.xxx.com
mynetworks_style = class
myorigin = $myhostname
newaliases_path = /usr/bin/newaliases.postfix
queue_directory = /var/spool/postfix
readme_directory = /usr/share/doc/postfix-2.10.1/README_FILES
recipient_delimiter = +
sample_directory = /usr/share/doc/postfix-2.10.1/samples
sendmail_path = /usr/sbin/sendmail.postfix
setgid_group = postdrop
smtpd_banner = $myhostname ESMTP $mail_name
smtpd_relay_restrictions = permit_mynetworks permit_sasl_authenticated defer_unauth_destination
smtpd_sasl_auth_enable = yes
smtpd_sasl_local_domain = $myhostname
smtpd_sasl_path = private/auth
smtpd_sasl_security_options = noanonymous
smtpd_sasl_type = dovecot
smtpd_tls_cert_file = /etc/pki/tls/certs/mailinglist.xxx.com.crt
smtpd_tls_key_file = /etc/pki/tls/certs/mailinglist.xxx.com.key
smtpd_tls_session_cache_database = btree:/etc/postfix/smtpd_scache
smtpd_use_tls = yes
unknown_local_recipient_reject_code = 550
=======================================


vi /etc/postfix/master.cf
=============
# line 26-28: uncomment
smtps  inet n  -  n  -  -  smtpd
 -o syslog_name=postfix/smtps
 -o smtpd_tls_wrappermode=yes
 
======postfix与mailman交互脚本======
# line end: uncomment
mailman   unix  -       n       n       -       -       pipe
  flags=FR user=list argv=/usr/lib/mailman/bin/postfix-to-mailman.py
  ${nexthop} ${user}
=========================



Dovecot 安装及配置
yum -y install dovecot

vim /etc/dovecot/dovecot.conf
	
# line 24: uncomment
protocols = imap pop3 lmtp
# line 30: uncomment and change ( if not use IPv6,　use: listen = * )
listen = *, ::

vim /etc/dovecot/conf.d/10-auth.conf
	
# line 10: uncomment and change ( allow plain text auth )
disable_plaintext_auth = no
# line 100: add
auth_mechanisms = plain login


vim /etc/dovecot/conf.d/10-mail.conf
	
# line 30: uncomment and add
mail_location = maildir:~/Maildir

vim /etc/dovecot/conf.d/10-master.conf
	
# line 96-98: uncomment and add like follows
# Postfix smtp-auth
unix_listener /var/spool/postfix/private/auth {
 mode = 0666
 user = postfix
 group = postfix
}


vim /etc/dovecot/conf.d/10-ssl.conf
	
# line 8: change
ssl = yes
# line 14,15: specify certificates
ssl_cert = </etc/pki/tls/certs/$(hostname).crt
ssl_key = </etc/pki/tls/certs/$(hostname).key

==============

systemctl restart postfix
systemctl enable postfix
systemctl start dovecot
systemctl enable dovecot


调试邮件服务器:
确保ｄｎｓ可以解析到你的域名 ,　本地调试可以修改ｈｏｓｔ文件
echo “hello”|mail user@mailinglist.xxx.com
cat /var/log/maillog
netstat -ant
service postfix status
根据响应报错调整参数

最后在本地使用客户端Thunderbird连接到服务器对外发送邮件测试,qq邮箱收信成功，服务器内部用户收发邮件成功。
个别服务器收不到邮件，请联系服务器管理人员将你的域名添加到对方信任列表。

添加和删除postfix用户，基于系统用户
useradd -s /sbin/nologin user1
passwd user1
userdel user1



安装apache
yum install httpd

安装mailman
yum install mailman

设置mailman与Postfix Apache的交互

#vi  /etc/mailman/mm_cfg.py

修改和添加
DEFAULT_URL_HOST = 'mailinglist.xxx.com'
DEFAULT_EMAIL_HOST = 'mailinglist.xxx.com'
MTA = 'Postfix'


#vi /etc/httpd/conf.d/mailman.conf


RedirectMatch ^/mailman[/]*$ http://mailinglist.xxx.com/mailman/listinfo


设置网站管理密码
/usr/lib/mailman/bin/mmsitepass password


导入crontab
cd /usr/lib/mailman/cron
crontab -u mailman crontab.in

新建list
/usr/lib/mailman/bin/newlist mailman

/usr/bin/newaliases

mailman相关调试用/usr/lib/mailman/bin/check_perms　-f根据报错处理

配置完成将各种服务加入开机自启动。确保ｄｎｓ可以解析到你的域名

部署完成，访问:

http://mailinglist.xxx.com/mailman/listinfo

进行邮件列表的创建，订阅，成员管理等，详见mailman用户手册。

mailman实用的命令，以下是最重要的几个命令。

sudo newlist                     //创建一个新的列表
sudo rmlist "listname"           //删除listname这个列表
sudo list_lists                  //列出所有的列表
sudo list_members "listname"     //列出listname这个列表所有的订阅用户

//将user@example.com添加到listname这个列表
echo "user@example.com" | sudo add_members -w y -r - "listname" 
  
sudo add_members -r file "listname"               //将file文件中的邮箱地址添加到listname这个列表中
sudo remove_members "listname" "user@example.com"  //将user@example.com从listname这个列表中删除
sudo mmsitepass                                    //设置网页管理界面的密码




ref:
http://www.list.org/mailman-install/
https://help.ubuntu.com/lts/serverguide/mailman.html
https://boredwookie.net/blog/m/how-to-install-and-configure-gnu-mailman-on-centos-6-2
http://www.jb51.net/article/113292.htm
