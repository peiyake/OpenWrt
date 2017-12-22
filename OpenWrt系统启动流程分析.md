<link rel="stylesheet" href="http://yandex.st/highlightjs/6.2/styles/googlecode.min.css">
 
<script src="http://code.jquery.com/jquery-1.7.2.min.js"></script>
<script src="http://yandex.st/highlightjs/6.2/highlight.min.js"></script>
 
<script>hljs.initHighlightingOnLoad();</script>
<script type="text/javascript">
 $(document).ready(function(){
      $("h2,h3,h4,h5,h6").each(function(i,item){
        var tag = $(item).get(0).localName;
        $(item).attr("id","wow"+i);
        $("#category").append('<a class="new'+tag+'" href="#wow'+i+'">'+$(this).text()+'</a></br>');
        $(".newh2").css("margin-left",0);
        $(".newh3").css("margin-left",20);
        $(".newh4").css("margin-left",40);
        $(".newh5").css("margin-left",60);
        $(".newh6").css("margin-left",80);
      });
 });
</script>
<div id="category"></div>
## 一.主要流程
跟其他unix系统一样,openwrt系统启动,首先是boot加载kernel,kernel最终调用`/etc/preinit`开始整个启动流程.

原文请参考:[Preinit and Root Mount and Firstboot Scripts](https://wiki.openwrt.org/doc/techref/preinit_mount)

	preinit -> init -> inittab -> rcS -> /etc/rc.d/^S* (S开头的脚本)
![](https://i.imgur.com/WGZ1eXV.png)
## 二.各流程理解
### 1. preinit流程

+ **文件位置:**`1.8.xx\package\base-files\files\etc`

+ **源文件:**

		#!/bin/sh
		# Copyright (C) 2006 OpenWrt.org
		# Copyright (C) 2010 Vertical Communications
		
		#初始化PATH环境变量
		export PATH=/bin:/sbin:/usr/bin:/usr/sbin:/appfs/usr/sbin:/appfs/usr/bin
		
		#定义的一些变量供后期使用
		pi_ifname=
		pi_ip=192.168.1.1
		pi_broadcast=192.168.1.255
		pi_netmask=255.255.255.0
		
		fs_failsafe_ifname=
		fs_failsafe_ip=192.168.1.1
		fs_failsafe_broadcast=192.168.1.255
		fs_failsafe_netmask=255.255.255.0
		
		fs_failsafe_wait_timeout=2
		
		pi_suppress_stderr="y"
		pi_init_suppress_stderr="y"
		pi_init_path="/bin:/sbin:/usr/bin:/usr/sbin"
		pi_init_cmd="/sbin/init"	
		
		#加载functions.sh和boot.sh 可以理解为C语言的#include<xxx>
		#注意. shell.sh和 source shell.sh执行方式的差别
		. /lib/functions.sh
		. /lib/functions/boot.sh
		
		#设置5个hook结点,每个结点可以理解成一个步骤或者一种模式
		boot_hook_init preinit_essential	#必要流程,挂在proc文件系统等操作
		boot_hook_init preinit_main			#主要流程
		boot_hook_init failsafe				#进入安全模式使用
		boot_hook_init initramfs			#INITRAMFS不为NULL时才执行
		boot_hook_init preinit_mount_root	#INITRAMFS=1时才走此流程
		
		
		#把/lib/preinit/下所有脚本执行一遍,并且所有脚本函数都加载到本shell
		for pi_source_file in /lib/preinit/*; do
		    . $pi_source_file
		done
		
		#执行preinit_essential钩子结点的操作
		boot_run_hook preinit_essential
		
		#不知道干啥
		pi_mount_skip_next=false
		pi_jffs2_mount_success=false
		pi_failsafe_net_message=false
		
		#执行preinit_main钩子结点的操作
		boot_run_hook preinit_main
+ **hook理解**
	
preinit脚本中调用的`boot.sh`中设置钩子函数,主要是这三个函数:

	boot_hook_init
	boot_hook_add
	boot_run_hook
首先,`preinit`调用`boot_hook_init`设置了5个钩子点,你可以想象成一个人开了5间房,先放在一边.

然后,

	for pi_source_file in /lib/preinit/*; do
	    . $pi_source_file
	done
这个for循环,执行各个脚本. 注意:每个脚本的最后都调用了`boot_hook_add`,此函数有两个参数,第一个参数是指定需要注册的钩子点(进入哪个房间),第二个参数指定了在钩子内计划执行的操作(相当于进入房间后,计划一下要干什么).

最后,preinit先后调用了`boot_run_hook preinit_essential`和`boot_run_hook preinit_main`,意思是执行这两个钩子点挂载的钩子操作. 也就是刚刚说到的.

+ **preinit结尾**:

preinit整个操作最后调用的一个钩子操作是`99_10_run_init`

	#!/bin/sh
	# Copyright (C) 2006 OpenWrt.org
	# Copyright (C) 2010 Vertical Communications
	
	run_init() {
	    preinit_echo "- init -"
	    preinit_ip_deconfig
	    if [ "$pi_init_suppress_stderr" = "y" ]; then
		exec env - PATH=$pi_init_path $pi_init_env $pi_init_cmd 2>&0
	    else
		exec env - PATH=$pi_init_path $pi_init_env $pi_init_cmd
	    fi
	}
	
	boot_hook_add preinit_main run_init
可以看到,`run_init`最终调用`exec env - PATH=$pi_init_path $pi_init_env $pi_init_cmd`,这里的`pi_init_cmd`就是在脚本开始处指明的init程序`/sbin/init`,然后开始init流程~!

### 2. init流程

+ **源文件:**`init`可执行程序由`busybox`编译得到.
+ **过程:**`init`程序最终会演变成init进程,一直存在于系统. 这个过程中它会解析`/etc/inittab`文件,并根据规则执行inittab中预定的操作.

### 3. inittab

+ **源文件:**`1.8.xx\package\base-files\files\etc`
+ **源码解析:**

		::sysinit:/etc/init.d/rcS S boot
		::shutdown:/etc/init.d/rcS K shutdown
		ttyS0::askfirst:/bin/ash --login
		tty1::askfirst:/bin/ash --login
有关`inittab`文件的介绍请看这里:[inittab文件说明](https://git.busybox.net/busybox/tree/examples/inittab)

其中第一行:`::sysinit:/etc/init.d/rcS S boot`指定,在系统启动或者重启时才执行,执行的程序是:
`/etc/init.d/rcS S boot`
接下来就进入了`rcS`的执行过程

### 4. rcS

+ **源文件:**`1.8.xx\package\base-files\files\etc\init.d\rcS`
+ **源码:**

		#!/bin/sh
		# Copyright (C) 2006 OpenWrt.org
		
		#查找所有/etc/rc.d下所有S开头的脚本,依据优先级依次执行
		run_scripts() {
			for i in /etc/rc.d/$1*; do
				[ -x $i ] && $i $2 2>&1
			done | $LOGGER
		}
		
		system_config() {
			config_get_bool foreground $1 foreground 0
		}
		
		LOGGER="cat"
		[ -x /usr/bin/logger ] && LOGGER="logger -s -p 6 -t sysinit"
		
		. /lib/functions.sh
		
		config_load system
		config_foreach system_config system
		
		if [ "$1" = "S" -a "$foreground" != "1" ]; then
			run_scripts "$1" "$2" &
		else
			run_scripts "$1" "$2"
		fi
+ **rcS最终**

`/etc/rc.d/^S*`执行每个脚本,执行完毕,系统也就启动完成了.

## 三. /etc/init.d/自定义启动脚本
写法很简单,看这里[官方文档](https://wiki.openwrt.org/doc/techref/initscripts#enableanddisable)
## 四. 如何实现固件中程序开机自启动

### 1. 方法
+ 首先,要先实现一个自定义的模块可以编译到固件中,比如下面是一个例子的目录结构:
![](https://i.imgur.com/SKy2HNr.png)
+ 然后,编写程序启动/停止控制脚本 `files/demo.init`,例如:

		#!/bin/sh /etc/rc.common
		START=10
		STOP=15
		
		start()
		{
			echo "process demo start!"
			/usr/sbin/demo &	#这个方法不可取,好像可以直接使用service_start函数启动,还没有研究
		}
		stop()
		{
			echo "process demo stop!"
			killall demo
		}
**如果要实现程序开机自启动,脚本中的`START`定义必不可少,如果定义了`START`和`STOP`,那么固件编译过程中会自动在最终文件系统`/etc/rc.d/`目录下创建程序开机启动和关机停止的软连接.这样根据系统启动流程中描述,就可以知道程序会在系统启动过程中被执行**
+ 最后,还有一点必不可少,就是在`Makefile`的install中必须将`demo.init`拷贝到`/etc/init.d`目录下,如下:

		define Package/demo/install
			$(INSTALL_DIR) $(1)/usr/sbin
			$(INSTALL_BIN) $(PKG_BUILD_DIR)/demo $(1)/usr/sbin/
			$(INSTALL_DIR) $(1)/etc/init.d
			$(INSTALL_BIN) files/demo.init $(1)/etc/init.d/demo
		endef

按照此方法,编译最终固件烧入硬件设备,那么这个`demo`程序在开机后就直接启动了!!
### 2. 原理
`$TOPDIR/package/Makefile`中会遍历`/etc/init.d/`下所有控制脚本,并对所有脚本调用`enable`函数(该函数在`/etc/rc.common`中被定义),如果脚本中定义了`START`和`STOP`就在`/etc/rc.d/`目录下创建软连接.

+ `$TOPDIR/package/Makefile`部分代码:
![](https://i.imgur.com/G9mHf4f.png)
+ `/etc/rc.common`文件中`enable`函数定义:
![](https://i.imgur.com/U2gbu5S.png)

## 附件
### 代码1

		include $(TOPDIR)/rules.mk
		
		PKG_NAME:=demo
		PKG_VERSION:=2015-08-24
		PKG_RELEASE:=
		
		#PKG_SOURCE_LOCAL:=$(TOPDIR)/local/$(PKG_NAME)
		
		#PKG_BUILD_DIR:=$(BUILD_DIR)/$(PKG_NAME)$(if $(PKG_VERSION),-$(PKG_VERSION))
		#PKG_BUILD_DIR:=$(KERNEL_BUILD_DIR)/$(PKG_NAME)$(if $(PKG_VERSION),-$(PKG_VERSION))
		PKG_BUILD_DIR:=$(BUILD_DIR)/$(PKG_NAME)
		
		#include $(INCLUDE_DIR)/package-version-override.mk
		include $(INCLUDE_DIR)/package.mk
		
		define Package/demo
		  SECTION:=local
		  CATEGORY:=Local Packages
		  SUBMENU:=App
		  TITLE:=Package for demo
		endef
		
		define Package/demo/description
		 This package is just a demo!
		endef
		
		
		define Build/Prepare
			rm -fr $(PKG_BUILD_DIR);
			cp -fr src $(PKG_BUILD_DIR);
		endef
		
		#define Build/Compile
		#endef
		
		#define Build/Clean
		#endef
		
		
		define Package/demo/install
			$(INSTALL_DIR) $(1)/usr/sbin
			$(INSTALL_BIN) $(PKG_BUILD_DIR)/demo $(1)/usr/sbin/
			$(INSTALL_DIR) $(1)/etc/init.d
			$(INSTALL_BIN) files/demo.init $(1)/etc/init.d/demo
		endef
		
		$(eval $(call BuildPackage,demo))

