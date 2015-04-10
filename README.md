# OpenWRT Hotplug原理分析
本次研究基于OpenWRT 14_07 trunk。其他版本有部分差异，请阅读时注意。

## <a name='TOC'>目录表</a>
  1. [Hotplug原理](#profiles)
  1. [Hotplug应用](#situations)
  1. [参考](#references)


## <a name='profiles'>Hotplug原理</a>

**Hotplug**即热插拔，在新版本`OpenWRT`上，`hotplug`，`coldplug`与`watchdog`等被集成到全新的Procd系统中。  

**Procd**是`OpenWRT`下新的预初始化，初始化，热插拔和事件系统。在`openwrt` 中, `procd` 作为 `init` 进程会处理许多事情, 其中就包括 `hotplug`。`procd`本身并不知道如何处理`hotplug`事件，也没有必要知道，因为它只实现机制，而不实现策略。事件的处理是由配置文件决定的，这些配置文件即所谓的`rules`.。老版本下独立的`hotplug2`在`r36987`被移除了。所以下面我们要介绍的就是新版本下`Hotplug`的机制。  

要了解`Hotplug`运行的整个过程，首先得了解`procd`系统的工作流程。才能从全局了解`hotplug`是如何工作的。在这里我们重点介绍与`hotplug`相关的`procd`启动过程。

### Procd启动过程分析

#### preinit()函数

	void
	preinit(void)
	{
		char *init[] = { "/bin/sh", "/etc/preinit", NULL };
		char *plug[] = { "/sbin/procd", "-h", "/etc/hotplug-preinit.json", NULL };
	
		LOG("- preinit -\n");
	
		plugd_proc.cb = plugd_proc_cb;
		plugd_proc.pid = fork();
		if (!plugd_proc.pid) {
			execvp(plug[0], plug);
			ERROR("Failed to start plugd\n");
			exit(-1);
		}
		if (plugd_proc.pid <= 0) {
			ERROR("Failed to start new plugd instance\n");
			return;
		}
		uloop_process_add(&plugd_proc);
	
		setenv("PREINIT", "1", 1);
	
		preinit_proc.cb = spawn_procd;
		preinit_proc.pid = fork();
	
		if (!preinit_proc.pid) {
			execvp(init[0], init);
			ERROR("Failed to start preinit\n");
			exit(-1);
		}
		if (preinit_proc.pid <= 0) {
			ERROR("Failed to start new preinit instance\n");
			return;
		}
		uloop_process_add(&preinit_proc);
	
		DEBUG(4, "Launched preinit instance, pid=%d\n", (int) preinit_proc.pid);
	}

1. 创建子进程执行`/etc/preinit`脚本，此时`PREINI`T环境变量被设置为1，主进程同时使用`uloop_process_add()`把`/etc/preinit`子进程加入`uloop`进行监控，当`/etc/preinit`执行结束时回调`plugd_proc_cb()`函数把监控`/etc/preinit`进程对应对象中`pid`属性设置为0，表示`/etc/preinit`已执行完成。

2. 创建子进程执行`/sbin/procd -h  /etc/hotplug-preinit.json`，主进程同时使用`uloop_process_add()`把`/sbin/procd`子进程加入`uloop`进行监控，当`/sbin/procd`进程结束时回调`spawn_procd()`函数。

3. `spawn_procd()`函数繁衍后继真正使用的`/sbin/procd`进程，从`/tmp/debuglevel`读出`debug`级别并设置到环境变量`DBGLVL`中，把`watchdog fd`设置到环境变量`WDTFD`中，最后调用`execvp()`繁衍`/sbin/procd`进程。

#### procd进程

在这里我们主要分析`procd`的五个状态,分别为 `STATE_EARLY`、`STATE_INIT`、`STATE_RUNNING`、`STATE_SHUTDOWN`、`STATE_HALT`，这5个状态将按顺序变化，当前状态保存在全局变量`state`中，可通过`procd_state_next()`函数使用状态发生变化。

	static void state_enter(void)
	{
		char ubus_cmd[] = "/sbin/ubusd";
	
		switch (state) {
		case STATE_EARLY:
			LOG("- early -\n");
			watchdog_init(0);
			hotplug("/etc/hotplug.json");
			procd_coldplug();
			break;
	
		case STATE_INIT:
			// try to reopen incase the wdt was not available before coldplug
			watchdog_init(0);
			LOG("- ubus -\n");
			procd_connect_ubus();
	
			LOG("- init -\n");
			service_init();
			service_start_early("ubus", ubus_cmd);
	
			procd_inittab();
			procd_inittab_run("respawn");
			procd_inittab_run("askconsole");
			procd_inittab_run("askfirst");
			procd_inittab_run("sysinit");
			break;
	
		case STATE_RUNNING:
			LOG("- init complete -\n");
			break;
	
		case STATE_SHUTDOWN:
			LOG("- shutdown -\n");
			procd_inittab_run("shutdown");
			sync();
			break;
	
		case STATE_HALT:
			LOG("- reboot -\n");
			reboot(reboot_event);
			break;
	
		default:
			ERROR("Unhandled state %d\n", state);
			return;
		};
	}

* #####STATE_EARLY状态 - init前准备工作
	1. 初始化`watchdog`
	2. 根据"`/etc/hotplug.json`"规则监听`hotplug`
	3. `procd_coldplug()`函数处理，把`/dev`挂载到`tmpfs`中，`fork udevtrigger`进程产生冷插拔事件，以便让`hotplug`监听进行处理
	4. `udevstrigger`进程处理完成后回调`procd_state_next()`函数把状态从`STATE_EARLY`转变为`STATE_INIT`

* #####STATE_INIT状态 - 初始化工作
	1. 连接`ubusd`，此时实际上`ubusd`并不存在，所以`procd_connect_ubus`函数使用了定时器进行重连，而`uloop_run()`需在初始化工作完成后才真正运行。当成功连接上`ubusd`后，将注册`service main_object`对象，`system_object`对象、`watch_event`对象(`procd_connect_ubus()`函数)，
	2. 初始化`services`（服务）和`validators`（服务验证器）全局`AVL tree`
	3. 把`ubusd`服务加入`services`管理对象中`(service_start_early)`
	4. 根据`/etc/inittab`内容把`cmd`、`handler`对应关系加入全局链表`actions`中
	5. 顺序加载`respawn、askconsole、askfirst、sysinit`命令
	6. `sysinit`命令把`/etc/rc.d/`目录下所有启动脚本执行完成后将回调`rcdone()`函数把状态从`STATE_INIT`转变为`STATE_RUNNING`

* #####STATE_RUNNING状态
	1. 进入`STATE_RUNNING`状态后`procd`运行`uloop_run()`主循环 


### Hotplug原理图

`Hotplug`原理的整个流程如下所示：

	-----------------------
	|    procd daemon     |
	|    (hotplug.json)   |
	-----------------------
			  netlink| socket			user space
    -------------------------------------------------
					 |				    kernel space
	-----------------------
	|    (uevent [json])  |
	|    kernel           |
	-----------------------

主要过程分为以下两个部分：  


1.  **内核发出uevent事件**

内核使用`uevent`事件通知用户空间，`uevent`首先在内核中调用`netlink_kernel_create()`函数创建一个`socket`套接字，该函数原型在`netlink.h`中定义。这是一种特殊类型的`socket` ，专门用于内核空间与用户空间的异步通信。  
`kobject_uevent()`产生`uevent`事件`(/lib/kobject_uevent.c)`，事件的部分信息通过环境变量传递，如`$ACTION, $DEVPATH, $SUBSYSTEM`等，产生的`uevent`先由`netlink_broadcast_filtered()`发出，最后调用`uevent_helper[]`所指定的程序来处理。  
在`linux`中，`uevent_helper[]`里默认指定`”/sbin/hotplug”`，但可以通过`/sys/kernel/uevent_helper（kernel/ksysfs.c）`或`/proc/kernel/uevent_helper(kernel/sysctl.c)`来修改成指定的程序。
在新`OpenWRT`中，并不使用`user_helper[]`指定程序来处理`uevent（/sbin/hotplug`不存在，在以前版本中存在），而是通过`PF_NETLINK`套接字来获取来自内核空间的`uevent`。  

2.  **用户空间监听uevent**

在`proc/plug/hotplug.c`中，创建一个`PF_NETLINK`套接字来监听内核`netlink_broadcast_filtered()`发出的`uevent`。收到`uevent`之后，在根据`/etc/hotplug.json`里的描述，定位到对应的执行函数来处理。  
通常情况下，`/etc/hotplug.json`会调用`/sbin/hotplug-call`来处理`uevent`，它根据`uevent`的`$SUBSYSTEM`变量来分别调用`/etc/hotplug.d`下不同目录中的脚本。  
`/sbin/hotplug-call`脚本如下所示，这里面的`$1`表示`hotplug-call`的第一个参数：

	root@OpenWrt:/sbin# cat hotplug-call 
	#!/bin/sh
	# Copyright (C) 2006-2010 OpenWrt.org
	
	export HOTPLUG_TYPE="$1"
	
	. /lib/functions.sh
	
	PATH=/bin:/sbin:/usr/bin:/usr/sbin
	LOGNAME=root
	USER=root
	export PATH LOGNAME USER
	export DEVICENAME="${DEVPATH##*/}"
	
	[ \! -z "$1" -a -d /etc/hotplug.d/$1 ] && {
	        for script in $(ls /etc/hotplug.d/$1/* 2>&-); do (
	                [ -f $script ] && . $script
	        ); done
	}

下表是`hotplug.json`的具体内容，重点关注蓝色字段。

    root@OpenWrt:/etc# cat hotplug.json 

	[
	        [ "case", "ACTION", {
	                "add": [
	                        [ "if",
	                                [ "and",
	                                        [ "has", "MAJOR" ],
	                                        [ "has", "MINOR" ],
	                                ],
	                                [
	                                        [ "if",
	                                                [ "or",
	                                                        [ "eq", "DEVNAME",
	                                                                [ "null", "full", "ptmx", "zero" ],
	                                                        ],
	                                                        [ "regex", "DEVNAME",
	                                                                [ "^gpio", "^hvc" ],
	                                                        ],
	                                                ],
	                                                [
	                                                        [ "makedev", "/dev/%DEVNAME%", "0666" ],
	                                                        [ "return" ],
	                                                ]
	                                        ],
	                                        [ "if",
	                                                [ "or",
	                                                        [ "eq", "DEVNAME", "mapper/control" ],
	                                                        [ "regex", "DEVPATH", "^ppp" ],
	                                                ],
	                                                [
	                                                        [ "makedev", "/dev/%DEVNAME%", "0600" ],
	                                                        [ "return" ],
	                                                ],
	                                        ],
	                                        [ "if",
	                                                [ "has", "DEVNAME" ],
	                                                [ "makedev", "/dev/%DEVNAME%", "0644" ],
	                                        ],
	                                ],
	                        ],
	                        [ "if",
	                                [ "has", "FIRMWARE" ],
	                                [
	                                        [ "exec", "/sbin/hotplug-call", "%SUBSYSTEM%" ],
	                                        [ "load-firmware", "/lib/firmware" ],
	                                        [ "return" ]
	                                ]
	                        ],
	                ],
	                "remove" : [
	                        [ "if",
	                                [ "and",
	                                        [ "has", "DEVNAME" ],
	                                        [ "has", "MAJOR" ],
	                                        [ "has", "MINOR" ],
	                                ],
	                                [ "rm", "/dev/%DEVNAME%" ]
	                        ]
	                ]
	        } ],
	        [ "if",
	                [ "eq", "SUBSYSTEM", "platform" ],
	                [ "exec", "/sbin/hotplug-call", "%SUBSYSTEM%" ]
	        ],
	        [ "if",
	                [ "and",
	                        [ "has", "BUTTON" ],
	                        [ "eq", "SUBSYSTEM", "button" ],
	                ],
	                [ "exec", "/etc/rc.button/%BUTTON%" ]
	        ],
	        [ "if",
	                [ "eq", "SUBSYSTEM",
	                        [ "net", "input", "usb", "ieee1394", "block", "atm", "zaptel", "tty", "button" ]
	                ],
	                [ "exec", "/sbin/hotplug-call", "%SUBSYSTEM%" ]
	        ],
	        [ "if",
	                [ "and",
	                        [ "eq", "SUBSYSTEM", "usb-serial" ],
	                        [ "regex", "DEVNAME",
	                                [ "^ttyUSB", "^ttyACM" ]
	                        ],
	                ],
	                [ "exec", "/sbin/hotplug-call", "tty" ]
	        ],
	]



**[[⬆]](#TOC)**
## <a name='situations'>Hotplug应用</a>

### U盘的自动挂载卸载
`Hotplug`一个常见的实例应用就是U盘或SD卡等外设的自动挂载和卸载功能。所以这里我们主要介绍如何利用`hotplug`实现U盘，移动硬盘等外设自动挂载的方法和原理。本文中的例子还需要根据实际情况作相应适配。  

当然，首先得内核有相应的驱动程序支持才行。当U盘插入后，会产生`uevent`事件，`hotplug`收到这个内核广播事件后，根据`uevent` 事件`json`格式的附带信息内容，在`hotplug.json`中进行定位。事件包含的信息一般为如下所示：

    ACTION(add), DEVPATH(devpath), SUBSYSTEM(block), MAJOR(8), MINOR(1), DEVNAME(devname), DEVTYPE(devtype), SEQNUM(865)

根据上面的信息，就可以在`hotplug.json`中定位到两个条目，如上面`hotplug.json`中蓝色显示字段。第一个条目执行的是`makedev`，该命令会创建设备节点。第二个条目会根据附带信息中的`ACTION, DEVPATH, SUBSYSTEM, DEVNAME, DEVTYPE `等变量，调用命令`exec`去执行`hotplug-call`脚本。

于是 `hotplug-call` 会尝试执行 `/etc/hotplug.d/block/` 目录下的所有可执行脚本。

所以我们可以在这里放置我们的自动挂载/卸载处理脚本。
例如，编写`/etc/hotplug.d/block/30-usbmount`,填入以下内容实现U盘自动挂载，卸载：

	#!/bin/sh
	 
	[ "$SUBSYSTEM" = block ] || exit0
	[ "$DEVTYPE" = partition -a"$ACTION" = add ] && {
	    echo"$DEVICENAME" | grep 'sd[a-z][1-9]' || exit 0
	    test-d /mnt/$DEVICENAME || mkdir /mnt/$DEVICENAME
	    mount  -o iocharset=utf8,rw /dev/$DEVICENAME/mnt/$DEVICENAME || \
	        mount-o rw /dev/$i /mnt/$i
	}
	 
	[ "$DEVTYPE" = partition -a"$ACTION" = remove ] && {
	    echo"$DEVICENAME" | grep 'sd[a-z][1-9]' || exit 0
	    umount/mnt/$DEVICENAME && rmdir /mnt/$DEVICENAME
	}


### Button按键的检测

在`OpenWRT`中，按键的检测也是通过`Hotplug`机制来实现的。

它首先写了一个内核模块：`gpio_button_hotplug`, 用于监听按键，有中断和 `poll` 两种方式。然后在发出事件的同时, 将记录并计算得出的两次按键时间差也作为 `uevent` 变量发出来。这样在用户空间收到这个 `uevent `事件时就知道该次按键按下了多长时间。

`hotplug.json `中有描述, 如果 `uevent` 中含有` BUTTON` 字符串, 而且` SUBSYSTEM` 为 "`button`", 则执行` /etc/rc.button/ `下的 `%BUTTON% `脚本来处理。

细节描述如下：
当按键时，则触发`button_hotplug_even`t函数（`gpio-button-hotplug.c`）

调用`button_hotplug_create_event`产生`uevent`事件，调用`button_hotplug_fill_event`填充事件(`JSON`格式)，并最终调用`button_hotplug_work`发出`uevent`广播。

上述广播，被守护进程`procd`中的`hotplug_handler (procd/plug/hotplug.c)` 收到，并根据`etc/hotplug.json`中预先定义的`JSON`内容匹配条件，定位到对应的执行函数，具体如下所示，命中了两个条目，所以会依次执行这两个条目队列中的操作函数：

	[ "if",
		[ "and",
			[ "has", "BUTTON" ],
			[ "eq", "SUBSYSTEM", "button" ],
		],
		[ "exec", "/etc/rc.button/%BUTTON%" ]
	],
	和
	[ "if",
		[ "eq", "SUBSYSTEM",
		[ "net", "input", "usb", "ieee1394", "block", "atm", "zaptel", "tty", "button" ]
		],
		[ "exec", "/sbin/hotplug-call", "%SUBSYSTEM%" ]
	],

在`rc.button`目录下，我们定义了`reset`按钮的执行脚本：

	root@OpenWrt:/etc/rc.button# cat reset
	#!/bin/sh
	
	[ "${ACTION}" = "released" ] || exit 0
	
	. /lib/functions.sh
	
	logger "$BUTTON pressed for $SEEN seconds"
	
	if [ "$SEEN" -lt 1 ]
	then
	        echo "REBOOT" > /dev/console
	        sync
	        reboot
	elif [ "$SEEN" -gt 5 ]
	then
	        echo "FACTORY RESET" > /dev/console
	        jffs2reset -y && reboot &
	fi

从脚本中我们可以清晰地看出，当按键时间小于1s时，执行`reboot`重启命令，当按键时间超过5s时，执行恢复出厂设置并重启命令。

第二个条目，由于默认情况下没有在`/etc/hotplug.d`目录下创建`button`子目录，因此执行为空。

使用 `export DBGLVL=10; procd -h /etc/hotplug.json` 截获一些打印信息看看:

	root@OpenWrt:/etc/rc.button# export DBGLVL=10; procd -h /etc/hotplug.json
	procd:hotplug_handler_debug(404): {{"HOME":"\/","PATH":"\/sbin:\/bin:\/usr\/sbin:\/usr\/bin","SUBSYSTEM":"button","ACTION":"pressed","BUTTON":"reset","SEEN":"42949450","SEQNUM":"331"}}
	procd: rule_handle_command(355): Command: exec
	procd: rule_handle_command(357):  /etc/rc.button/reset
	procd: rule_handle_command(358): 
	procd: rule_handle_command(360): Message:
	procd: rule_handle_command(362):  HOME=/
	procd: rule_handle_command(362):  PATH=/sbin:/bin:/usr/sbin:/usr/bin
	procd: rule_handle_command(362):  SUBSYSTEM=button
	procd: rule_handle_command(362):  ACTION=pressed
	procd: rule_handle_command(362):  BUTTON=reset
	procd: rule_handle_command(362):  SEEN=42949450
	procd: rule_handle_command(362):  SEQNUM=331
	procd: rule_handle_command(363): 
	procd: queue_next(281): Launched hotplug exec instance, pid=987
	procd: rule_handle_command(355): Command: exec
	procd: rule_handle_command(357):  /sbin/hotplug-call
	procd: rule_handle_command(357):  button
	procd: rule_handle_command(358): 
	procd: rule_handle_command(360): Message:
	procd: rule_handle_command(362):  HOME=/
	procd: rule_handle_command(362):  PATH=/sbin:/bin:/usr/sbin:/usr/bin
	procd: rule_handle_command(362):  SUBSYSTEM=button
	procd: rule_handle_command(362):  ACTION=pressed
	procd: rule_handle_command(362):  BUTTON=reset
	procd: rule_handle_command(362):  SEEN=42949450
	procd: rule_handle_command(362):  SEQNUM=331
	procd: rule_handle_command(363): 
	procd: queue_proc_cb(286): Finished hotplug exec instance, pid=987
	...


### 接口状态检测

当接口状态出现`ifup`或者`ifdown`时，`netifd`守护进程会调用`call_hotplug（）（/interface-event.c）`来处理这个事件，`call_hotplug()`执行`run_cmd（）`,并且设置系统环境变量`$ACTION, $INTERFACE, $DEVICE`, 同时调用`hotplug_cmd_path(=DEFAULT_HOTPLUG_PATH=/sbin/hotplug-call`, 在`netifd.h`中)并传入参数`iface`。下表是上述变量的介绍。

	---------------------------------------------------------------
	变量名称				|				说明
	---------------------------------------------------------------
	ACTION                事件，如ifup,ifdown,ifupdate
	---------------------------------------------------------------
	INTERFACE       	  发生事件动作的接口名，如（wan, ppp0）
	---------------------------------------------------------------
	DEVICE                发生事件动作的物理接口名，如（eth0.1或br-lan）
	---------------------------------------------------------------

这样用户空间脚本`hotplug-call`就会将`/etc/hotplug.d/iface`目录下的所有脚本执行一遍。

举例说明：

我们在`iface`目录下编写一个脚本名字叫`13-my-action`, 内容如下：

	root@OpenWrt:/etc/hotplug.d/iface# cat 13-my-action 
	#!/bin/sh
	
	[ "$ACTION" = ifup ] && {
	  echo Device:$DEVICE  Action:$ACTION "13-my-action" > /dev/console
	} 

让接口`down`，从下面的`log`中可以看出，`iface`下的自定义脚本被执行了一遍。

	root@OpenWrt:/etc/hotplug.d/iface# ubus call network.interface.lan up
	
	[  462.370000] IPv6: ADDRCONF(NETDEV_UP): eth1: link is not ready
	[  462.370000] device eth1 entered promiscuous mode
	[  462.380000] IPv6: ADDRCONF(NETDEV_UP): br-lan: link is not ready
	Device:br-lan Action:ifup 13-my-action
	[  462.980000] eth1: link up (1000Mbps/Full duplex)
	[  462.980000] br-lan: port 1(eth1) entered forwarding state
	[  462.990000] br-lan: port 1(eth1) entered forwarding state
	[  462.990000] IPv6: ADDRCONF(NETDEV_CHANGE): eth1: link becomes ready
	[  463.040000] IPv6: ADDRCONF(NETDEV_CHANGE): br-lan: link becomes ready
	procd: Not starting instance igmpproxy::instance1, an error was indicated
	[  464.990000] br-lan: port 1(eth1) entered forwarding state

>[备注] 由于守护进程`netifd`在`ubus`中注册了服务，因此我们可以通过`ubus`调用`netifd`提供的服务接口，例如使接口`ifdown`命令为： `ubus call network.interface.lan up/down`。

### 早期Hotplug2

早期的`Hotplug`机制，单独运行守护进程，内核会指定`hotplug2`进程来处理系统内核广播出来的`uevent`事件。原理和上面介绍的大同小异，`hotplug2`采用了与`linux`中的`udev`相同的`rule`编写规则。


**[[⬆]](#TOC)**
## <a name='references'>参考</a>

`https://openwrt.org/`
