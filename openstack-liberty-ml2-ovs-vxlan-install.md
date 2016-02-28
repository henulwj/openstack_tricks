## 基本环境配置
ubuntu 14.04 LTS
###系统安装完成后，更新系统：

	sudo apt-get update
	sudo apt-get upgrade
	sudo apt-get dist-upgrade
 
<!-- more -->

###配置网络：

主要配置IP和DNS

####IP配置 /etc/network/interfaces 

controller节点:

	auto lo
	iface lo inet loopback
	
	auto eth0
	iface eth0 inet static
	address 192.168.0.170
	netmask 255.255.255.0
	gateway 192.168.0.2
	
	auto eth1
	iface eth1 inet static
	address 10.0.0.170
	netmask 255.255.255.0


compute01节点:

	auto lo
	iface lo inet loopback
	
	auto eth0
	iface eth0 inet static
	address 192.168.0.171
	netmask 255.255.255.0
	gateway 192.168.0.2
	
	auto eth1
	iface eth1 inet static
	address 10.0.0.171
	netmask 255.255.255.0



compute02节点:

	auto lo
	iface lo inet loopback
	
	auto eth0
	iface eth0 inet static
	address 192.168.0.172
	netmask 255.255.255.0
	gateway 192.168.0.2
	
	auto eth1
	iface eth1 inet static
	address 10.0.0.172
	netmask 255.255.255.0

####DNS配置/etc/resolvconf/resolv.conf.d/base

	nameserver 202.197.64.6
	nameserver 114.114.114.114

####hosts文件配置/etc/hosts

	# controller
	192.168.0.170       controller
	
	# compute01
	192.168.0.171       compute01
	
	# compute02
	192.168.0.172       compute02

>注：注释或移除 127.0.1.1

###NTP安装

	apt-get install chrony

####controller节点配置/etc/chrony/chrony.conf 
	
	server 0.cn.pool.ntp.org iburst
	server 1.cn.pool.ntp.org iburst

####其他节点配置/etc/chrony/chrony.conf 

	server controller iburst

####重启服务
	
	service chrony restart

####验证
	
	chronyc sources

###openstack packages安装

	apt-get install software-properties-common
	add-apt-repository cloud-archive:liberty
	apt-get update && apt-get upgrade && apt-get dist-upgrade

	apt-get install python-openstackclient
	#安装完成之后重启各个节点

###数据库安装在控制节点

	apt-get install mariadb-server python-pymysql
	#配置一个合适的密码

####创建编辑/etc/mysql/conf.d/mysqld_openstack.cnf文件
	
	[mysqld]
	bind-address = 192.168.0.170
	default-storage-engine = innodb
	innodb_file_per_table
	collation-server = utf8_general_ci
	init-connect = 'SET NAMES utf8'
	character-set-server = utf8

####重启服务

	service mysql restart
	
####初始化
	
	mysql_secure_installation

###安装NOSQL数据库在控制节点

	apt-get install mongodb-server mongodb-clients python-pymongo

####配置/etc/mongodb.conf文件

	bind_ip = 192.168.0.170
	smallfiles = true

####重启完成安装

	service mongodb stop
	rm /var/lib/mongodb/journal/prealloc.*
	service mongodb start

###安装消息队列在控制节点

	apt-get install rabbitmq-server

####增加openstack用户
	
	rabbitmqctl add_user openstack RABBIT_PASS
	#赋予权限
	rabbitmqctl set_permissions openstack ".*" ".*" ".*"


##安装keystone在控制节点

###创建数据库
	mysql -u root -p
	CREATE DATABASE keystone;
	GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' \
	  IDENTIFIED BY 'KEYSTONE_DBPASS';
	GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' \
	  IDENTIFIED BY 'KEYSTONE_DBPASS';

>生成一个超级管理员token
>
> openssl rand -hex 10
> 
> f3fa43728d790762a231

###安装

	# 禁止keystone服务自动启动
	echo "manual" > /etc/init/keystone.override
	# 安装
	apt-get install keystone apache2 libapache2-mod-wsgi \
	  memcached python-memcache

###配置

####编辑	/etc/keystone/keystone.conf文件

	[DEFAULT]
	...
	verbose = True
	admin_token = f3fa43728d790762a231
	[database]
	...
	connection = mysql+pymysql://keystone:KEYSTONE_DBPASS@controller/keystone
	[memcache]
	...
	servers = localhost:11211
	[token]
	...
	provider = uuid
	driver = memcache
	[revoke]
	...
	driver = sql

###生成数据库

	su -s /bin/sh -c "keystone-manage db_sync" keystone

###配置apache http服务器

####编辑 /etc/apache2/apache2.conf 

	ServerName controller
	
####创建/etc/apache2/sites-available/wsgi-keystone.conf

	Listen 5000
	Listen 35357
	
	<VirtualHost *:5000>
	    WSGIDaemonProcess keystone-public processes=5 threads=1 user=keystone group=keystone display-name=%{GROUP}
	    WSGIProcessGroup keystone-public
	    WSGIScriptAlias / /usr/bin/keystone-wsgi-public
	    WSGIApplicationGroup %{GLOBAL}
	    WSGIPassAuthorization On
	    <IfVersion >= 2.4>
	      ErrorLogFormat "%{cu}t %M"
	    </IfVersion>
	    ErrorLog /var/log/apache2/keystone.log
	    CustomLog /var/log/apache2/keystone_access.log combined
	
	    <Directory /usr/bin>
	        <IfVersion >= 2.4>
	            Require all granted
	        </IfVersion>
	        <IfVersion < 2.4>
	            Order allow,deny
	            Allow from all
	        </IfVersion>
	    </Directory>
	</VirtualHost>
	
	<VirtualHost *:35357>
	    WSGIDaemonProcess keystone-admin processes=5 threads=1 user=keystone group=keystone display-name=%{GROUP}
	    WSGIProcessGroup keystone-admin
	    WSGIScriptAlias / /usr/bin/keystone-wsgi-admin
	    WSGIApplicationGroup %{GLOBAL}
	    WSGIPassAuthorization On
	    <IfVersion >= 2.4>
	      ErrorLogFormat "%{cu}t %M"
	    </IfVersion>
	    ErrorLog /var/log/apache2/keystone.log
	    CustomLog /var/log/apache2/keystone_access.log combined
	
	    <Directory /usr/bin>
	        <IfVersion >= 2.4>
	            Require all granted
	        </IfVersion>
	        <IfVersion < 2.4>
	            Order allow,deny
	            Allow from all
	        </IfVersion>
	    </Directory>
	</VirtualHost>

####启用

	ln -s /etc/apache2/sites-available/wsgi-keystone.conf /etc/apache2/sites-enabled
	

####重启
	
	service apache2 restart
	# 删除sqlite文件
	rm -f /var/lib/keystone/keystone.db

###创建服务端点和API
	
####配置
	
	export OS_TOKEN=f3fa43728d790762a231
	export OS_URL=http://controller:35357/v3	
	export OS_IDENTITY_API_VERSION=3

#####创建
	
	# 创建服务实体
	openstack service create \
	  --name keystone --description "OpenStack Identity" identity
	# 创建认证服务API
	openstack endpoint create --region RegionOne \
	  identity public http://controller:5000/v2.0
	openstack endpoint create --region RegionOne \
	  identity internal http://controller:5000/v2.0
	openstack endpoint create --region RegionOne \
	  identity admin http://controller:35357/v2.0

	# 创建项目 用户 角色
	# 创建admin项目
	openstack project create --domain default \
	  --description "Admin Project" admin
	# 创建admin用户
	 openstack user create --domain default \
	  --password-prompt admin
	# 创建admin、user角色
	openstack role create admin
	openstack role create user
	# 关联
	openstack role add --project admin --user admin admin
	# 创建服务项目
	openstack project create --domain default \
	  --description "Service Project" service

####验证

	#编辑/etc/keystone/keystone-paste.ini文件，从[pipeline:public_api]，[pipeline:admin_api]和[pipeline:api_v3] 移除admin_token_auth。

	unset OS_TOKEN OS_URL
	
	openstack --os-auth-url http://controller:35357/v3 \
	   --os-project-domain-id default --os-user-domain-id default \
	   --os-project-name admin --os-username admin --os-auth-type password \
	   token issue
	
####环境脚本创建

	# 编辑admin-openrc.sh,替换密码
	export OS_PROJECT_DOMAIN_ID=default
	export OS_USER_DOMAIN_ID=default
	export OS_PROJECT_NAME=admin
	export OS_TENANT_NAME=admin
	export OS_USERNAME=admin
	export OS_PASSWORD=ADMIN_PASS
	export OS_AUTH_URL=http://controller:35357/v3
	export OS_IDENTITY_API_VERSION=3


##安装glance在控制节点
	
###数据库配置
	
	mysql -u root -p
	CREATE DATABASE glance;
	GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' \
	  IDENTIFIED BY 'GLANCE_DBPASS';
	GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' \
	  IDENTIFIED BY 'GLANCE_DBPASS';

###创建服务

	source admin-openrc.sh
	openstack user create --domain default --password-prompt glance
	openstack role add --project service --user glance admin

	openstack service create --name glance \
	  --description "OpenStack Image service" image

###创建服务API

	openstack endpoint create --region RegionOne \
	  image public http://controller:9292
	openstack endpoint create --region RegionOne \
	  image internal http://controller:9292
	openstack endpoint create --region RegionOne \
	  image admin http://controller:9292

###安装配置
	
	apt-get install glance python-glanceclient

	# 编辑/etc/glance/glance-api.conf文件，替换密码
	[database]
	...
	connection = mysql+pymysql://glance:GLANCE_DBPASS@controller/glance

	[keystone_authtoken]
	...
	auth_uri = http://controller:5000
	auth_url = http://controller:35357
	auth_plugin = password
	project_domain_id = default
	user_domain_id = default
	project_name = service
	username = glance
	password = GLANCE_PASS
	
	[paste_deploy]
	...
	flavor = keystone

	[glance_store]
	...
	default_store = file
	filesystem_store_datadir = /var/lib/glance/images/

	[DEFAULT]
	...
	verbose = True
	notification_driver = noop

	# 编辑/etc/glance/glance-registry.conf文件，替换密码
	[database]
	...
	connection = mysql+pymysql://glance:GLANCE_DBPASS@controller/glance

	[keystone_authtoken]
	...
	auth_uri = http://controller:5000
	auth_url = http://controller:35357
	auth_plugin = password
	project_domain_id = default
	user_domain_id = default
	project_name = service
	username = glance
	password = GLANCE_PASS

	[paste_deploy]
	...
	flavor = keystone

	[DEFAULT]
	...
	verbose = True
	notification_driver = noop
	
	#生成数据库
	su -s /bin/sh -c "glance-manage db_sync" glance


###重启服务完成安装

	service glance-registry restart
	service glance-api restart
	rm -f /var/lib/glance/glance.sqlite

###验证

	#修改环境变量文件
	echo "export OS_IMAGE_API_VERSION=2" \
	  | tee -a admin-openrc.sh demo-openrc.sh
	# 启用
	source admin-openrc.sh	
	# 下载镜像
	wget http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img
	# 上传镜像
	glance image-create --name "cirros" \
	  --file cirros-0.3.4-x86_64-disk.img \
	  --disk-format qcow2 --container-format bare \
	  --visibility public --progress
	# 查看镜像
	glance image-list


##安装计算服务Nova

###controller节点安装配置

####数据库配置

	mysql -u root -p
	CREATE DATABASE nova;
	GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' \
	  IDENTIFIED BY 'NOVA_DBPASS';
	GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' \
	  IDENTIFIED BY 'NOVA_DBPASS';

####配置服务

	source admin-openrc.sh	
	# 创建nova用户
	openstack user create --domain default --password-prompt nova
	# 赋予角色
	openstack role add --project service --user nova admin
	# 创建服务
	openstack service create --name nova \
	  --description "OpenStack Compute" compute

####配置服务API
	
	openstack endpoint create --region RegionOne \
	  compute public http://controller:8774/v2/%\(tenant_id\)s

	openstack endpoint create --region RegionOne \
	  compute internal http://controller:8774/v2/%\(tenant_id\)s
	
	openstack endpoint create --region RegionOne \
	  compute admin http://controller:8774/v2/%\(tenant_id\)s

####安装软件配置
	
	apt-get install nova-api nova-cert nova-conductor \
	  nova-consoleauth nova-novncproxy nova-scheduler \
	  python-novaclient
	
	# 编辑/etc/nova/nova.conf文件，替换密码
	[database]
	...
	connection = mysql+pymysql://nova:NOVA_DBPASS@controller/nova
	
	[DEFAULT]
	...
	verbose = True
	rpc_backend = rabbit
	auth_strategy = keystone
	my_ip = 192.168.0.170
	network_api_class = nova.network.neutronv2.api.API
	security_group_api = neutron
	linuxnet_interface_driver = nova.network.linux_net.LinuxOVSInterfaceDriver
	firewall_driver = nova.virt.firewall.NoopFirewallDriver

	enabled_apis=osapi_compute,metadata
	
	[oslo_messaging_rabbit]
	...
	rabbit_host = controller
	rabbit_userid = openstack
	rabbit_password = RABBIT_PASS

	[keystone_authtoken]
	...
	auth_uri = http://controller:5000
	auth_url = http://controller:35357
	auth_plugin = password
	project_domain_id = default
	user_domain_id = default
	project_name = service
	username = nova
	password = NOVA_PASS

	[vnc]
	...
	vncserver_listen = $my_ip
	vncserver_proxyclient_address = $my_ip

	[glance]
	...
	host = controller

	[oslo_concurrency]
	...
	lock_path = /var/lib/nova/tmp

	# 生成数据库
	su -s /bin/sh -c "nova-manage db sync" nova
	
	# 重启服务
	service nova-api restart
	service nova-cert restart
	service nova-consoleauth restart
	service nova-scheduler restart
	service nova-conductor restart
	service nova-novncproxy restart

	# 删除sqlite文件
	rm -f /var/lib/nova/nova.sqlite



###计算节点安装配置

####安装配置
	
	apt-get install nova-compute sysfsutils
	
	#编辑/etc/nova/nova.conf文件,替换密码
	[DEFAULT]
	...
	verbose = True
	rpc_backend = rabbit
	auth_strategy = keystone
	my_ip = 192.168.0.171
	network_api_class = nova.network.neutronv2.api.API
	security_group_api = neutron
	linuxnet_interface_driver = nova.network.linux_net.LinuxOVSInterfaceDriver
	firewall_driver = nova.virt.firewall.NoopFirewallDriver
	
	
	[oslo_messaging_rabbit]
	...
	rabbit_host = controller
	rabbit_userid = openstack
	rabbit_password = RABBIT_PASS
	
	[keystone_authtoken]
	...
	auth_uri = http://controller:5000
	auth_url = http://controller:35357
	auth_plugin = password
	project_domain_id = default
	user_domain_id = default
	project_name = service
	username = nova
	password = NOVA_PASS
	
	[vnc]
	...
	enabled = True
	vncserver_listen = 0.0.0.0
	vncserver_proxyclient_address = $my_ip
	novncproxy_base_url = http://controller:6080/vnc_auto.html
	
	[glance]
	...
	host = controller
	
	[oslo_concurrency]
	...
	lock_path = /var/lib/nova/tmp

	# 查看是否支持硬件加速
	egrep -c '(vmx|svm)' /proc/cpuinfo
	#如果返回0,则配置/etc/nova/nova-compute.conf 如下，否则不用配置
	[libvirt]
	...
	virt_type = qemu
	# 重启服务
	service nova-compute restart
	# 删除sqlite文件
	rm -f /var/lib/nova/nova.sqlite

###验证

	# 在控制节点
	source admin-openrc.sh
	# 查看nova服务
	nova service-list
	# 查看服务端点
	nova endpoints
	# nova查看镜像
	nova image-list

		
##安装网络服务Neutron

###控制节点安装配置
	
####数据库配置
	
	mysql -u root -p
	CREATE DATABASE neutron;
	GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' \
	  IDENTIFIED BY 'NEUTRON_DBPASS';
	GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' \
	  IDENTIFIED BY 'NEUTRON_DBPASS';

####创建服务
	
	source admin-openrc.sh
	# 创建neutron用户
	openstack user create --domain default --password-prompt neutron
	# 关联admin角色
	openstack role add --project service --user neutron admin
	# 创建服务实体
	openstack service create --name neutron \
	  --description "OpenStack Networking" network

####创建服务API
	
	openstack endpoint create --region RegionOne \
	  network public http://controller:9696

	openstack endpoint create --region RegionOne \
	  network internal http://controller:9696
	
	openstack endpoint create --region RegionOne \
	  network admin http://controller:9696

####安装软件配置

	apt-get install neutron-server neutron-plugin-ml2 \
	  neutron-plugin-openvswitch-agent neutron-l3-agent neutron-dhcp-agent \
	  neutron-metadata-agent python-neutronclient conntrack
	# 编辑/etc/neutron/neutron.conf文件
	[database]
	...
	connection = mysql+pymysql://neutron:NEUTRON_DBPASS@controller/neutron

	[DEFAULT]
	...
	verbose = True
	core_plugin = ml2
	service_plugins = router
	allow_overlapping_ips = True
	
	rpc_backend=rabbit

	auth_strategy = keystone

	notify_nova_on_port_status_changes = True
	notify_nova_on_port_data_changes = True
	nova_url = http://controller:8774/v2

	[oslo_messaging_rabbit]
	...
	rabbit_host = controller
	rabbit_userid = openstack
	rabbit_password = RABBIT_PASS

	[keystone_authtoken]
	...
	auth_uri = http://controller:5000
	auth_url = http://controller:35357
	auth_plugin = password
	project_domain_id = default
	user_domain_id = default
	project_name = service
	username = neutron
	password = NEUTRON_PASS
	
	[nova]
	...
	auth_url = http://controller:35357
	auth_plugin = password
	project_domain_id = default
	user_domain_id = default
	region_name = RegionOne
	project_name = service
	username = nova
	password = NOVA_PASS

	# 编辑配置/etc/neutron/plugins/ml2/ml2_conf.ini文件
	[ml2]
	...
	type_drivers = flat,vlan,vxlan
	tenant_network_types = vxlan
	mechanism_drivers = openvswitch
	extension_drivers = port_security
	
	[ml2_type_flat]
	...
	flat_networks = external

	[ml2_type_vxlan]
	...
	vni_ranges = 65537:69999

	[securitygroup]
	...
	enable_security_group = True
	enable_ipset = True
	firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver

	[ovs]
	...
	local_ip = 10.0.0.170
	bridge_mappings = external:br-ex
	tunnel_bridge = br-tun
	integration_bridge = br-int
	tunnel_id_ranges = 65537:69999
	tenant_network_type = vxlan
	enable_tunneling = True
	tunnel_types = vxlan
	
	[agent]
	...
	tunnel_types = vxlan
	vxlan_udp_port = 4789
	l2_population = False

	# 编辑配置/etc/neutron/l3_agent.ini文件
	[DEFAULT]
	...
	verbose = True
	interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
	external_network_bridge =
	router_delete_namespaces = True

	# 编辑配置/etc/neutron/dhcp_agent.ini文件
	[DEFAULT]
	...
	verbose = True
	interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
	dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
	dhcp_delete_namespaces = True
	enable_isolated_metadata = True
	
	dnsmasq_config_file =  /etc/neutron/dnsmasq-neutron.conf
	
	# 创建编辑/etc/neutron/dnsmasq-neutron.conf文件
	dhcp-option-force=26,1450
	
	# 配置metadata agent
	# 编辑/etc/neutron/metadata_agent.ini文件
	[DEFAULT]
	...
	verbose = True
	auth_uri = http://controller:5000
	auth_url = http://controller:35357
	auth_region = RegionOne
	auth_plugin = password
	project_domain_id = default
	user_domain_id = default
	project_name = service
	username = neutron
	password = NEUTRON_PASS

	nova_metadata_ip = controller
	metadata_proxy_shared_secret = METADATA_SECRET
	
	# 编辑配置/etc/nova/nova.conf文件
	[neutron]
	...
	url = http://controller:9696
	auth_url = http://controller:35357
	auth_plugin = password
	project_domain_id = default
	user_domain_id = default
	region_name = RegionOne
	project_name = service
	username = neutron
	password = NEUTRON_PASS
	
	service_metadata_proxy = True
	metadata_proxy_shared_secret = METADATA_SECRET

	# 生成数据库
	su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf \
	  --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron

	

	# 重启服务
	service nova-api restart
	

	# 配置网络
	service openvswitch-switch restart
	ovs-vsctl add-br br-ex
	ovs-vsctl add-port br-ex eth0
	# 修改网络接口配置文件/etc/network/interfaces
	auto eth0
	iface eth0 inet manual
	    up ip link set dev $IFACE up
	    down ip link set dev $IFACE down
	
	auto br-ex
	iface br-ex inet static
	address 192.168.0.170
	netmask 255.255.255.0
	gateway 192.168.0.2

	# 重启服务
	service neutron-server restart
	service neutron-plugin-openvswitch-agent restart
	service neutron-dhcp-agent restart
	service neutron-metadata-agent restart
	service neutron-l3-agent restart	

	# 删除sqlite文件
	rm -f /var/lib/neutron/neutron.sqlite


###计算节点安装配置

	apt-get install neutron-plugin-ml2 neutron-plugin-openvswitch-agent conntrack
	# 编辑/etc/neutron/neutron.conf文件
	# 注释[database]中connection
	[DEFAULT]
	...
	verbose = True
	rpc_backend = rabbit
	auth_strategy = keystone

	core_plugin = ml2
	service_plugins = router
	allow_overlapping_ips = True

	[oslo_messaging_rabbit]
	...
	rabbit_host = controller
	rabbit_userid = openstack
	rabbit_password = RABBIT_PASS

	[keystone_authtoken]
	...
	auth_uri = http://controller:5000
	auth_url = http://controller:35357
	auth_plugin = password
	project_domain_id = default
	user_domain_id = default
	project_name = service
	username = neutron
	password = NEUTRON_PASS

	# 编辑/etc/neutron/plugins/ml2/ml2_conf.ini文件
	[ml2]
	type_drivers = flat,vlan,vxlan
	tenant_network_types = vxlan
	mechanism_drivers = openvswitch
	extension_drivers = port_security
	

	[ml2_type_vxlan]
	...
	vni_ranges = 65537:69999

	[securitygroup]
	...
	enable_security_group = True
	enable_ipset = True
	firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver

	[ovs]
	...
	local_ip = 10.0.0.171
	tunnel_type = vxlan
	tunnel_bridge = br-tun
	integration_bridge = br-int
	tunnel_id_ranges = 65537:69999
	tenant_network_type = vxlan
	enable_tunneling = True
	tunnel_types = vxlan
	
	[agent]
	...
	tunnel_types = vxlan
	vxlan_udp_port = 4789
	l2_population = False

	# 编辑/etc/nova/nova.conf文件
	[neutron]
	...
	url = http://controller:9696
	auth_url = http://controller:35357
	auth_plugin = password
	project_domain_id = default
	user_domain_id = default
	region_name = RegionOne
	project_name = service
	username = neutron
	password = NEUTRON_PASS

	#重启服务
	service nova-compute restart
	service neutron-plugin-openvswitch-agent restart

###验证
	
	# 在控制节点
	source admin-openrc.sh
	# 验证neutron-server启动
	neutron ext-list
	# 验证neutron agents的启动
	neutron agent-list
	


##安装horizon在控制节点

	apt-get install openstack-dashboard

	# 编辑配置文件/etc/openstack-dashboard/local_settings.py文件
	OPENSTACK_HOST = "controller"
	ALLOWED_HOSTS = ['*', ]
	CACHES = {
	    'default': {
	         'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
	         'LOCATION': '127.0.0.1:11211',
	    }
	}
	
	OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"
	
	# 使用keystone v3认证
	OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True
	OPENSTACK_API_VERSIONS = {
	    "identity": 3,
	    "volume": 2,
	}

	# 时区 修改TIME_ZONE
	TIME_ZONE = "TIME_ZONE"

	# 重启apache服务
	service apache2 reload

	# 移除ubuntu主题包
	apt-get remove -y --purge openstack-dashboard-ubuntu-theme

	# 验证
	# 使用admin或其他用户名登陆http://controller/horizon
	

##安装Ceilometer服务

###在控制节点
	
	#创建ceilometer数据库
	mongo --host controller --eval '
	  db = db.getSiblingDB("ceilometer");
	  db.addUser({user: "ceilometer",
	  pwd: "CEILOMETER_DBPASS",
	  roles: [ "readWrite", "dbAdmin" ]})'
	
	source admin-openrc.sh

	# 创建ceilometer服务
	openstack user create --domain default --password-prompt ceilometer
	openstack role add --project service --user ceilometer admin
	openstack service create --name ceilometer \
	  --description "Telemetry" metering

	# 创建ceilometer服务API
	openstack endpoint create --region RegionOne \
	  metering public http://controller:8777
	
	openstack endpoint create --region RegionOne \
	  metering internal http://controller:8777

	openstack endpoint create --region RegionOne \
	  metering admin http://controller:8777

	# 安装软件
	apt-get install ceilometer-api ceilometer-collector \
	  ceilometer-agent-central ceilometer-agent-notification \
	  ceilometer-alarm-evaluator ceilometer-alarm-notifier \
	  python-ceilometerclient

	# 编辑/etc/ceilometer/ceilometer.conf文件
	
	[database]
	...
	connection = mongodb://ceilometer:CEILOMETER_DBPASS@controller:27017/ceilometer

	[DEFAULT]
	...
	verbose = True
	rpc_backend = rabbit
	auth_strategy = keystone
	
	[oslo_messaging_rabbit]
	...
	rabbit_host = controller
	rabbit_userid = openstack
	rabbit_password = RABBIT_PASS	
	
	[keystone_authtoken]
	...
	auth_uri = http://controller:5000
	auth_url = http://controller:35357
	auth_plugin = password
	project_domain_id = default
	user_domain_id = default
	project_name = service
	username = ceilometer
	password = CEILOMETER_PASS

	[service_credentials]
	...
	os_auth_url = http://controller:5000/v2.0
	os_username = ceilometer
	os_tenant_name = service
	os_password = CEILOMETER_PASS
	os_endpoint_type = internalURL
	os_region_name = RegionOne

	# 重启服务
	service ceilometer-agent-central restart
	service ceilometer-agent-notification restart
	service ceilometer-api restart
	service ceilometer-collector restart
	service ceilometer-alarm-evaluator restart
	service ceilometer-alarm-notifier restart

	# 启用镜像服务的meter
	# 编辑/etc/glance/glance-api.conf和/etc/glance/glance-registry.conf文件
	[DEFAULT]
	...
	notification_driver = messagingv2
	rpc_backend = rabbit
	
	[oslo_messaging_rabbit]
	...
	rabbit_host = controller
	rabbit_userid = openstack
	rabbit_password = RABBIT_PASS

	# 重启服务
	service glance-registry restart
	service glance-api restart


###在计算节点

	# 启用计算服务的meter
	apt-get install ceilometer-agent-compute

	# 编辑/etc/ceilometer/ceilometer.conf文件
	[DEFAULT]
	...
	verbose = True
	rpc_backend = rabbit
	auth_strategy = keystone
	
	[oslo_messaging_rabbit]
	...
	rabbit_host = controller
	rabbit_userid = openstack
	rabbit_password = RABBIT_PASS

	[keystone_authtoken]
	...
	auth_uri = http://controller:5000
	auth_url = http://controller:35357
	auth_plugin = password
	project_domain_id = default
	user_domain_id = default
	project_name = service
	username = ceilometer
	password = CEILOMETER_PASS

	[service_credentials]
	...
	os_auth_url = http://controller:5000/v2.0
	os_username = ceilometer
	os_tenant_name = service
	os_password = CEILOMETER_PASS
	os_endpoint_type = internalURL
	os_region_name = RegionOne
	
	# 编辑/etc/nova/nova.conf文件
	[DEFAULT]
	...
	instance_usage_audit = True
	instance_usage_audit_period = hour
	notify_on_state_change = vm_and_task_state
	notification_driver = messagingv2

	# 重启服务
	service ceilometer-agent-compute restart
	service nova-compute restart

###验证

	source admin-openrc.sh
	ceilometer meter-list

	# 下载镜像
	IMAGE_ID=$(glance image-list | grep 'cirros' | awk '{ print $2 }')
	glance image-download $IMAGE_ID > /tmp/cirros.img
	
	ceilometer meter-list

	ceilometer statistics -m image.download -p 60

	rm /tmp/cirros.img


##启用FWaaS

	# 在控制节点编辑/etc/neutron/neutron.conf文件
	[DEFAULT]
	service_plugins = firewall
	
	[service_providers]
	...
	service_provider = FIREWALL:Iptables:neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver:default

	# 编辑/etc/neutron/fwaas_driver.ini文件
	[fwaas]
	driver = ：neutron_fwaas.services.firewall.drivers.linux.iptables_fwaas.IptablesFwaasDriver
	enabled = True

	# 同步数据库
	neutron-db-manage --service fwaas upgrade head
	
	# horizon面板启用，编辑/etc/openstack_dashboard/local_settings.py文件
	OPENSTACK_NEUTRON_NETWORK = {
	    ...
	    'enable_firewall' = True,
	    ...
	}
	
	# 重启服务
	service neutron-l3-agent restart
	service neutron-server restart


	
	