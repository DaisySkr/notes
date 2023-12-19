#  GitHub配置多账号

#### 1.配置账号

```js
// 可以配置一个全局账号，一个局部账号
git config --global user.name "yangyuzhuo"
git config --global user.email "yangyz@zhiwyl.com"

// 查看全局配置
git config --list

// 配置局部账号
git config user.name "DaisySkr"
git config user.email "3218387007@qq.com"

// 查看本地配置
git config user.email
git config --local --list
```

#### 2.生成公私钥SSH

```js
// 生成两个邮箱对应的ssh公私钥
ssh-keygen -t rsa -C "yangyz@zhiwyl.com"
ssh-keygen -t rsa -C "3218387007@qq.com"
```

#### ![image-20231219092355783](D:\work\testcode\readme\github\github配置多账号.assets\image-20231219092355783.png)3.在对应账号配置公钥SSH

将第2步对应账号.pub文件中的内容复制到下边

#### ![image-20231219092728360](D:\work\testcode\readme\github\github配置多账号.assets\image-20231219092728360.png)4.配置config

config 文件在 默认生成公私钥的位置(.ssh目录下)，如果没有这个文件，可以自行手动创建

![image-20231219093031180](D:\work\testcode\readme\github\github配置多账号.assets\image-20231219093031180.png)

```nginx
# 举例
# Host：仓库网站的别名，随意取
# HostName：仓库网站的域名（PS：IP 地址应该也可以）
# User：仓库网站上的用户名
# IdentityFile：私钥的绝对路径
# Port：SSH默认端口号为22，某些私有部署的git仓库会更换端口号


# 配置 daisyskr	ssh -T git@daisyskr
Host daisyskr
HostName github.com
User DaisySkr
IdentityFile ~/.ssh/id_rsa_daisyskr

# 配置 zwyl
Host zwyl
HostName 192.168.1.91    #这也可以是 ip 地址
User yangyuzhuo
IdentityFile ~/.ssh/id_rsa
Port 9680
```

#### 5.测试

```nginx
$ ssh -T git@daisyskr
Hi DaisySkr! You've successfully authenticated, but GitHub does not provide shell access.
```

#### 6.拉取代码

```js
//初始化仓库
git init
//查看账户信息
git config --list --local
git config --list --global
//配置本地账户信息
git config user.name "DaisySkr"
git config user.email "3218387007@qq.com"
//拉取仓库代码（注意将github.com替换成config文件中配置的Host名，p）
git clone git@daisyskr:DaisySkr/code.git
```

![image-20231219095826907](D:\work\testcode\readme\github\github配置多账号.assets\image-20231219095826907.png)

![image-20231219094433586](D:\work\testcode\readme\github\github配置多账号.assets\image-20231219094433586.png)

![image-20231219095100388](D:\work\testcode\readme\github\github配置多账号.assets\image-20231219095100388.png)

#### 补充知识：

```
git config --global 这个命令是改变git的全局配置文件。 
实际上面git有三种配置文件， local 、global 、sysytem，他们的优先级是local > global > system
```

#### 参考资料：

[如何使用Git上传项目代码到github]: https://cloud.tencent.com/developer/article/1434350
[Git 多账号配置]: https://blog.csdn.net/DespairC/article/details/125148215
[Git 多用户配置踩坑]: https://blog.csdn.net/q1025387665a/article/details/128375828?spm=1001.2014.3001.5502
[Git上配置多个不同的账号]: https://blog.csdn.net/YKQi_/article/details/81909382?spm=1001.2101.3001.6650.5&amp;utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-5-81909382-blog-125148215.235%5Ev39%5Epc_relevant_yljh&amp;depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-5-81909382-blog-125148215.235%5Ev39%5Epc_relevant_yljh&amp;utm_relevant_index=9
[git切换用户、多用户切换的正确方式 git commit和git push 切换用户]: https://zhuanlan.zhihu.com/p/345915480
[【github】如何将私有仓库公有化？]: https://blog.csdn.net/weixin_45906196/article/details/123531954
[git config命令使用]: https://zhuanlan.zhihu.com/p/76467410

