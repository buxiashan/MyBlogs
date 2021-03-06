.. _storage_intro:


########################
ceph与分布式存储入门
########################


..
    标题 ####################
    一号 ====================
    二号 ++++++++++++++++++++
    三号 --------------------
    四号 ^^^^^^^^^^^^^^^^^^^^



.. contents:: 目录

--------------------------

最近在学习Ceph存储和运维相关知识，可是在参考ceph官方文档时，发现一堆术语名词看的云里雾里。
比如"Ceph 独一无二地用统一的系统提供了对象、块、和文件存储功能"，那么又什么是块存储、对象存储、文件
存储呢？于是在网上找了很多资料进行学习总结，如有不当之处，请指正。


对象存储
========

概念
+++++

对象存储的概念并没有一个准确的定义，可以简单的认为它是 **分布式的key/value存储** .

通过查找相关文献，发现对对象存储阐述的比较好的有以下观点：

- 	之所以出现了对象存储这种东西，是为了克服块存储与文件存储各自的缺点，
	发扬它俩各自的优点。简单来说块存储读写快，不利于共享，文件存储读写慢，
	利于共享。能否弄一个读写快，利 于共享的出来呢。于是就有了对象存储。

	首先，一个文件包含了了属性（术语叫metadata，元数据，
	例如该文件的大小、修改时间、存储路径等）以及内容（以下简称数据）。
	以往像FAT32这种文件系统，是直接将一份文件的数据与metadata
	一起存储的，存储过程先将文件按照文件系统的最小块大小来打散
	（如4M的文件，假设文件系统要求一个块4K，那么就将文件打散成
	为1000个小块），再写进硬盘里面，过程中没有区分数据/metadata
	的。而每个块最后会告知你下一个要读取的块的地址，然后一直这样
	顺序地按图索骥，最后完成整份文件的所有块的读取。


	这种情况下读写速率很慢，因为就算你有100个机械手臂在读写，
	但是由于你只有读取到第一个块，才能知道下一个块在哪里，
	其实相当于只能有1个机械手臂在实际工作。


	而对象存储则将元数据独立了出来，控制节点叫元数据服务器（服务器+对象存储管理软件），
	里面主要负责存储对象的属性（主要是对象的数据被打散存放到了那几台分布式服务器中的信息），
	而其他负责存储数据的分布式服务器叫做OSD，主要负责存储文件的数据部分。
	当用户访问对象，会先访问元数据服务器，元数据服务器只负责反馈对象存储在哪些OSD，
	假设反馈文件A存储在B、C、D三台OSD，那么用户就会再次直接访问3台OSD服务器去读取数据。
	这时候由于是3台OSD同时对外传输数据，所以传输的速度就加快了。当OSD服务器数量越多，
	这种读写速度的提升就越大，通过此种方式，实现了读写快的目的。


	另一方面，对象存储软件是有专门的文件系统的，所以OSD对外又相
	当于文件服务器，那么就不存在文件共享方面的困难了，也解决了文件共享方面的问题。
	所以对象存储的出现，很好地结合了块存储与文件存储的优点。
	
	.. [#] https://www.zhihu.com/question/21536660
	
-	有云公司存储工程师认为：

	分布式存储的应用场景相对于其存储接口，现在流行分为三种:

	*	对象存储: 也就是通常意义的键值存储，其接口就是简单的GET、PUT、DEL和其他扩展，如七牛、又拍、Swift、S3

	*	块存储: 这种接口通常以QEMU Driver或者Kernel Module的方式存在，这种接口需要实现Linux的Block Device的接口或者QEMU提供的Block Driver接口，如Sheepdog，AWS的EBS，青云的云硬盘和阿里云的盘古系统，还有Ceph的RBD（RBD是Ceph面向块存储的接口）

	*	文件存储: 通常意义是支持POSIX接口，它跟传统的文件系统如Ext4是一个类型的，但区别在于分布式存储提供了并行化的能力，如Ceph的CephFS(CephFS是Ceph面向文件存储的接口)，但是有时候又会把GFS，HDFS这种非POSIX接口的类文件存储接口归入此类。
	
	

	按照这三种接口和其应用场景，很容易了解这三种类型的IO特点，括号里代表了它在非分布式情况下的对应：

	*	对象存储（键值数据库）： 接口简单，一个对象我们可以看成一个文件，只能全写全读，通常以大文件为主，要求足够的IO带宽。

	*	块存储（硬盘）：它的IO特点与传统的硬盘是一致的，一个硬盘应该是能面向通用需求的，即能应付大文件读写，也能处理好小文件读写。但是硬盘的特点是容量大，热点明显。因此块存储主要可以应付热点问题。另外，块存储要求的延迟是最低的。

	*	文件存储（文件系统）：支持文件存储的接口的系统设计跟传统本地文件系统如Ext4这种的特点和难点是一致的，它比块存储具有更丰富的接口，需要考虑目录、文件属性等支持，实现一个支持并行化的文件存储应该是最困难的。但像HDFS、GFS这种自己定义标准的系统，可以通过根据实现来定义接口，会容易一点。
	
	.. [#] http://www.infoq.com/cn/articles/virtual-forum-three-basic-issues-about-distributed-storage
	
-	类似标准化组织SNIA定义:

	OSD（object storage device）和MDS（Metadata Server）为基本组成部分的分布式存储，通常是分布式文件系统。我们经常听到的分布式存储Ceph的底层RADOS（Reliable Autonomous Distributed Object Store），即属于这类对象存储。
	
	.. [#] http://www.testlab.com.cn/Index/article/id/1082.html
	.. [#] http://limu713.blog.163.com/blog/static/15086904201222024847744/
	

特点
++++++

下面从接口，结构等方面概述对象存储的特点。

接口
----------------

对于大多数文件系统来说，尤其是POSIX兼容的文件系统，提供open、close、read、write和lseek等接口。

而对象存储的接口是REST风格的，通常是基于HTTP协议的RESTful Web API，通过HTTP请求中的PUT和GET等操作进行文件的上传即写入和下载即读取，通过DELETE操作删除文件。

对象存储和文件系统在接口上的本质区别是对象存储不支持和fread和fwrite类似的随机位置读写操作，即一个文件PUT到对象存储里以后，如果要读取，只能GET整个文件，如果要修改一个对象，只能重新PUT一个新的到对象存储里，覆盖之前的对象或者形成一个新的版本。

如果结合平时使用云盘的经验，就不难理解这个特点了，用户会上传文件到云盘或者从云盘下载文件。如果要修改一个文件，会把文件下载下来，修改以后重新上传，替换之前的版本。实际上几乎所有的互联网应用，都是用这种存储方式读写数据的，比如微信，在朋友圈里发照片是上传图像、收取别人发的照片是下载图像，也可以从朋友圈中删除以前发送的内容；微博也是如此，通过微博API我们可以了解到，微博客户端的每一张图片都是通过REST风格的HTTP请求从服务端获取的，而我们要发微博的话，也是通过HTTP请求将数据包括图片传上去的。在没有对象存储以前，开发者需要自己为客户端提供HTTP的数据读写接口，并通过程序代码转换为对文件系统的读写操作。

扁平的数据组织结构
-------------------

对比文件系统，对象存储的第二个特点是没有嵌套的文件夹，而是采用扁平的数据组织结构，往往是两层或者三层，例如AWS S3和华为的UDS，每个用户可以把它的存储空间划分为“容器”（Bucket），然后往每个容器里放对象，对象不能直接放到租户的根存储空间里，必须放到某个容器下面，而不能嵌套，也就是说，容器下面不能再放一层容器，只能放对象。OpenStack Swift也类似

这就是所谓“扁平数据组织结构”，因为它和文件夹可以一级一级嵌套不同，层次关系是固定的，而且只有两到三级，是扁平的。每一级的每个元素，例如S3中的某个容器或者某个对象，在系统中都有唯一的标识，用户通过这个标识来访问容器或者对象，所以，对象存储提供的是一种K/V的访问方式。

.. [#] http://www.testlab.com.cn/Index/article/id/1082.html
.. [#] http://limu713.blog.163.com/blog/static/15086904201222024847744/

.. tip::

	在学习ceph时，有一个pool的概念，可以认为pool就是划分存储空间的容器。


对象存储系统组成
+++++++++++++++++++++

对象存储结构组成部分包括：对象、对象存储设备、元数据服务器、对象存储系统的客户端。

对象
----------

对象是系统中数据存储的基本单位，一个对象实际上就是文件的数据和一组属性信息（Meta Data）的组合，这些属性信息可以定义基于文件的RAID参数、数据分布和服务质量等，而传统的存储系统中用文件或块作为基本的存储单位，在块存储系统中还需要始终追踪系统中每个块的属性，对象通过与存储系统通信维护自己的属性。在存储设备中，所有对象都有一个对象标识，通过对象标识OSD命令访问该对象。通常有多种类型的对象，存储设备上的根对象标识存储设备和该设备的各种属性，组对象是存储设备上共享资源管理策略的对象集合等。 

对象存储设备
-----------------

对象存储设备具有一定的智能，它有自己的CPU、内存、网络和磁盘系统，OSD同块设备的不同不在于存储介质，而在于两者提供的访问接口。OSD的主要功能包括数据存储和安全访问。
 
 
元数据服务器（Metadata Server，MDS）
---------------------------------------
 
MDS控制Client与OSD对象的交互。
 
 
对象存储系统的客户端Client
---------------------------

为了有效支持Client支持访问OSD上的对象，需要在计算节点实现对象存储系统的Client，通常提供POSIX文件系统接口，允许应用程序像执行标准的文件系统操作一样。

.. [#] http://limu713.blog.163.com/blog/static/15086904201222024847744/


OpenStack利用ceph备份还原虚拟机
===============================

-	首先启动一台虚拟机，利用ceph作为后端存储！

-	获取该虚拟机对应ceph中的volume！有两种方式，一种方式直接在dashboard中查看虚拟机详情。

	.. figure:: /_static/images/instance_volume.png
	   :scale: 100
	   :align: center

	   获取虚拟机ceph卷
	   
	另外一种方式可以使用命令行：
	
	::
	
		nova show instance
		
-	查找ceph中对应的卷。::
	
		rbd ls volumes | grep 5b4df728-1968-4b59-b5c8-e6ce6198e007

	.. figure:: /_static/images/find_volume.png
	   :scale: 100
	   :align: center

	   获取虚拟机ceph卷

-	导出卷。::

		rbd export -p volumes volume-5b4df728-1968-4b59-b5c8-e6ce6198e007 cirros-volume.bk

	.. figure:: /_static/images/export_volume.png
	   :scale: 100
	   :align: center

	   ceph volume导出
	   
-	关闭该虚拟机，并删除原ceph volume。::

		nova stop instance
		rbd rm -p volumes volume-5b4df728-1968-4b59-b5c8-e6ce6198e007
		
-	导入备份ceph volume::

		rbd import -p  volumes cirros-volume.bk volume-5b4df728-1968-4b59-b5c8-e6ce6198e007

	注意：导入备份ceph volume 需要删除原来的镜像，否则会提示错误!

	.. figure:: /_static/images/import_volume_file_exist.png
	   :scale: 100
	   :align: center

	   删除原volume才可以导入备份volume
	
	然后尝试在dashboard重新打开虚拟机，成功！

-	另外需要注意的是：我们通过 ``rbd export`` 导出的ceph volume镜像，可以通过kvm启动。

	.. figure:: /_static/images/kvm_start_ceph_volume.png
	   :scale: 100
	   :align: center
	   
	.. figure:: /_static/images/kvm_start_ceph_volume2.png
	   :scale: 100
	   :align: center
	   
	   kvm启动导出的备份成功！

.. [#] http://www.tuicool.com/articles/ee26FnI

---------------------

参考
=====


