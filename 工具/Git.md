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

