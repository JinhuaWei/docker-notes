# RunC 学习笔记

## 带着疑问学习
### Docker容器可以cpu、memory等资源限制，在容器中，如何查询当前容器可使用的资源？
在真是物理环境中，可以通过`proc`文件系统获取，当前可以使用的CPU的信息
```
cat /proc/cpuinfo
```
