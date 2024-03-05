#### core dump

```sh
# core dump文件输出
ulimit -S -c unlimited
echo -e "\n# enable coredump whith unlimited file-size for all users\n* soft core unlimited" >> /etc/security/limits.conf

mkdir -p /tmp/coredump/ && chmod 777 /tmp/coredump

echo '/tmp/coredump/%t-%e-%p-%c.core' > /proc/sys/kernel/core_pattern

echo -e "/tmp/coredump/%t-%e-%p-%c.core" > /etc/sysctl.conf

echo -e "1" > /proc/sys/kernel/core_uses_pid

# 永久保存
## 在 /etc/security/limits.conf里添加
@root soft core unlimited
@root hard core unlimited

## 在 /proc/sys/kernel/core_pattern里添加
kernel.core_pattern = /tmp/coredump/%t-%e-%p-%c.core
kernel.core_uses_pid = 1

## 执行
sysctl -p
```
