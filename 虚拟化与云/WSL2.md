#### Clashx代理wsl

> [wsl2在NAT网络模式下使用主机clash代理 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/676747307)

- 在~/.proxyrc中写：

  ```sh
  host_ip=$(cat /etc/resolv.conf |grep "nameserver" |cut -f 2 -d " ")
  host_ip=172.23.128.1
  export all_proxy="socks5://$host_ip:7890"
  export https_proxy="http://${host_ip}:7890"
  export http_proxy="http://${host_ip}:7890"
  ```

  需要时，使用：`source .proxyrc`

#### 查看宿主机ip

```sh
ip route show | grep -i default | awk '{ print $3}'
```

