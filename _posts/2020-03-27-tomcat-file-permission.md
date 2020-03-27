tomcat 文件权限配置
===

## 问题
测试上传文件接口,上传文件保存到 `webapps` 下的一个静态文件夹 `static/up-imgs`内
所有静态文件通过 `nginx` 进行代理. 发现上传文件成功,但是无法获取, nginx 返回403

## 解决

### 0.访问测试
手动修改该目下文件权限,发现修改后可以正常访问.但问题并没有根除,因为新创建的文件权限仍有问题,我不可能等每一次用户上传文件后手动去修改一下他的权限吧 ^_^.
下图为上传后保存文件的权限格式
![](http://img.shargeebar.cc/20200326233700.png)
看到这个文件,只有用户\组 `sguajiu:sguajiu`,才拥有访问权限.

`ps -aux | grep nginx ` 看到nginx的操作用户分别是 `root` 和`www-data`, 对照上图,他们确实不配访问那个文件
![](http://img.shargeebar.cc/20200326234228.png)
看到这里嘴角浮现一丝微笑,小样.  修改一下 `umask`,保证新创建文件权限默认为`o+r`的话,其他人不就能读了? ^_^

### 1. 修改用户的默认umask配置
修改用户 `umask`, 主文件夹下 `.bash_profile` 或者 `.profile`文件,我的Ubuntu 上没有第一个配置文件,修改第二个
.最后一行加上`umask 022` 对应的 就是 `644 `了
  - 这个地方说一下为什么不是`755`而是`644`呢,应该是处于系统安全性的考虑,umask对于创建文件与文件夹所分配的默认权限是有差别的,新建**文件**不会自动附加`x`执行权限, 所以如果是新建的文件的话,他的权限对应为 `666 - 022 = 644`,对应为`-rw-r--r--`,如果是**文件夹***的话对应则是`777 - 022 = 755`,对应为`-rwx-r-x-r-x`
言归正传
  - 在环境配置文件中加上`umask 022`保存退出,使用`source .profile`重新刷新权限,终端输入`umask` 显示 `0022`,ok配置完成 
  - 重启tomcat,加载最新的环境配置,重启后继续进行测试,测试上传后访问仍旧403 -_- ,这是要干嘛?发现新增文件权限仍旧为 `-rw-r-----`, 使用`touch a && ll` 显示文件a 的权限确实是`-rw-r--r--`没错, 在这个时候事情突然开始变得棘手起来.....  按照国际管理以及我多年的开发经验来看,一个应用程序启动的权限应该是跟这个启动用户的环境配置一致才对
  - 输入 `ps -aux | grep tomcat `
![](http://img.shargeebar.cc/20200327105828.png)确实是我本人启动的它,既然这样,我想我已经猜到了事情的真相了

### 2. 拨开迷雾
经过反复查阅[大量相关文献资料](https://tomcat.apache.org/tomcat-9.0-doc/security-howto.html)^_^,
>File permissions should also be suitably restricted. In the .tar.gz distribution, files and directories are not world readable and the group does not have write access. On Unix like operating systems, Tomcat runs with a **default umask** of `0027` to maintain these permissions for files created while Tomcat is running (e.g. log files, expanded WARs, etc.).

熟悉的剧情,同样是处于安全考虑,tomcat这个二五仔会在启动的时候再次设置`umask`为`0027`,意思就是无论你用户的设置的umask为多少,我tomcat启动的时候都会重新设置一下这个值,设置完成后在启动,这也是为啥第一步设置后新创建文件仍为`-rw-r-----`,可怜的nginx没有权限读取到那个文件,只能返回403了.
找到了问题根源,解决办法:
  - 在tomcat bin目录下,新建 setenv.sh 文件(我的9.0.30下没有这个文件,手动创建),输入`export UMASK=0022`, 再次启动, 搞定.

