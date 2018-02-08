# kubernetes 1.9.1

## 环境说明

| 主机名称 | IP | 备注 | 系统 | 配置 |
| ----- |:----:|:-----:|:-----:|:----:|
| k8s-master-149 | 192.168.134.149 | master、etcd、flannel、node | CentOS7.4 x86_64 | 2C2G20G(虚拟机) |
| k8s-mastet-150 | 192.168.134.150 | master、etcd、flannel、node | CentOS7.4 x86_64 | 2C2G20G(虚拟机) |
| k8s-master-151 | 192.168.134.151 | master、etcd、flannel、node | CentOS7.4 x86_64 | 2C2G20G(虚拟机) |

## 初始化环境

```
hostnamectl set-hostname k8s-master-149
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
setenforce 0
systemctl disable firewalld
systemctl stop firewalld
```

```
#编辑 /etc/hosts 文件

vim /etc/hosts

192.168.134.149 k8s-master-149
192.168.134.150 k8s-master-150
192.168.134.151 k8s-master-151
```
