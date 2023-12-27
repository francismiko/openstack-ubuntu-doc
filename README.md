# 👉🏻[ubuntu搭建Openstack桌面云详细步骤](./ubuntu-22.04.3LTS-arm64.md)

## 🧐 阅前须知

- Linux版本: **Ubuntu Server 22.04.3 LTS**

  > 对Ubuntu 16.04+ (LTS)的版本都可以安装Openstack - yoga相关软件包,但本文档不一定均适用

- 本文档没有配置apt国内镜像源, 有需要请自行参考各种镜像源配置教程

- 所有虚拟机的安装本文档均不涉及⛔️，网卡接口按需配置，但需要配置一个固定ip4为相关服务提供调度端口

- 文档内容基于官网教程翻译及个人实践整合

  > 更详细内容请移步官网：https://docs.openstack.org/yoga/install/index.html

## 🚧 施工进度

### 最小化配置

- [x] Keystone
- [x] Horizon
- [x] Nova
- [x] Placement
- [x] Cinder
- [x] Glance
- [ ] Neutron

### 以下为选配内容

- [ ] Freezer
- [ ] Ironic
- [ ] Senlin
- [ ] Zun
- [ ] Ec2-Api
- [ ] Watcher
- [ ] Masakari
- [ ] Barbican
- [ ] Octavia
- [ ] Tacker
- [ ] Swift
- [ ] Heat
- [ ] Vitrage
- [ ] Blazar
- [ ] Manila
- [ ] Aodh
- [ ] Ceilometer
