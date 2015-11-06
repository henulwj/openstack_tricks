#Glance管理的镜像

如果采用local storage，glance将镜像文件默认存储到/var/lib/glance/image目录下

	root@controller:/var/lib/glance/images# ll -h
	total 259M
	drwxr-xr-x 2 glance glance 4.0K Aug  3 17:13 ./
	drwxr-xr-x 4 glance glance 4.0K Jul 30 10:54 ../
	-rw-r----- 1 glance glance 247M Aug  3 17:14 7dfa159a-7e13-4668-985b-f94106c7bd30
	-rw-r----- 1 glance glance  13M Jul 30 11:08 bd1cfdf7-e306-4ffb-86bf-d365875691a2

通过qemu-img info命令，查看一下镜像文件的大小和格式

	root@controller:/var/lib/glance/images# qemu-img info bd1cfdf7-e306-4ffb-86bf-d365875691a2 
	image: bd1cfdf7-e306-4ffb-86bf-d365875691a2
	file format: qcow2
	virtual size: 39M (41126400 bytes)
	disk size: 13M
	cluster_size: 65536
	Format specific information:
	    compat: 0.10

创建的虚拟机存放在/var/lib/nova/instances目录下，该目录的大体结构如下：

	root@compute2:/var/lib/nova/instances# ll -h
	total 36K
	drwxr-xr-x 8 nova nova 4.0K Nov  3 14:38 ./
	drwxr-xr-x 9 nova nova 4.0K Aug  2 11:45 ../
	drwxr-xr-x 2 nova nova 4.0K Oct 23 15:56 432695fd-d696-498a-a1b5-db105c101982/
	drwxr-xr-x 2 nova nova 4.0K Oct  5 22:33 67046d40-5878-43e4-ae2c-5018d7532702/
	drwxr-xr-x 2 nova nova 4.0K Oct  5 17:07 6751ac80-14c9-4096-8743-5e4cbf41a8b9/
	drwxr-xr-x 2 nova nova 4.0K Oct 23 15:50 ac61b296-308f-4825-81e7-e97fb4521cb4/
	drwxr-xr-x 2 nova nova 4.0K Aug 25 08:53 _base/   //相当于镜像文件的cache目录，在此host上创建的所有的vm，都会先cache到这里
	-rw-r--r-- 1 nova nova   31 Nov  4 16:13 compute_nodes
	drwxr-xr-x 2 nova nova 4.0K Aug  3 15:50 locks/

(1)从vm的描述文件中获得所使用的image文件的ID，然后向Glance发起索取image文件的HTTP请求，结果是image文件从Glance的存储节点下载到发起请求的host机器上，即：/var/lib/nova/instances/_base目录下

(2)镜像下载成功后，openstack先去判断image文件的类型是否为qcow2，如果是，则现将其转化为raw格式，否则，直接进入（3）


	root@compute2:/var/lib/nova/instances/432695fd-d696-498a-a1b5-db105c101982# ll
	total 1708
	drwxr-xr-x 2 nova         nova    4096 Oct 23 15:56 ./
	drwxr-xr-x 8 nova         nova    4096 Nov  3 14:38 ../
	-rw-rw---- 1 nova         kvm    19288 Nov  3 14:38 console.log
	-rw-r--r-- 1 libvirt-qemu kvm  1769472 Nov  4 02:39 disk
	-rw-r--r-- 1 nova         nova      79 Oct 23 15:56 disk.info
	-rw-r--r-- 1 nova         nova    2577 Nov  3 14:38 libvirt.xml
	root@compute2:/var/lib/nova/instances/432695fd-d696-498a-a1b5-db105c101982# qemu-img info disk
	image: disk
	file format: qcow2
	virtual size: 1.0G (1073741824 bytes)
	disk size: 1.6M
	cluster_size: 65536
	backing file: /var/lib/nova/instances/_base/e541536d92037d8c96d03bb8c1cb9f6ffa0233eb
	Format specific information:
	    compat: 1.1
	    lazy refcounts: false


	root@compute2:/var/lib/nova/instances/432695fd-d696-498a-a1b5-db105c101982# qemu-img info disk.info 
	image: disk.info
	file format: raw
	virtual size: 512 (512 bytes)
	disk size: 4.0K



参考：

1.http://www.openstack.cn/?p=358

2.http://blog.chinaunix.net/uid-20940095-id-3504622.html