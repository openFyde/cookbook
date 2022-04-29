# Before hack

这个recipe描述了如何重现挂载openFyde的rootfs为rw并且开始ssh 密码登录。

## Preparation

Nothing.

## How to cook

1. 登录openFyde, 按下 `Ctrl + t + alt`，输入 `shell`,之后输入
```
$ sudo su

$ mount -oremount,rw /

# change password
$ passwd

# enable PasswordAuthentication
$ sed -i "s/PasswordAuthentication.*/PasswordAuthentication yes/g" /etc/ssh/sshd_config

# restart sshd
$ initctl status openssh-server
```

之后便可以通过 ssh连接openFyde了。
