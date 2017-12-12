# OpenWrt--利用虚拟机安装X86之OpenWrt
## 一.编译固件
### 1. 环境安装
[官方文档](https://wiki.openwrt.org/doc/howto/buildroot.exigence)

	git clone https://github.com/openwrt/openwrt.git
	cd openwrt
	./scripts/feeds update -a
	./scripts/feeds install -a
	make menuconfig
### 2.make menuconfig配置项
1.目标平台选择`X86`

	Target System (x86)  ---> 
2.目标镜像增加`VMDK`

	Target Images  ---> 
    	[*] Build VMware image files (VMDK)
3.基础系统根据需要勾选需要的工具

	Base system  --->
		busybox/base-files/firewall/lib*/opkg...等等
4.其它的工具或者软件自行根据需要选择.

### 3. 开始编译
	make V=s	//首次编译时间会很长,耐心等待

编译完成后,在`openwrt\bin\x86`目录下,可以找到`.vmdk`固件,这个是接下来安装虚拟机要用到的.如图:
![](https://i.imgur.com/6CjPzvO.png)
## 二.安装虚拟机系统
1. 在`window`系统电脑任一盘下创建文件夹,比如:`openwrtVM`,将刚刚编译好的固件拷贝到该目录
2. 打开`VMware` 新建虚拟机,并选择`自定义(高级)`类型
3. 选择`VMware版本`,选择`稍后安装操作系统`
4. 操作系统选择`linux`以及`其它linux3.x或者更高版本内核`
5. 自定义虚拟机名称,比如:`OpenWrt`,虚拟机位置选择刚刚我们创建的文件夹`openwrtVM`.点击下一步,出现如图提示:![](https://i.imgur.com/lsmCZZN.png)
选择`继续`,然后进入下一步.
6. 处理器/内存配置根据自身电脑配置自行设定
7. 网络类型选择`桥接模式`
8. I/O控制器使用`默认`,磁盘类型选择`SATA`(*测试过:选择IDE和SCSI的话,我的电脑上openwrt无法启动*).
9. 选择磁盘,`使用现有磁盘`,选择刚才我们创建的目录`openwrtVM`,提示框中选择`保留现有格式`
10. 点击`完成`,这样我们的虚拟机就创建好了.
11. 正常的话,此时就可以开启openwrt虚拟机了.

## 三.进阶配置
### 1.预设root账户密码,使虚拟机启动后登陆密码

[官方参考](https://wiki.openwrt.org/doc/howto/serial.console.password)

第一步:make menuconfig 增加`Login`

	Base system  ---> <*> busybox--->Login/Password Management Utilities  --->  [*] login 
第二步:修改源码配置文件

+ 找到源码中:`package\base-files\files\etc\shadow`文件,修改root用户的密码配置,如图:![](https://i.imgur.com/FbpR0pp.png),图中红框就是我为root用户提前设置的好密码,这是加密后的形式.具体细节请参考[shadow文件说明](https://linux.die.net/man/5/shadow),加密后的密码,你可以事先登录刚刚做好的openwrt系统,然后执行`passwd`命令设置一个密码,然后`cat /etc/shadow`来获取.
+ 找到源码中:`package\base-files\files\etc\inittab`文件修改后如图:![](https://i.imgur.com/9jWcKTJ.png),具体细节请参考:[inittab配置文件说明](https://git.busybox.net/busybox/tree/examples/inittab)
 
第三步:重新编译固件

	make
编译完成后把之前创建的虚拟机删除,重新创建虚拟机.

### 2.启用SSH

2.1 启用ssh需要你之前把`openssh`或者是`dropbear`,这两个可以实现SSH连接.

	Base system  ---> 
		< > dropbear..

或者

	Network  --->
		SSH  ---> 
			-*- openssh-client
			<*> openssh-server.
	 		<*> openssh-sftp-client
			<*> openssh-sftp-server
这里使用Openssh

2.2 启动虚拟机,进入系统后修改ssh配置文件`/etc/ssh/sshd_config`

	PermitRootLogin yes 		//允许root账户通过ssh登录
	然后执行下面命令:
	/etc/init.d/sshd restart  	//重启sshd服务
	/etc/init.d/firewall stop  	//关闭防火墙才可以
2.3 网卡设置

默认情况下,openwrt启动后把虚拟机识别的第一个网卡(openwrt系统把它当做lan口处理),并把eth0挂在`br-lan`桥下. 在这里呢.我的电脑有两张网卡,我想全部利用起来.如下操作.

第一步:关闭虚拟机

第二步:在`虚拟机设置`中新增`网络适配器`,如图:![](https://i.imgur.com/Vb4kPiC.png)这里的`VMnet2`你要先在VMware软件的`编辑->虚拟网络编辑器`中把`VMnet2`桥接到你的第二张网卡.如图:![](https://i.imgur.com/bC42Lsg.png).

第三步:配置完成,重启openwrt后`ifconfig -a`可以看到eth0,eth1两张网卡,由于我的eth0桥接的网卡是连接互联网的,所以我这里需要设置eth0为wan口,eth1设置成lan后,修改配置文件`/etc/config/network`,如图:

![](https://i.imgur.com/xlA6nJh.png)

第四步:修改完成重启网络 `/etc/init.d/network restart`,再次查看,你的eth0可以dhcp自动获取IP,并可以连接互联网. eth1被加在了br-lan桥下.

**接下来你就可以利用虚拟机OpenWrt进行开发啦!!**

## 四.HelloWorld

