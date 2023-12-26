# ubuntu-22.04.3LTS arm64

- [x] Keystone-身份验证服务
- [x] Horizon-仪表盘
- [x] Nova控制节点
  - [x] Nova-控制节点
  - [x] Nova-计算节点
- [x] Placement-安置服务
- [x] Cinder-块存储服务
- [x] Glance-镜像服务
- [ ] Neutron-网络服务

# 控制节点

### 虚拟机配置

### 网卡

修改配置文件`/etc/netplan/00-installer-config.yaml`:

```yaml
dhcp4: no
  addresses: [192.168.0.11/24]
  gateway4: 192.168.0.1
  nameservers:
    addresses: [8.8.8.8, 8.8.4.4]
```

```bash
netplan apply
```

在`/etc/hosts`中声明：

```yaml
192.168.0.11 controller
```

## 数据库

### 安装和配置组件

1. 安装软件包：

   ```bash
   apt install mariadb-server python3-pymysql
   ```

2. 创建并编辑 `/etc/mysql/mariadb.conf.d/99-openstack.cnf` 文件并完成以下操作：

   创建 `[mysqld]` 部分，并将 `bind-address` 键设置为控制节点的管理IP地址，以便其他节点通过管理网络访问。设置附加键以启用有用的选项和 UTF-8 字符集：

   ```py
   [mysqld]
   bind-address = 192.168.0.11
   
   default-storage-engine = innodb
   innodb_file_per_table = on
   max_connections = 4096
   collation-server = utf8_general_ci
   character-set-server = utf8
   ```

### 完成安装

1. 重启数据库服务：

   ```bash
   service mysql restart
   ```

2. 通过运行 `mysql_secure_installation` 脚本来保护数据库服务。特别是，为数据库 `root` 帐户选择合适的密码：

   ```bash
   mysql_secure_installation
   ```

## 消息队列

### 安装和配置组件

1. 安装包：

   ```bash
   apt install rabbitmq-server
   ```

2. 添加 `openstack` 用户：

   ```bash
   rabbitmqctl add_user openstack RABBIT_PASS
   ```

3. 允许 `openstack` 用户进行配置、写入和读取访问：

   ```bash
   rabbitmqctl set_permissions openstack ".*" ".*" ".*"
   ```

## 内存缓存

### 安装和配置组件

1. 安装软件包：

   ```bash
   apt install memcached python3-memcache
   ```

2. 编辑 `/etc/memcached.conf` 文件并将服务配置为使用控制器节点的管理 IP 地址。这是为了允许其他节点通过管理网络进行访问：

   ```py
   -l 192.168.0.11
   ```

### 完成安装

1. 重启Memcached服务：

   ```bash
   service memcached restart
   ```

## Etcd

1. 安装 `etcd` 包：

   ```bash
   apt install etcd
   ```

2. 编辑 `/etc/default/etcd` 文件，将 `ETCD_INITIAL_CLUSTER` 、 `ETCD_INITIAL_ADVERTISE_PEER_URLS` 、 `ETCD_ADVERTISE_CLIENT_URLS` 、 `ETCD_LISTEN_CLIENT_URLS` 设置为管理 IP 地址控制器节点允许其他节点通过管理网络进行访问：

   ```yaml
   ETCD_NAME="controller"
   ETCD_DATA_DIR="/var/lib/etcd"
   ETCD_INITIAL_CLUSTER_STATE="new"
   ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster-01"
   ETCD_INITIAL_CLUSTER="controller=http://192.168.0.11:2380"
   ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.0.11:2380"
   ETCD_ADVERTISE_CLIENT_URLS="http://192.168.0.11:2379"
   ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
   ETCD_LISTEN_CLIENT_URLS="http://192.168.0.11:2379"
   ```

3. 启用并重启etcd服务：

   ```bash
   systemctl enable etcd
   systemctl restart etcd
   ```

## 安装Openstack命令行客户端

### 安装软件

```bash
apt install python2.7
apt install python2-dev python-pip
apt install --reinstall python3-pip // 如果pip不生效
```

### 安装 OpenStack 客户端

```bash
pip install python-openstackclient
openstack --version
```

### 创建并获取 OpenStack RC 文件

1. 在文本编辑器中，创建一个名为 `admin-openrc.sh` 的文件并添加以下身份验证信息：

   ```bash
   export OS_PROJECT_DOMAIN_NAME=Default
   export OS_USER_DOMAIN_NAME=Default
   export OS_PROJECT_NAME=admin
   export OS_USERNAME=admin
   export OS_PASSWORD=705432
   export OS_AUTH_URL=http://192.168.0.11:5000/v3
   export OS_IDENTITY_API_VERSION=3
   export OS_IMAGE_API_VERSION=2
   ```

   ```bash
   source admin-openrc.sh
   ```

## 部署KeyStone

1. 使用数据库访问客户端以 `root` 用户连接数据库服务器:

   ```sql
   CREATE DATABASE keystone;
   ```

2. 授予对 `keystone` 数据库的正确访问权限：

   ```sql
   GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY '705432';
   GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY '705432';
   ```

3. 运行以下命令来安装软件包：

   ```bash
   apt install keystone
   ```

4. 编辑 `/etc/keystone/keystone.conf` 文件并完成以下操作：

   在 `[database]` 部分中，配置数据库访问：

   ```py
   [database]
   # ...
   connection = mysql+pymysql://keystone:705432@controller/keystone
   ```

   在 `[token]` 部分中，配置 Fernet 令牌提供程序：

   ```py
   [token]
   # ...
   provider = fernet
   ```

5. 填充身份服务数据库：

   ```bash
   su -s /bin/sh -c "keystone-manage db_sync" keystone
   ```

6. 初始化 Fernet 密钥存储库：

   ```bash
   keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
   keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
   ```

7. 引导身份服务：

   ```bash
   keystone-manage bootstrap --bootstrap-password 705432 \
   --bootstrap-admin-url http://192.168.0.11:5000/v3/ \
   --bootstrap-internal-url http://192.168.0.11:5000/v3/ \
   --bootstrap-public-url http://192.168.0.11:5000/v3/ \
   --bootstrap-region-id RegionOne
   ```



### Apache HTTP 服务器

1. 编辑 `/etc/apache2/apache2.conf` 文件并配置 `ServerName` 选项以引用控制器节点：

   ```yaml
   ServerName 192.168.0.11
   ```

2. 重新启动 Apache 服务：

   ```bash
   service apache2 restart
   ```

3. 通过设置适当的环境变量来配置管理帐户：

   ```bash
   source /admin-openrc.sh
   ```

### 验证Openstack服务

```bash
openstack project list
+----------------------------------+-------+
| ID                               | Name  |
+----------------------------------+-------+
| ae602a54a2f74c39977275eda5d4ef47 | admin |
+----------------------------------+-------+
```

### 创建域、项目、用户和角色

1. 尽管本指南中的 keystone-manage 引导步骤中已经存在“默认”域，但创建新域的正式方法是：

   ```bash
   openstack domain create --description "An Example Domain" example
   ```

2. 本指南使用的服务项目包含您添加到环境中的每项服务的唯一用户。创建 `service` 项目：

   ```bash
   openstack project create --domain default --description "Service Project" service
   ```

3. 常规（非管理）任务应使用非特权项目和用户。例如，本指南创建 `myproject` 项目和 `myuser` 用户：

   ```bash
   openstack project create --domain default --description "Demo Project" myproject
   openstack user create --domain default --password-prompt myuser
   ```

4. 将 `myrole` 角色添加到 `myproject` 项目和 `myuser` 用户：

   ```bash
   openstack role create myrole
   openstack role add --project myproject --user myuser myrole
   ```

### 验证

1. 取消设置临时 `OS_AUTH_URL` 和 `OS_PASSWORD` 环境变量：

   ```bash
   unset OS_AUTH_URL OS_PASSWORD
   ```

2. 作为 `admin` 用户，请求身份验证令牌：

   ```bash
   openstack --os-auth-url http://192.168.0.11:5000/v3 \
   --os-project-domain-name Default --os-user-domain-name Default \
   --os-project-name admin --os-username admin token issue
   ```

3. 作为之前创建的 `myuser` 用户，请求身份验证令牌：

   ```bash
   openstack --os-auth-url http://192.168.0.11:5000/v3 \
   --os-project-domain-name Default --os-user-domain-name Default \
   --os-project-name myproject --os-username myuser token issue
   ```

## 部署Horizon

### 安装和配置组件

1. 安装软件包：

   ```bash
   apt install openstack-dashboard
   ```

2. 编辑 `/etc/openstack-dashboard/local_settings.py` 文件并完成以下操作：

   配置仪表板以使用 `controller` 节点上的 OpenStack 服务：

   ```py
   OPENSTACK_HOST = "192.168.0.11"
   ```

   在仪表板配置部分中，允许所有主机访问仪表板：

   ```py
   ALLOWED_HOSTS = ['*']
   ```

   配置 `memcached` 会话存储服务：

   ```py
   SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
   
   CACHES = {
       'default': {
            'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
            'LOCATION': '192.168.0.11:11211',
       }
   }
   ```

   启用身份 API 版本 3：

   ```py
   OPENSTACK_KEYSTONE_URL = "http://%s:5000/identity/v3" % OPENSTACK_HOST
   ```

   启用对域的支持：

   ```py
   OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True
   ```

   配置API版本：

   ```py
   OPENSTACK_API_VERSIONS = {
       "identity": 3,
       "image": 2,
       "volume": 3,
   }
   ```

   将 `Default` 配置为您通过仪表板创建的用户的默认域：

   ```py
   OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = "Default"
   ```

   将 `user` 配置为您通过仪表板创建的用户的默认角色：

   ```python
   OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"
   ```

   如果您选择网络选项 1，请禁用对第 3 层网络服务的支持：

   ```py
   OPENSTACK_NEUTRON_NETWORK = {
       ...
       'enable_router': False,
       'enable_quotas': False,
       'enable_ipv6': False,
       'enable_distributed_router': False,
       'enable_ha_router': False,
       'enable_fip_topology_check': False,
   }
   ```

3. 如果未包含以下行，请将其添加到 `/etc/apache2/conf-available/openstack-dashboard.conf` 中：

   ```python
   WSGIApplicationGroup %{GLOBAL}
   ```

### 完成安装

重新加载 Web 服务器配置：

```bash
systemctl reload apache2.service
```

### 访问Web端

http://192.168.0.11/horizon

## 控制节点部署Nova

1. 要创建数据库，请完成以下步骤：

   使用数据库访问客户端以 `root` 用户连接数据库服务器：

   ```bash
   mysql -u root -p
   ```

   创建 `nova_api` 、 `nova` 和 `nova_cell0` 数据库：

   ```sql
   CREATE DATABASE nova_api;
   CREATE DATABASE nova;
   CREATE DATABASE nova_cell0;
   ```

   授予对数据库的正确访问权限：

   ```sql
   GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' IDENTIFIED BY '705432';
   GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' IDENTIFIED BY '705432';
   
   GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' IDENTIFIED BY '705432';
   GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY '705432';
   
   GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost' IDENTIFIED BY '705432';
   GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' IDENTIFIED BY '705432';
   ```

2. 获取 `admin` 凭据以访问仅限管理员的 CLI 命令：

   ```bash
   source /admin-openrc.sh
   ```

3. 创建计算服务凭证：

   创建 `nova` 用户：

   ```bash
   openstack user create --domain default --password-prompt nova
   ```

   将 `admin` 角色添加到 `nova` 用户：

   ```bash
   openstack role add --project service --user nova admin
   ```

   创建 `nova` 服务实体：

   ```bash
   openstack service create --name nova --description "OpenStack Compute" compute
   ```

4. 创建计算 API 服务端点：

   ```bash
   openstack endpoint create --region RegionOne compute public http://192.168.0.11:8774/v2.1
   
   openstack endpoint create --region RegionOne compute internal http://192.168.0.11:8774/v2.1
   
   openstack endpoint create --region RegionOne compute admin http://192.168.0.11:8774/v2.1
   ```

### 安装 Placement 服务

1. 使用数据库访问客户端以 `root` 用户连接数据库服务器：

   ```bash
   mysql
   ```

2. 创建 `placement` 数据库：

   ```sql
   CREATE DATABASE placement;
   ```

3. 授予对数据库的正确访问权限：

   ```sql
   GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'localhost' IDENTIFIED BY '705432';
   GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'%' IDENTIFIED BY '705432';
   ```

4. 获取 `admin` 凭据以访问仅限管理员的 CLI 命令：

   ```bash
   source /admin-openrc.sh
   ```

5. 创建展示位置服务用户：

   ```bash
   openstack user create --domain default --password-prompt placement
   ```

6. 将 Placement 用户添加到具有 admin 角色的服务项目：

   ```bash
   openstack role add --project service --user placement admin
   ```

7. 在服务目录中创建 Placement API 条目：

   ```bash
   openstack service create --name placement --description "Placement API" placement
   ```

8. 创建 Placement API 服务端点:

   ```bash
   openstack endpoint create --region RegionOne placement public http://192.168.0.11:8778
   
   openstack endpoint create --region RegionOne placement internal http://192.168.0.11:8778
   
   openstack endpoint create --region RegionOne placement admin http://192.168.0.11:8778
   ```

9. 安装软件包：

   ```bash
   apt install placement-api
   ```

10. 编辑 `/etc/placement/placement.conf` 文件并完成以下操作：

    在 `[placement_database]` 部分中，配置数据库访问：

    ```py
    [placement_database]
    # ...
    connection = mysql+pymysql://placement:705432@192.168.0.11/placement
    ```

    在 `[api]` 和 `[keystone_authtoken]` 部分中，配置身份服务访问：

    ```py
    [api]
    # ...
    auth_strategy = keystone
    
    [keystone_authtoken]
    # ...
    auth_url = http://192.168.0.11:5000/v3
    memcached_servers = 192.168.0.11:11211
    auth_type = password
    project_domain_name = Default
    user_domain_name = Default
    project_name = service
    username = placement
    password = 705432
    ```

11. 填充 `placement` 数据库：

    ```bash
    su -s /bin/sh -c "placement-manage db sync" placement
    ```

12. 重新加载 Web 服务器进行调整以获得新的放置配置设置:

    ```bash
    service apache2 restart
    ```

### 验证安装

1. 获取 `admin` 凭据以访问仅限管理员的 CLI 命令：

   ```bash
   source /admin-openrc.sh
   ```

2. 执行状态检查以确保一切正常：

   ```bash
   placement-status upgrade check
   ```

3. 针对放置 API 运行一些命令：

   安装 osc-placement 插件：

   ```bash
   pip3 install osc-placement
   ```

   列出可用的资源类和特征：

   ``` bash
   openstack --os-placement-api-version 1.2 resource class list --sort-column name
   openstack --os-placement-api-version 1.6 trait list --sort-column name
   ```

### 安装和配置组件

1. 安装软件包：

   ```bash
   apt install nova-api nova-conductor nova-novncproxy nova-scheduler
   ```

2. 编辑 `/etc/nova/nova.conf` 文件并完成以下操作：

   在 `[api_database]` 和 `[database]` 部分中，配置数据库访问：

   ```py
   [api_database]
   # ...
   connection = mysql+pymysql://nova:705432@192.168.0.11/nova_api
   
   [database]
   # ...
   connection = mysql+pymysql://nova:705432@192.168.0.11/nova
   ```

   在 `[DEFAULT]` 部分中，配置 `RabbitMQ` 消息队列访问：

   ```py
   [DEFAULT]
   # ...
   transport_url = rabbit://openstack:705432@192.168.0.11:5672/
   ```

   在 `[api]` 和 `[keystone_authtoken]` 部分中，配置身份服务访问：

   注释掉或删除 `[keystone_authtoken]` 部分中的任何其他选项。

   ```py
   [api]
   # ...
   auth_strategy = keystone
   
   [keystone_authtoken]
   # ...
   www_authenticate_uri = http://192.168.0.11:5000/
   auth_url = http://192.168.0.11:5000/
   memcached_servers = 192.168.0.11:11211
   auth_type = password
   project_domain_name = Default
   user_domain_name = Default
   project_name = service
   username = nova
   password = 705432
   ```

   在 `[service_user]` 部分中，配置服务用户令牌：

   ```py
   [service_user]
   send_service_user_token = true
   auth_url = https://192.168.0.11/identity
   auth_strategy = keystone
   auth_type = password
   project_domain_name = Default
   project_name = service
   user_domain_name = Default
   username = nova
   password = 705432
   ```

   在 `[DEFAULT]` 部分中，配置 `my_ip` 选项以使用控制器节点的管理接口IP地址：

   ```py
   [DEFAULT]
   # ...
   my_ip = 192.168.0.11
   ```

   !!!配置 /etc/nova/nova.conf 的 `[neutron]` 部分。有关详细信息，请参阅网络服务安装指南。

   在 `[vnc]` 部分中，配置VNC代理以使用控制器节点的管理接口IP地址：

   ```py
   [vnc]
   enabled = true
   # ...
   server_listen = $my_ip
   server_proxyclient_address = $my_ip
   ```

   在 `[glance]` 部分，配置图片服务API的位置：

   ```py
   [glance]
   # ...
   api_servers = http://192.168.0.11:9292
   ```

   在 `[oslo_concurrency]` 部分，配置锁定路径：

   ```py
   [oslo_concurrency]
   # ...
   lock_path = /var/lib/nova/tmp
   ```

   由于打包错误，请从 `[DEFAULT]` 部分中删除 `log_dir` 选项。

   在 `[placement]` 部分中，配置对 Placement 服务的访问：

   ```py
   [placement]
   # ...
   region_name = RegionOne
   project_domain_name = Default
   project_name = service
   auth_type = password
   user_domain_name = Default
   auth_url = http://192.168.0.11:5000/v3
   username = placement
   password = 705432
   ```

3. 填充 `nova-api` 数据库：

   ```bash
   su -s /bin/sh -c "nova-manage api_db sync" nova
   ```

4. 注册 `cell0` 数据库：

   ```bash
   su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova
   ```

5. 创建 `cell1` 单元格：

   ```bash
   su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova
   ```

6. 填充 nova 数据库：

   ```bash
   su -s /bin/sh -c "nova-manage db sync" nova
   ```

7. 验证 nova cell0 和 cell1 是否已正确注册：

   ```bash
   su -s /bin/sh -c "nova-manage cell_v2 list_cells" nova
   ```

### 完成安装

重新启动计算服务：

```bash
service nova-api restart
service nova-scheduler restart
service nova-conductor restart
service nova-novncproxy restart
```

### 验证

1. 列出服务组件以验证每个进程的成功启动和注册：

   ```bash
   openstack compute service list		
   ```

2. 列出身份服务中的 API 端点以验证与身份服务的连接：

   ```bash
   openstack catalog list
   ```

3. 列出镜像服务中的镜像以验证与镜像服务的连接：

   ```bash
   openstack image list
   ```

4. 检查单元格和放置 API 是否成功运行以及其他必要的先决条件是否已到位：

   ```bash
   nova-status upgrade check
   ```

## 部署Cinder

### 安装和配置控制器节点

1. 要创建数据库，请完成以下步骤：

   使用数据库访问客户端以 `root` 用户连接数据库服务器：

   ```bash
   mysql
   ```

   创建 `cinder` 数据库：

   ```sql
   CREATE DATABASE cinder;
   ```

   授予对 `cinder` 数据库的正确访问权限：

   ```sql
   GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'localhost' IDENTIFIED BY '705432';
   GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'%' IDENTIFIED BY '705432';
   ```

2. 获取 `admin` 凭据以访问仅限管理员的 CLI 命令：

   ```bash
   source /admin-openrc.sh
   ```

3. 要创建服务凭证，请完成以下步骤：

   创建 `cinder` 用户：

   ```bash
   openstack user create --domain default --password-prompt cinder
   ```

   将 `admin` 角色添加到 `cinder` 用户：

   ```bash
   openstack role add --project service --user cinder admin
   ```

   创建 `cinderv3` 服务实体：

   ```bash
   openstack service create --name cinderv3 --description "OpenStack Block Storage" volumev3
   ```

4. 创建块存储服务 API 端点：

   ```bash
   openstack endpoint create --region RegionOne volumev3 public http://192.168.0.11:8776/v3/%\(project_id\)s
   
   openstack endpoint create --region RegionOne volumev3 internal http://192.168.0.11:8776/v3/%\(project_id\)s
   
   openstack endpoint create --region RegionOne volumev3 admin http://192.168.0.11:8776/v3/%\(project_id\)s
   ```

### 安装和配置组件

1. 安装软件包：

   ```bash
   apt install cinder-api cinder-scheduler
   ```

2. 编辑 `/etc/cinder/cinder.conf` 文件并完成以下操作：

   在 `[database]` 部分中，配置数据库访问：

   ```py
   [database]
   # ...
   connection = mysql+pymysql://cinder:705432@192.168.0.11/cinder
   ```

   在 `[DEFAULT]` 部分中，配置 `RabbitMQ` 消息队列访问：

   ```py
   [DEFAULT]
   # ...
   transport_url = rabbit://openstack:705432@192.168.0.11
   ```

   在 `[DEFAULT]` 和 `[keystone_authtoken]` 部分中，配置身份服务访问：

   注释掉或删除 `[keystone_authtoken]` 部分中的任何其他选项。

   ```py
   [DEFAULT]
   # ...
   auth_strategy = keystone
   
   [keystone_authtoken]
   # ...
   www_authenticate_uri = http://192.168.0.11:5000
   auth_url = http://192.168.0.11:5000
   memcached_servers = 192.168.0.11:11211
   auth_type = password
   project_domain_name = default
   user_domain_name = default
   project_name = service
   username = cinder
   password = 705432
   ```

   在 `[DEFAULT]` 部分中，配置 `my_ip` 选项以使用控制器节点的管理接口IP地址：

   ```py
   [DEFAULT]
   # ...
   my_ip = 192.168.0.11
   ```

3. 在 `[oslo_concurrency]` 部分，配置锁定路径：

   ```py
   [oslo_concurrency]
   # ...
   lock_path = /var/lib/cinder/tmp
   ```

4. 填充块存储数据库：

   ```bash
   su -s /bin/sh -c "cinder-manage db sync" cinder
   ```

### 配置计算以使用块存储

编辑 `/etc/nova/nova.conf` 文件并向其中添加以下内容：

```py
[cinder]
os_region_name = RegionOne
```

### 完成安装

1. 重启计算API服务：

   ```bash
   service nova-api restart
   ```

2. 重新启动块存储服务：

   ```bash
   service cinder-scheduler restart
   service apache2 restart
   ```

### 验证

列出服务组件以验证每个进程是否成功启动：

```bash
openstack volume service list
```

## Glance

### 安装和配置

1. 要创建数据库，请完成以下步骤：

   使用数据库访问客户端以 `root` 用户连接数据库服务器：

   ```bash
   mysql
   ```

   创建 `glance` 数据库：

   ```sql
   CREATE DATABASE glance;
   ```

   授予对 `glance` 数据库的正确访问权限：

   ```
   GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY '705432';
   GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY '705432';
   ```

2. 获取 `admin` 凭据以访问仅限管理员的 CLI 命令：

   ```bash
   source /admin-openrc.sh
   ```

3. 要创建服务凭证，请完成以下步骤：

   创建 `glance` 用户：

   ```bash
   openstack user create --domain default --password-prompt glance
   ```

   将 `admin` 角色添加到 `glance` 用户和 `service` 项目：

   ```bash
   openstack role add --project service --user glance admin
   ```

   创建 `glance` 服务实体：

   ```bash
   openstack service create --name glance --description "OpenStack Image" image
   ```

4. 创建镜像服务 API 端点：

   ```
   openstack endpoint create --region RegionOne image public http://192.168.0.11:9292
   openstack endpoint create --region RegionOne image internal http://192.168.0.11:9292
   openstack endpoint create --region RegionOne image admin http://192.168.0.11:9292
   ```

### 安装和配置组件

1. 安装软件包：

   ```bash
   apt install glance
   ```

2. 编辑 `/etc/glance/glance-api.conf` 文件并完成以下操作：

   在 `[database]` 部分中，配置数据库访问：

   ```py
   [database]
   # ...
   connection = mysql+pymysql://glance:705432@192.168.0.11/glance
   ```

   在 `[keystone_authtoken]` 和 `[paste_deploy]` 部分中，配置身份服务访问：

   注释掉或删除 `[keystone_authtoken]` 部分中的任何其他选项。

   ```bash
   [keystone_authtoken]
   # ...
   www_authenticate_uri = http://192.168.0.11:5000
   auth_url = http://192.168.0.11:5000
   memcached_servers = 192.168.0.11:11211
   auth_type = password
   project_domain_name = Default
   user_domain_name = Default
   project_name = service
   username = glance
   password = 705432
   
   [paste_deploy]
   # ...
   flavor = keystone
   ```

   在 `[glance_store]` 部分中，配置本地文件系统存储和镜像文件的位置：

   ```py
   [glance_store]
   # ...
   stores = file,http
   default_store = file
   filesystem_store_datadir = /var/lib/glance/images/
   ```

3. 填充镜像服务数据库：

   ```bash
   su -s /bin/sh -c "glance-manage db_sync" glance
   ```

### 完成安装

重新启动镜像服务：

```bash
service glance-api restart
```

### 验证

1. 下载源镜像：

   ```bash
   wget http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img
   ```

2. 使用QCOW2磁盘格式、裸容器格式和公共可见性将镜像上传到镜像服务，以便所有项目都可以访问它：

   ```bash
   glance image-create --name "cirros" \
   --file cirros-0.4.0-x86_64-disk.img \
   --disk-format qcow2 --container-format bare \
   --visibility=public
   ```

3. 确认上传镜像并验证属性：

   ```bash
   glance image-list
   ```

# 计算节点

## 部署Nova

### 安装和配置组件

1. 安装软件包：

   ```bash
   apt install nova-compute
   ```

2. 编辑 `/etc/nova/nova.conf` 文件并完成以下操作：

   在 `[DEFAULT]` 部分中，配置 `RabbitMQ` 消息队列访问：

   ```py
   [DEFAULT]
   # ...
   transport_url = rabbit://openstack:705432@192.168.0.11
   ```

   在 `[api]` 和 `[keystone_authtoken]` 部分中，配置身份服务访问：

   注释掉或删除 `[keystone_authtoken]` 部分中的任何其他选项。

   ```py
   [api]
   # ...
   auth_strategy = keystone
   
   [keystone_authtoken]
   # ...
   www_authenticate_uri = http://192.168.0.11:5000/
   auth_url = http://192.168.0.11:5000/
   memcached_servers = 192.168.0.11:11211
   auth_type = password
   project_domain_name = Default
   user_domain_name = Default
   project_name = service
   username = nova
   password = 705432
   ```

   在 `[service_user]` 部分中，配置服务用户令牌：

   ```py
   [service_user]
   send_service_user_token = true
   auth_url = https://192.168.0.11/identity
   auth_strategy = keystone
   auth_type = password
   project_domain_name = Default
   project_name = service
   user_domain_name = Default
   username = nova
   password = 705432
   ```

   在 `[DEFAULT]` 部分中，配置 `my_ip` 选项：

   ```py
   [DEFAULT]
   # ...
   my_ip = 192.168.0.xx // 11 -> 31 41
   ```

   !!!配置 /etc/nova/nova.conf 的 `[neutron]` 部分。有关更多详细信息，请参阅网络服务安装指南。

   在 `[vnc]` 部分中，启用并配置远程控制台访问：

   ```py
   [vnc]
   # ...
   enabled = true
   server_listen = 0.0.0.0
   server_proxyclient_address = $my_ip
   novncproxy_base_url = http://192.168.0.11:6080/vnc_auto.html
   ```

   在 `[glance]` 部分，配置图片服务API的位置：

   ```py
   [glance]
   # ...
   api_servers = http://192.168.0.11:9292
   ```

   在 `[oslo_concurrency]` 部分，配置锁定路径：

   ```py
   [oslo_concurrency]
   # ...
   lock_path = /var/lib/nova/tmp
   ```

   在 `[placement]` 部分中，配置 Placement API：

   ```py
   [placement]
   # ...
   region_name = RegionOne
   project_domain_name = Default
   project_name = service
   auth_type = password
   user_domain_name = Default
   auth_url = http://192.168.0.11:5000/v3
   username = placement
   password = 705432
   ```

### 完成安装

1. 确定您的计算节点是否支持虚拟机硬件加速：

   ```bash
   egrep -c '(vmx|svm)' /proc/cpuinfo
   ```

   - 如果此命令返回值 `one or greater` ，则您的计算节点支持硬件加速，通常不需要额外的配置。

   - 如果此命令返回值 `zero` ，则您的计算节点不支持硬件加速，您必须配置 `libvirt` 以使用 QEMU 而不是 KVM。

     编辑 `/etc/nova/nova-compute.conf` 文件中的 `[libvirt]` 部分，如下所示：

     ```py
     [libvirt]
     # ...
     virt_type = qemu
     ```

2. ```bash
   service nova-compute restart
   ```

### 将计算节点添加到单元数据库

***<u>在控制节点上运行以下命令</u>***

1. 获取管理员凭据以启用仅限管理员的 CLI 命令，然后确认数据库中存在计算主机：

   ```bash
   source /admin-openrc.sh
   
   openstack compute service list --service nova-compute
   ```

2. 发现计算主机：

   ```bash
   su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova
   ```

### 验证

1. 列出服务组件以验证每个进程的成功启动和注册：

   ```bash
   openstack compute service list		
   ```

2. 列出身份服务中的 API 端点以验证与身份服务的连接：

   ```bash
   openstack catalog list
   ```

3. 列出镜像服务中的镜像以验证与镜像服务的连接：

   ```bash
   openstack image list
   ```

4. 检查单元格和放置 API 是否成功运行以及其他必要的先决条件是否已到位：

   ```bash
   nova-status upgrade check
   ```