# 与 OpenFyde 交互

## 目标

你已经成功制作出了你的 openFyde 可启动镜像，干得好！🎊

我猜现在你已经按捺不住，想要登入到 openFyde 来一窥究竟了！但在此之前，你需要：

- 挂载 openFyde 设备的根文件系统
- 为默认 shell 用户 chronos 设置密码
- 启用密码认证

## 准备工作

在目标设备上启动 openFyde 镜像，并确保你可通过物理输入设备，例如键盘、显示器和鼠标来进行访问操作。

## 操作流程

1. 在目标设备上启动 openFyde 并连接到网络，调用 shell 命令行界面
2. 输入以下指令：

```
(device)$ sudo su

(device)$ mount -o remount,rw /

# change password
(device)$ passwd

# enable PasswordAuthentication
(device)$ sed -i "s/PasswordAuthentication.*/PasswordAuthentication yes/g" /etc/ssh/sshd_config

# restart sshd
(device)$ initctl restart openssh-server
```

1. 现在你能够通过密码验证从另一台设备通过 ssh 客户端连接到你的 openFyde 设备上。

## 工作原理

这里如果你选择不挂载根文件系统，密码验证会提示失败。如果你想处理根文件系统，一定要记得把根文件系统挂载为rw模式。
