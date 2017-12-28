---
title: Ubuntu上装windows10结果进不到Ubuntu了- -||
author: smona
published: true
date: '2017-06-19 23:09:01'
layout: post
---

# 前言
  感觉好久没有写博客了，最近都在忙工作，然后也在搞离校的事情，大学毕业嘛，杂七杂八的事情接踵而至，而最近又多了个搬家的任务，所以博客感觉荒废了很久，其实最近也有挺多东西想写下来的，但是感觉短时间内总结不完，所以只是在草稿写一点是一点，而没有post出来。
  事情是这样的，昨晚回到宿舍，然后见到一群人围着俺们舍长，俺们舍长原本被我安利linux，所以就把windows删除了，只是留了个ubuntu，但是现在过了一段学期，换了一块SSD，就想念windows了(虽然我不太喜欢windows啦，但是每个人的都有自己的需求的嘛，我倒是觉得他是想体验一下SSD上跑win的快感，因为linux在HDD和SSD上都是很快的，感受不出来，而HDD里面像蜗牛一样的win，到SSD却能有不错的体验)，所以一群人就噼里啪啦地把原本挂载/home的那块分区用DiskGenius删除了给windows安装用。然而，"贪婪"的他，还是想能够启动到linux，因为原本的linux毕竟还是没有完全删除嘛，然后就用win下的easyBCD工具加入了linux grub引导，然后重启再进入linux的时候，发现卡在了"只有一条下划线一直在跳动....跳动......跳动"的画面: )，然后一群人一哄而散，说linux不好，然后鼓吹windows，这让我如何受得了，我这么一个连macOS都鄙视的人当然要站出来搞定这个"只有一条下划线一直在跳动....跳动......跳动"的问题。

##  Try Ubuntu
  可能你们都已经知道问题可能出哪里了，因为当中唯一的操作就是用DiskGenius删除/home目录所挂载的那个磁盘空间，可是我当时没觉得一个/home目录会导致什么问题，我的/boot和/都在，为什么启动不了？好！具体问题具体分析，因为给俺们舍长装的是ubunut16.04所以，就google了一番，发现网友们给出的方案里面最低成本尝试的有一个是：可以用原本安装镜像里面的try ubuntu去修复grub。事不宜迟，我马上拿出了我之前给俺们舍长安装ubuntu用的小U盘，F12，选择以U盘引导，进入了安装系统，顺便说一句，俺们舍长的电脑是Lenovo G480，没有升级过CPU，内存或者显卡。
  选择Try Ubuntu，然后接下来就是打开终端来操作啦!(Linux的终端真是种享受，如果有自己的配置文件在就更好了)
```
1.sudo su - root
# 或者sudo -i总之能获得root管理员权限就可以了

2.fdisk -l
# 然后找到你原来装linux的磁盘，比如说是/dev/sda，再假设原来his挂载/boot到/dev/sda1，挂载/到/dev/sda3，而/home到/dev/sda4，现在/dev/sda4被删除并且安装了windows而且是以/dev/sda4主引导启动的(看到/dev/sda4前面的*没有？那个就是主引导)

3.mount /dev/sda1 /mnt
# 挂载需要修复引导的系统的 / 目录对应的/dev/sda1到/mnt

4.mount /dev/sda3 /mnt/boot
# 如果 /boot 分区是单独分区的，那么也需要挂载

5.for i in /dev /dev/pts /proc /sys; do sudo mount -B $i /mnt$i; done
# 把剩下的运行系统目录也挂上去

6.chroot /mnt
进入需要修复引导的系统

7.grub-install /dev/sda && update-grub
# update-grub会搜索所有接入的磁盘中的系统，然后放入MBR中
# 重新安装 GRUB 到 MBR, 并更新 GRUB
其实对于主引导记录来说就，它并不是存放于/dev/sda3，也不是/dev/sda1，更加不是/dev/sda4上，而是有一个单独的小空间给他，所以这里只用写/dev/sda就可以了，而不用写/dev/sda3或者/dev/sda1了
```
Reference：[Ubuntu 系统的引导修复](http://www.dreamxu.com/ubuntu-boot-repair/)
搞定以后重启，惊喜地发现，又出现了熟悉的grub启动选择，然而问题在我enter进入第一个选项ubuntu，又出现了一个新问题，这次不是"只有一条下划线一直在跳动....跳动......跳动"的问题，而是没有完全进入系统，只是返回了一个rescovery mode一样的shell给我

##  Read-Only file system?

  首先shell根据给出的提示，去查看系统日志 journal -xb，发现出错的地方有好几个，经验告诉我得直接看到最下面最新的标红的错误，然后向前面找历史的第一次报错(一般都是标红)的地方，发现是启动的时候挂载/home到某个磁盘的时候timeout了，回想之前的删除/home所挂载的磁盘空间来装windows的操作，故我分析可能是这个/home的挂载的问题导致timeout。
  
  vim打开/etc/fstab，删除/home挂载配置那两行，保存，结果告诉我这个文件是只读，好吧可能是这个文件没有给w权限，退出vim，ls -l发现已经是664权限了，而且我都忘了自己已经在root管理员权限下了，稍微尝试执行其他的一些touch和mkdir之类的操作，得到的提示"Read-Only file system"，谷歌一下，说这种情况下可能你已经在rescovery mode下，所以都是只读的，如果要修改文件的话需要重新挂载。
  好吧，那我就
```
mount -t ext4 -o remount  rw /dev/sda1 /
```
  以及
```
mount -t ext4 -o remount  rw /dev/sda3 /boot
```
重新挂载后就可以读写了。

  然后继续vim打开/etc/fstab，删除/home挂载配置那两行并保存，这回保存成功了，然后在家目录新建一个和之前用户同名的目录，不然待会儿重启之后就算输入对了密码还是登不进去。

  如果忘记在家目录新建一个和之前用户同名的目录，那也不打紧，我们手贱reboot了，然后正常启动了，输入对了密码还是登不进去的原因是/home原来的磁盘空间没有了，所以/home下原本的用户目录也被删除了，导致即使账户密码正确也无法加载用户配置。
  而我们重新进入recovery mode的方法是在grub启动选择的时候，选择ubuntu advanced，然后就是选recovery mode即可，进去后新建一个用户目录，重启，登入系统，OK!
  
##  Recovery Mode安全问题

经过上面的操作，有些处女座可能会有疑惑了，这recovery mode不需要密码就可以进去了(bypass)，而且权限还那么高，那岂不是很不安全？有没有办法可以让它稍微安全一点，至少不是那种傻瓜学一下就可以break我的OS的程度呢？
提供几个思路:
1.加密你的数据盘;
2.无论何时,root都需要密码;
3.禁用recovery mode;
4.加密Grub;
具体实现得你自己去搜索了~

"Once someone has access to the machine it becomes very hard to completely secure it."
"One of the reasons I can see to be this way is that an admin can allways forget his password and needs a way to get in the system"
所以建议还是要尽可能防止别人物理接触你的电脑


2017.7.27补充：
Ubuntu禁用的Recovery Mode方法挺简单的，我就直接写了：
```
vim /etc/default/grub
```
找到下面这两行
```
# Uncomment to disable generation of recovery mode menu entries
#GRUB_DISABLE_RECOVERY="true"
```
去掉注释的'#',改成：
```
# Uncomment to disable generation of recovery mode menu entries
GRUB_DISABLE_RECOVERY="true"
```
最后再更新grub就禁用了Recovery Mode了
```
sudo update-grub
```

Reference:  
1.[Migrating To Encrypted Home Directory](http://blog.dustinkirkland.com/2009/06/migrating-to-encrypted-home-directory.html)  
2.[How To Secure Grub Recovery Mode](https://askubuntu.com/questions/31605/how-to-secure-grub-recovery-mode)  
2.[How Can I Prevent Someone From Resetting My Password With A Live CD](https://askubuntu.com/questions/76987/how-can-i-prevent-someone-from-resetting-my-password-with-a-live-cd/78051#78051)  
3.[How To Disable Recovery Mode Single User Mode](https://askubuntu.com/questions/186176/how-to-disable-recovery-mode-single-user-mode)  
4.[Password protect your GRUB entries](https://ubuntuforums.org/showthread.php?t=7353)  
