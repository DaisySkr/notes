### git本地提交时，忽略某文件的提交、取消忽略、忽略列表

本地提交，有时不想修改.gitignore或其他项目文件，那么可以通过git命令来忽略文件，使该文件不会被更新，命令如下：

```bash
// 1.将文件加入忽略队列
git update-index --assume-unchanged 你的文件路径	
 
// 2.将文件移出忽略队列
git update-index --no-assume-unchanged 你的文件路径
	
// 3.列出所有被忽略的文件
git ls-files -v | grep '^h\ '
 
// 4.列出所有被忽略文件的路径
git ls-files -v | grep '^h\ ' | awk '{print $2}'
 
// 5.所有被忽略的文件，取消忽略
git ls-files -v | grep '^h' | awk '{print $2}' |xargs git update-index --no-assume-unchanged
```

参考资料：

[git忽略某个文件的提交](https://blog.csdn.net/cainiao1412/article/details/109517218)

[git update-index --assume-unchanged 找出所有被忽略的文件的办法](https://blog.csdn.net/DaSunWarman/article/details/79384307)