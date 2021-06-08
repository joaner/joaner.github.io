# Centos7迁移/home硬盘空间到根目录

## 问题

根目录的硬盘空间不够，需要从/home目录腾出空间给根目录。

```
[root@localhost ~]# df -h
文件系统                 容量  已用  可用 已用% 挂载点
/dev/mapper/centos-root   36G   20G   17G   54% /
/dev/mapper/centos-home   18G  4.8G   13G   28% /home
```

Google搜索了下，发现国内的博客都是复制粘贴，无法实操。最有用的还是StackOverflow上的答案，所以写这篇文章整理翻译下。

正确答案：https://serverfault.com/questions/771921/how-to-shrink-home-and-add-more-space-on-centos7

## 解决

由于Centos7默认的XFS文件系统不能缩小，只能扩大。

解决思路是先删掉/home卷，再将空闲的硬盘空间追加给根目录。

```bash
# 1. 备份/home卷
tar -czvf /root/home.tgz -C /home .
# 遍历检查备份的文件
tar -tvf /root/home.tgz

# 2. 卸载/home卷
umount /dev/mapper/centos-home
# 遇到错误：target is busy error
# 解决方法：增加-l参数 umount -l /dev/mapper/centos-home

# 3. 删除逻辑卷
lvremove /dev/mapper/centos-home
# 遇到错误：Logical volume centos/home contains a filesystem in use
# 解决方法：由于当前SSH连接使用了/home硬盘的文件，所以需要先退出SSH连接再重新进入。
#          如果依然不行，可能还需要关闭其它进程：yum install psmisc && fuser -kuc /dev/centos/home

# 4. 重新建立一个较小的/home卷（可选）
lvcreate -L 40GB -n home centos
mkfs.xfs /dev/centos/home
mount /dev/mapper/centos-home

# 5. 将空闲的空间追加到根目录
lvextend -r -l +100%FREE /dev/mapper/centos-root

# 6. 恢复/home目录文件
tar -xzvf /root/home.tgz -C /home

# 7. 更新文件系统配置
vi /etc/fstab
# 如果删除了home卷，则需要注释或删除/dev/mapper/centos-home开头的行，否则将无法顺利开机
```

至此，根目录空间扩大了，/home目录文件不变。

```
[root@localhost ~]# df -h
文件系统                 容量  已用  可用 已用% 挂载点
/dev/mapper/centos-root   54G   20G   35G   36% /
[root@localhost ~]# cat /etc/fstab
/dev/mapper/centos-root /                       xfs     defaults        0 0
UUID=d32c3ed5-79cc-4536-b3c8-4e084bc10dd7 /boot                   xfs     defaults        0 0
#/dev/mapper/centos-home /home                   xfs     defaults        0 0
/dev/mapper/centos-swap swap                    swap    defaults        0 0
```


