# 👉🏻[ubuntu搭建Openstack桌面云详细步骤](./ubuntu-22.04.3LTS-arm64.md)

## 🧐阅前须知

- Linux版本: **Ubuntu Server 22.04.3 LTS**

  > 对Ubuntu 16.04+ (LTS)的版本都可以安装Openstack - yoga相关软件包,但本文档不一定均适用

- 所有虚拟机的安装本文档均不涉及⛔️，网卡接口按需配置，但需要配置一个固定ip4为相关服务提供调度
- 文档内容基于官网教程翻译及个人实践整合，更详细内容请移步官网：https://docs.openstack.org/yoga/install/index.html

## 🚧 施工进度

- [x] Keystone
- [x] Horizon
- [x] Nova
- [x] Placement
- [x] Cinder
- [x] Glance
- [ ] Neutron
