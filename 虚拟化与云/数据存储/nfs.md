## 安装部署

> [NFS在Linux下的安装、部署与应用 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/370261171)

### 服务端安装

1. 安装

   - CentOS

     ```sh
     yum install nfs-utils
     ```

   - Ubuntu

     ```sh
     apt install nfs-kernel-server
     ```

2. 配置

   ```sh
   # 开机自启
   systemctl enable rpcbind
   systemctl enable nfs
   # 启动
   systemctl start rpcbind
   systemctl start rpcbind
   ```

3. 共享

   - 配置共享目录

     ```sh
     vi /etc/exports
     # 添加需要共享的目录：
     /shared_dir/ 192.168.0.0/24(rw,sync,no_root_squash,no_all_squash)
     ```

     - /shared_dir: 共享目录位置。
     - 192.168.0.0/24: 客户端 IP 范围，本文是限制某个子网，如果是* 代表没有限制。
     - rw: 权限设置，可读可写。
     - sync: 同步共享目录。
     - no_root_squash: 可以使用 root 授权。
     - no_all_squash: 可以使用普通用户授权。

   - 检查

     ```sh
     showmount -e localhost
     ```

### 客户端操作

1. 安装

   同服务端

2. 检查

   ```sh
   showmount -e 192.168.0.101
   ```

   查看服务端的共享目录

3. 挂载

   ```sh
   mount -t nfs 192.168.0.101:/data /mnt/data
   ```

   将服务端的/data挂载到本地的/mnt/data



## 生产环境下的配置

> [生产环境NFS服务配置应该考虑的因素 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/370261096)