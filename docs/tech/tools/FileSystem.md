# FileSystem Notes

开个坑，介绍一些常见的文件系统.

### NFS文件系统
[NFS文件系统参考](https://docs.redhat.com/zh_hans/documentation/red_hat_enterprise_linux/7/html/storage_administration_guide/ch-nfs#ch-nfs)

NFS文件系统的全称是Network File System，它允许远程主机通过网络挂载文件系统，并像它们是本地挂载的文件系统一样与它们进行交互，可以实现多台服务器之间数据共享

创建后使用`sudo mount 192.168.189.1:/overlay/nfs/hjj /home/hjj/nfs`命令来将nfs服务器上的文件系统挂载到本地服务器目录

[NFS服务器配置](https://www.cnblogs.com/zeq912/p/9606105.html)

### FAT32文件系统

### NTFS文件系统

### OverlayFS文件系统

### ZFS文件系统