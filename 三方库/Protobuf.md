### protobuf安装

> [Protobuf 安装及使用](https://www.jianshu.com/p/00be93ed230c)

### 导出动态库

```sh
protoc --cpp_out=dllexport_decl=TESSDATA_EXPORT:./ -I F:\Tess_git\TessData\ Distributed.proto

protoc --cpp_out=dllexport_decl=TESSDATA_EXPORT:./ -I F:\Tess_git\TessData\ Snapshot.proto

protoc --cpp_out=dllexport_decl=TESSDATA_EXPORT:./ -I F:\Tess_git\TessData\ Tess.proto
```

