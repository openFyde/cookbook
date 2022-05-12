# Talking to the openFyde (via ssh, not megaphone)


## Objectives

You have produced a bootable openFyde image, it boots! 🎊

Now you wish to connect to your openFyde instance and see what's going on inside the mysterious black box. To achieve this, we need to:
 - mount the root file system of the openFyde device
 - set a password for the default shell user, `chronos`
 - enable password authentication


## Preparation

Boot your openFyde image on the target device and make sure you have physical kvm(keyboard, monitor and mouse) access to it.


## Procedures

1. Boot openFyde on the target device, connect to the network, invoke the (bash) shell command-line interface

2. Enter the following commands:

```
(device)$ sudo su

(device)$ mount -o remount,rw /

# change password
(device)$ passwd

# enable PasswordAuthentication
(device)$ sed -i "s/PasswordAuthentication.*/PasswordAuthentication yes/g" /etc/ssh/sshd_config

# restart sshd
(device)$ initctl status openssh-server
```

3. You should now be able to ssh connect to your openFyde device from another machine that has ssh client via password authentication


## How it works

The primary gotcha is that if you don't mount the root file system, `passwd` command would fail. Always remember to mount the root file system to `rw` if you wish to mess with the root file system.
