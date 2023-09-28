## 一些问题

### LF和CRLF问题

`git config core.autocrlf false`

### .gitignore不生效

> [.gitignore不生效问题解决方法](https://blog.csdn.net/Saintmm/article/details/120847019)

- 清除缓存

  ```sh
  git rm -r --cached .
  git add .
  git commit -m "update gitignore"
  ```

- 手动添加忽略路径

  ```sh
  # 在PATH处输入要忽略的文件
  git update-index --assume-unchanged [PATH]
  ```

## 指令

### 保存用户名密码

```sh
git config --global credential.helper store
```

### 不追踪某些文件

> https://www.cnblogs.com/lovelyli/p/13359421.html

```sh
# 关闭跟踪app目录下后缀为.xml的文件
git update-index --assume-unchanged "/app/*.xml"
# 打开跟踪app目录下的所有文件
git update-index --assume-unchanged "/app/"
# git打开跟踪文件修改提交
git update-index --no-assume-unchanged "/root/tem/java/web/application-dev.yml"
```

