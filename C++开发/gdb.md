## 指令

```sh
# 查看对应地址的函数名 
info symbol [addr]
# 线程信息
info thread
# 进入某个线程
thread [num]
# 查看当前帧信息，并记住帧的编号（frame number）。
info frame
# 显示该帧中的局部变量。
info locals
# 显示该帧中的函数参数
info args
# 展开所有线程的堆栈
thread apply all bt
```

