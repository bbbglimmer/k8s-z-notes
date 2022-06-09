# 基于 KubeSphere 玩转 k8s-Ceph 安装手记

**大家好，我是老 Z！**

> 本系列文档是我在云原生技术领域的学习和运维实践的手记，**用输出倒逼输入**是一种高效的学习方法，能够快速积累经验和提高技术，只有把学到的知识写出来并能够让其他人理解，才能说明真正掌握了这项知识。
>
> 如果你喜欢本文，请分享给你的小伙伴！

**本系列文档内容涵盖 (但不限于) 以下技术领域：**

> - **KubeSphere**
>
> - **Kubernetes**
>
> - **Ansible**
>
> - **自动化运维**
>
> - **CNCF 技术栈**

## 1. 本文简介

本文简要介绍了几种常见的 Ceph 分布式存储安装方案，重点演示了 Ceph-deploy 方案的详细操作过程。

由于 Ceph-deploy 已经被官方所放弃，因此，本文适用于学习测试环境。

> 本文知识量

- 阅读时长：20 分
- 行：1600+
- 单词：8000+
- 字符：74400+
- 图片：0 张

> **本文知识点**

- 定级：**入门级**
- Ceph 的常用安装部署
- Ceph-deploy 安装部署 Ceph 集群
- Ceph-deploy 扩展 Ceph 集群

> **演示服务器配置**

|     主机名      |      IP      | CPU  | 内存 | 系统盘 | 数据盘 |               用途               |
| :-------------: | :----------: | :--: | :--: | :----: | :----: | :------------------------------: |
|  zdeops-master  | 192.168.9.9  |  2   |  4   |   40   |  200   |       Ansible 运维控制节点        |
| ks-k8s-master-0 | 192.168.9.91 |  8   |  32  |   40   |  200   | KubeSphere/k8s-master/k8s-worker |
| ks-k8s-master-1 | 192.168.9.92 |  8   |  32  |   40   |  200   | KubeSphere/k8s-master/k8s-worker |
| ks-k8s-master-2 | 192.168.9.93 |  8   |  32  |   40   |  200   | KubeSphere/k8s-master/k8s-worker |
|   ceph-node-0   | 192.168.9.85 |  4   |  8   |   40   |  200   |               Ceph               |
|   ceph-node-1   | 192.168.9.86 |  4   |  8   |   40   |  200   |               Ceph               |
|   ceph-node-2   | 192.168.9.87 |  4   |  8   |   40   |  200   |               Ceph               |

> **演示环境涉及软件版本信息**

- 操作系统：**CentOS-7.9-x86_64**
- KubeSphere：**3.2.1**
- Ansible：**2.8.20**
- Ceph：**Octopus(15.2.16 )** 



## 2. Ansible 配置

### 2.1. 增加 hosts 配置

```ini
# 主要增加 ceph 节点配置

[ceph]
ceph-node-0 ansible_ssh_host=192.168.9.85 host_name=ceph-node-0
ceph-node-1 ansible_ssh_host=192.168.9.86 host_name=ceph-node-1
ceph-node-2 ansible_ssh_host=192.168.9.87 host_name=ceph-node-2

[servers:children]
k8s
glusterfs
es
ceph
```



## 3. Ceph 安装方式介绍

### 3.1. Ceph-deploy

- [Ceph-deploy](https://docs.ceph.com/projects/ceph-deploy/en/latest/) 是一个快速部署 Ceph 集群的工具。

- **Ceph-deploy 不再 (actively) 积极维护。**
- **在高于 Nautilus 的版本上没有测试过。**
- **不支持 RHEL8, CentOS 8, 或者更新的操作系统。**
- CentOS 7.9 搭配 Ceph Nautilus 版本时首选，但是 **Ceph Nautilus**(2019-05-01 首发) 有点老旧了能不选还是不要选了。

> CentOS 8 我还没有玩过，也没有计划去搞，所有学习测试环境最终选择了 CentOS7.9 搭配 **Octopus** 的方案。

### 3.2. Cephadm

- [Cephadm](https://docs.ceph.com/en/quincy/cephadm/#cephadm) 使用容器和 systemd 安装和管理 Ceph 集群，并与 CLI 和仪表板 GUI 紧密集成。

- Cephadm 仅支持 Octopus  v15.2.0 和更新版本。

- Cephadm 与新的 orchestrator API 完全集成，并完全支持新的 CLI 和仪表盘功能来管理集群部署。

- Cephadm 需要容器支持 (podman 或 docker) 和 Python 3。

- 非 Kubernetes 环境首选。

### 3.3. Rook

- [Rook](https://rook.io/) 部署和管理运行在 Kubernetes 中的 Ceph 集群，同时也支持通过 Kubernetes APi 管理存储资源和 provisionin。
- 官方推荐使用 Rook 在 Kubernetes 中运行 Ceph，或者将现有的 Ceph 存储集群连接到 Kubernetes。
- Rook 只支持 Nautilus 和 Ceph 的更新版本。
- Rook 支持新的 orchestrator API，并完全支持在 CLI 和仪表盘 中的新的管理功能。

- Rook 是在 Kubernetes 上运行 Ceph 的首选方法，或者是将 Kubernetes 集群连接到现有 (外部)Ceph 集群的首选方法。

### 3.4. 手动安装

你也可以不采用任何部署工具，一步步的手工安装 Ceph 集群，安装软件包、初始化集群、安装 Mon、安装配置 OSD 等。

具体可以参考[官方手工安装文档](https://docs.ceph.com/en/latest/install/index_manual/)，除非你要自己编写自动化部署工具，可以参考官方的手工操作方式，否则不建议采用手工的方式。



## 4. Ceph 节点初始化配置

### 4.1. 检测服务器连通性

> **执行任务命令**

```shell
# 利用 ansible 检测服务器的连通性
cd /data/ansible/ansible-zdevops/inventories/dev/
source /opt/ansible2.8/bin/activate
ansible -m ping es 
```

> **正确执行输出参考**

```shell
[root@zdevops-master ~]# cd /data/ansible/ansible-zdevops/inventories/dev/
[root@zdevops-master dev]# source /opt/ansible2.8/bin/activate
(ansible2.8) [root@zdevops-master dev]# ansible -m ping ceph
/opt/ansible2.8/lib/python2.7/site-packages/ansible/parsing/vault/__init__.py:44: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.
  from cryptography.exceptions import InvalidSignature
ceph-node-2 | SUCCESS => {
    "changed": false, 
    "ping": "pong"
}
ceph-node-1 | SUCCESS => {
    "changed": false, 
    "ping": "pong"
}
ceph-node-0 | SUCCESS => {
    "changed": false, 
    "ping": "pong"
}
```

### 4.2. 初始化服务器配置

> **执行任务命令**

**重点注意：ansible-playbook 的 -l 参数，需要指定为 ceph，因为 init-base.yaml 文件默认指定的是所有服务器都执行，不加-l 就会把 hosts 文件里指定的所有服务器都初始化了。**

```shell
# 利用 ansible-playbook 初始化服务器配置
ansible-playbook ../../playbooks/init-base.yaml -l ceph
```

> **正确执行输出参考**

```shell
(ansible2.8) [root@zdevops-master dev]# ansible-playbook ../../playbooks/init-base.yaml -l ceph
/opt/ansible2.8/lib/python2.7/site-packages/ansible/parsing/vault/__init__.py:44: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.
  from cryptography.exceptions import InvalidSignature

PLAY [初始化服务器配置.] *************************************************************************************************

TASK [01-停止并禁用firewalld服务.] **************************************************************************************
changed: [ceph-node-0]
changed: [ceph-node-2]
changed: [ceph-node-1]

TASK [02-配置主机名.] *************************************************************************************************
changed: [ceph-node-0]
changed: [ceph-node-2]
changed: [ceph-node-1]

TASK [03-配置/etc/hosts.] ******************************************************************************************
changed: [ceph-node-2]
changed: [ceph-node-0]
changed: [ceph-node-1]

TASK [04-配置时区.] **************************************************************************************************
ok: [ceph-node-1]
ok: [ceph-node-0]
ok: [ceph-node-2]

TASK [05-升级操作系统.] ************************************************************************************************
skipping: [ceph-node-0]
skipping: [ceph-node-1]
skipping: [ceph-node-2]

TASK [05-升级操作系统后如果需要重启，则重启服务器.] **********************************************************************************
skipping: [ceph-node-0]
skipping: [ceph-node-1]
skipping: [ceph-node-2]

TASK [05-等待服务器完成重启.] *********************************************************************************************
skipping: [ceph-node-0]
skipping: [ceph-node-1]
skipping: [ceph-node-2]

PLAY [安装配置chrony服务器.] ********************************************************************************************

TASK [01-安装chrony软件包.] *******************************************************************************************
changed: [ceph-node-2] => (item=[u'chrony'])
changed: [ceph-node-1] => (item=[u'chrony'])
changed: [ceph-node-0] => (item=[u'chrony'])

TASK [02-配置chrony.conf.] *****************************************************************************************
changed: [ceph-node-1]
changed: [ceph-node-2]
changed: [ceph-node-0]

TASK [03-确认chrony服务启动并实现开机自启.] ***********************************************************************************
changed: [ceph-node-2]
changed: [ceph-node-1]
changed: [ceph-node-0]

TASK [04-查看chrony时间同步服务器列表(1).] **********************************************************************************
changed: [ceph-node-2]
changed: [ceph-node-1]
changed: [ceph-node-0]

TASK [04-查看chrony时间同步服务器列表(2).] **********************************************************************************
ok: [ceph-node-0] => {
    "chronyc_out.stdout_lines": [
        "210 Number of sources = 1", 
        "MS Name/IP address         Stratum Poll Reach LastRx Last sample               ", 
        "===============================================================================", 
        "^? 114.118.7.161                 0   6     0     -     +0ns[   +0ns] +/-    0ns"
    ]
}
ok: [ceph-node-1] => {
    "chronyc_out.stdout_lines": [
        "210 Number of sources = 1", 
        "MS Name/IP address         Stratum Poll Reach LastRx Last sample               ", 
        "===============================================================================", 
        "^? 114.118.7.161                 1   6     1     1   -63.8s[ -63.8s] +/- 6042us"
    ]
}
ok: [ceph-node-2] => {
    "chronyc_out.stdout_lines": [
        "210 Number of sources = 1", 
        "MS Name/IP address         Stratum Poll Reach LastRx Last sample               ", 
        "===============================================================================", 
        "^? 114.118.7.163                 0   6     0     -     +0ns[   +0ns] +/-    0ns"
    ]
}

PLAY RECAP *******************************************************************************************************
ceph-node-0                : ok=9    changed=7    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0   
ceph-node-1                : ok=9    changed=7    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0   
ceph-node-2                : ok=9    changed=7    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0
```

### 4.3. EPEL 软件源配置

> **执行任务命令**

```shell
ansible ceph -m shell -a 'yum install -y epel-release'
```

> **正确执行输出参考**

```shell
(ansible2.8) [root@zdevops-master dev]# ansible ceph -m shell -a 'yum install -y epel-release'
/opt/ansible2.8/lib/python2.7/site-packages/ansible/parsing/vault/__init__.py:44: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.
  from cryptography.exceptions import InvalidSignature
[WARNING]: Consider using the yum module rather than running 'yum'.  If you need to use command because yum is
insufficient you can add 'warn: false' to this command task or set 'command_warnings=False' in ansible.cfg to get
rid of this message.

ceph-node-2 | CHANGED | rc=0 >>
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
 * base: mirrors.aliyun.com
 * extras: mirrors.aliyun.com
 * updates: mirrors.aliyun.com
Resolving Dependencies
--> Running transaction check
---> Package epel-release.noarch 0:7-11 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

================================================================================
 Package                Arch             Version         Repository        Size
================================================================================
Installing:
 epel-release           noarch           7-11            extras            15 k

Transaction Summary
================================================================================
Install  1 Package

Total download size: 15 k
Installed size: 24 k
Downloading packages:
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Installing : epel-release-7-11.noarch                                     1/1 
  Verifying  : epel-release-7-11.noarch                                     1/1 

Installed:
  epel-release.noarch 0:7-11                                                    

Complete!

ceph-node-0 | CHANGED | rc=0 >>
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
 * base: mirrors.aliyun.com
 * extras: mirrors.aliyun.com
 * updates: mirrors.aliyun.com
Resolving Dependencies
--> Running transaction check
---> Package epel-release.noarch 0:7-11 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

================================================================================
 Package                Arch             Version         Repository        Size
================================================================================
Installing:
 epel-release           noarch           7-11            extras            15 k

Transaction Summary
================================================================================
Install  1 Package

Total download size: 15 k
Installed size: 24 k
Downloading packages:
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Installing : epel-release-7-11.noarch                                     1/1 
  Verifying  : epel-release-7-11.noarch                                     1/1 

Installed:
  epel-release.noarch 0:7-11                                                    

Complete!

ceph-node-1 | CHANGED | rc=0 >>
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
 * base: mirrors.aliyun.com
 * extras: mirrors.aliyun.com
 * updates: mirrors.aliyun.com
Resolving Dependencies
--> Running transaction check
---> Package epel-release.noarch 0:7-11 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

================================================================================
 Package                Arch             Version         Repository        Size
================================================================================
Installing:
 epel-release           noarch           7-11            extras            15 k

Transaction Summary
================================================================================
Install  1 Package

Total download size: 15 k
Installed size: 24 k
Downloading packages:
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Installing : epel-release-7-11.noarch                                     1/1 
  Verifying  : epel-release-7-11.noarch                                     1/1 

Installed:
  epel-release.noarch 0:7-11                                                    

Complete!
```

### 4.4. Ceph 软件源配置

>**执行任务命令**

```shell
# 利用 ansible 安装 ceph-release 包
ansible ceph -m shell -a 'yum install -y https://download.ceph.com/rpm-octopus/el7/noarch/ceph-release-1-1.el7.noarch.rpm'

# 验证生成的 ceph.repo 文件
ansible ceph -m shell -a 'cat /etc/yum.repos.d/ceph.repo'
```

>**正确执行输出参考**

```shell
# 利用 ansible 安装 ceph-release 包
(ansible2.8) [root@zdevops-master dev]# ansible ceph -m shell -a 'yum install -y https://download.ceph.com/rpm-octopus/el7/noarch/ceph-release-1-1.el7.noarch.rpm'
/opt/ansible2.8/lib/python2.7/site-packages/ansible/parsing/vault/__init__.py:44: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.
  from cryptography.exceptions import InvalidSignature
[WARNING]: Consider using the yum module rather than running 'yum'.  If you need to use command because yum is
insufficient you can add 'warn: false' to this command task or set 'command_warnings=False' in ansible.cfg to get
rid of this message.

ceph-node-1 | CHANGED | rc=0 >>
Loaded plugins: fastestmirror
Examining /var/tmp/yum-root-UJAX7G/ceph-release-1-1.el7.noarch.rpm: ceph-release-1-1.el7.noarch
Marking /var/tmp/yum-root-UJAX7G/ceph-release-1-1.el7.noarch.rpm to be installed
Resolving Dependencies
--> Running transaction check
---> Package ceph-release.noarch 0:1-1.el7 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

================================================================================
 Package          Arch       Version     Repository                        Size
================================================================================
Installing:
 ceph-release     noarch     1-1.el7     /ceph-release-1-1.el7.noarch     541  

Transaction Summary
================================================================================
Install  1 Package

Total size: 541  
Installed size: 541  
Downloading packages:
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Installing : ceph-release-1-1.el7.noarch                                  1/1 
  Verifying  : ceph-release-1-1.el7.noarch                                  1/1 

Installed:
  ceph-release.noarch 0:1-1.el7                                                 

Complete!

ceph-node-0 | CHANGED | rc=0 >>
Loaded plugins: fastestmirror
Examining /var/tmp/yum-root-JWJnoN/ceph-release-1-1.el7.noarch.rpm: ceph-release-1-1.el7.noarch
Marking /var/tmp/yum-root-JWJnoN/ceph-release-1-1.el7.noarch.rpm to be installed
Resolving Dependencies
--> Running transaction check
---> Package ceph-release.noarch 0:1-1.el7 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

================================================================================
 Package          Arch       Version     Repository                        Size
================================================================================
Installing:
 ceph-release     noarch     1-1.el7     /ceph-release-1-1.el7.noarch     541  

Transaction Summary
================================================================================
Install  1 Package

Total size: 541  
Installed size: 541  
Downloading packages:
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Installing : ceph-release-1-1.el7.noarch                                  1/1 
  Verifying  : ceph-release-1-1.el7.noarch                                  1/1 

Installed:
  ceph-release.noarch 0:1-1.el7                                                 

Complete!

ceph-node-2 | CHANGED | rc=0 >>
Loaded plugins: fastestmirror
Examining /var/tmp/yum-root-z2H1Lx/ceph-release-1-1.el7.noarch.rpm: ceph-release-1-1.el7.noarch
Marking /var/tmp/yum-root-z2H1Lx/ceph-release-1-1.el7.noarch.rpm to be installed
Resolving Dependencies
--> Running transaction check
---> Package ceph-release.noarch 0:1-1.el7 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

================================================================================
 Package          Arch       Version     Repository                        Size
================================================================================
Installing:
 ceph-release     noarch     1-1.el7     /ceph-release-1-1.el7.noarch     541  

Transaction Summary
================================================================================
Install  1 Package

Total size: 541  
Installed size: 541  
Downloading packages:
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Installing : ceph-release-1-1.el7.noarch                                  1/1 
  Verifying  : ceph-release-1-1.el7.noarch                                  1/1 

Installed:
  ceph-release.noarch 0:1-1.el7                                                 

Complete!

# 验证生成的 ceph.repo 文件
(ansible2.8) [root@zdevops-master dev]# ansible ceph -m shell -a 'cat /etc/yum.repos.d/ceph.repo'
/opt/ansible2.8/lib/python2.7/site-packages/ansible/parsing/vault/__init__.py:44: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.
  from cryptography.exceptions import InvalidSignature
ceph-node-2 | CHANGED | rc=0 >>
[Ceph]
name=Ceph packages for $basearch
baseurl=http://download.ceph.com/rpm-octopus/el7/$basearch
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://download.ceph.com/keys/release.asc

[Ceph-noarch]
name=Ceph noarch packages
baseurl=http://download.ceph.com/rpm-octopus/el7/noarch
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://download.ceph.com/keys/release.asc

[ceph-source]
name=Ceph source packages
baseurl=http://download.ceph.com/rpm-octopus/el7/SRPMS
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://download.ceph.com/keys/release.asc

ceph-node-1 | CHANGED | rc=0 >>
[Ceph]
name=Ceph packages for $basearch
baseurl=http://download.ceph.com/rpm-octopus/el7/$basearch
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://download.ceph.com/keys/release.asc

[Ceph-noarch]
name=Ceph noarch packages
baseurl=http://download.ceph.com/rpm-octopus/el7/noarch
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://download.ceph.com/keys/release.asc

[ceph-source]
name=Ceph source packages
baseurl=http://download.ceph.com/rpm-octopus/el7/SRPMS
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://download.ceph.com/keys/release.asc

ceph-node-0 | CHANGED | rc=0 >>
[Ceph]
name=Ceph packages for $basearch
baseurl=http://download.ceph.com/rpm-octopus/el7/$basearch
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://download.ceph.com/keys/release.asc

[Ceph-noarch]
name=Ceph noarch packages
baseurl=http://download.ceph.com/rpm-octopus/el7/noarch
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://download.ceph.com/keys/release.asc

[ceph-source]
name=Ceph source packages
baseurl=http://download.ceph.com/rpm-octopus/el7/SRPMS
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://download.ceph.com/keys/release.asc
```



## 5. Ceph-deploy 安装配置

**本节所有操作都在 ceph-node-0 节点执行**。

### 5.1. Ceph-deploy 节点 SSH 免密配置

指定 **ceph-node-0** 节点作为 Ceph-deploy 服务器，在该节点配置免密登录所有 Ceph 节点。

> **执行任务命令**

```shell
# 生成密钥
ssh-keygen -f $HOME/.ssh/id_rsa -t rsa -N ''

# 复制密钥(按提示输入 yes 和服务器密码)
ssh-copy-id root@192.168.9.85
ssh-copy-id root@192.168.9.86
ssh-copy-id root@192.168.9.87
```

> **正确执行输出参考**

```shell
# 生成密钥
[root@ceph-node-0 ~]# ssh-keygen -f $HOME/.ssh/id_rsa -t rsa -N ''
Generating public/private rsa key pair.
Your identification has been saved in /root/.ssh/id_rsa.
Your public key has been saved in /root/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:/h4qNJIck6K4nCWwBLXPvJKPPu/m/opjBVb31rFgABA root@ceph-node-0
The key's randomart image is:
+---[RSA 2048]----+
| .Eo....         |
|.  .. . o .      |
|. .. o o o o     |
|..+++   o o      |
|++ +++ .S        |
|+. o=.o.         |
|..*..o .. .      |
|.o+=. .  o .     |
| o+OBo....o      |
+----[SHA256]-----+

# 复制密钥
[root@ceph-node-0 ~]# ssh-copy-id root@192.168.9.85
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/root/.ssh/id_rsa.pub"
The authenticity of host '192.168.9.85 (192.168.9.85)' can't be established.
ECDSA key fingerprint is SHA256:/0HffAhG/oTriuqo7KjAdf/YQNoFSbwYk/SAVUYH6Fs.
ECDSA key fingerprint is MD5:34:67:af:a6:50:c0:5f:33:07:45:11:e8:b8:52:e2:ae.
Are you sure you want to continue connecting (yes/no)? yes
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
root@192.168.9.85's password: 

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh 'root@192.168.9.85'"
and check to make sure that only the key(s) you wanted were added.

```

### 5.2 Ceph-deploy 安装

> **执行任务命令**

```shell
# 安装ceph-deploy
yum install ceph-deploy -y

# 安装依赖
yum install python-setuptools -y
```

> **正确执行输出参考**

```shell
# 安装 ceph-deploy
[root@ceph-node-0 ~]# yum install ceph-deploy -y
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
epel/x86_64/metalink                                                                       | 6.3 kB  00:00:00     
 * base: mirrors.aliyun.com
 * epel: ftp.iij.ad.jp
 * extras: mirrors.aliyun.com
 * updates: mirrors.aliyun.com
epel                                                                                       | 4.7 kB  00:00:00     
(1/3): epel/x86_64/group_gz                                                                |  96 kB  00:00:00     
(2/3): epel/x86_64/updateinfo                                                              | 1.0 MB  00:00:03     
(3/3): epel/x86_64/primary_db                                                              | 7.0 MB  00:00:34     
Resolving Dependencies
--> Running transaction check
---> Package ceph-deploy.noarch 0:2.0.1-0 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

==================================================================================================================
 Package                      Arch                    Version                  Repository                    Size
==================================================================================================================
Installing:
 ceph-deploy                  noarch                  2.0.1-0                  Ceph-noarch                  286 k

Transaction Summary
==================================================================================================================
Install  1 Package

Total download size: 286 k
Installed size: 1.2 M
Downloading packages:
warning: /var/cache/yum/x86_64/7/Ceph-noarch/packages/ceph-deploy-2.0.1-0.noarch.rpm: Header V4 RSA/SHA256 Signature, key ID 460f3994: NOKEY
Public key for ceph-deploy-2.0.1-0.noarch.rpm is not installed
ceph-deploy-2.0.1-0.noarch.rpm                                                             | 286 kB  00:00:01     
Retrieving key from https://download.ceph.com/keys/release.asc
Importing GPG key 0x460F3994:
 Userid     : "Ceph.com (release key) <security@ceph.com>"
 Fingerprint: 08b7 3419 ac32 b4e9 66c1 a330 e84a c2c0 460f 3994
 From       : https://download.ceph.com/keys/release.asc
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Installing : ceph-deploy-2.0.1-0.noarch                                                                     1/1 
  Verifying  : ceph-deploy-2.0.1-0.noarch                                                                     1/1 

Installed:
  ceph-deploy.noarch 0:2.0.1-0                                                                                    

Complete!

# 安装 python-setuptools
[root@localhost ~]# yum install python-setuptools -y
Loaded plugins: fastestmirror, priorities
Loading mirror speeds from cached hostfile
 * base: mirrors.aliyun.com
 * epel: ftp.iij.ad.jp
 * extras: mirrors.aliyun.com
 * updates: mirrors.aliyun.com
Resolving Dependencies
--> Running transaction check
---> Package python-setuptools.noarch 0:0.9.8-7.el7 will be installed
--> Processing Dependency: python-backports-ssl_match_hostname for package: python-setuptools-0.9.8-7.el7.noarch
--> Running transaction check
---> Package python-backports-ssl_match_hostname.noarch 0:3.5.0.1-1.el7 will be installed
--> Processing Dependency: python-ipaddress for package: python-backports-ssl_match_hostname-3.5.0.1-1.el7.noarch
--> Processing Dependency: python-backports for package: python-backports-ssl_match_hostname-3.5.0.1-1.el7.noarch
--> Running transaction check
---> Package python-backports.x86_64 0:1.0-8.el7 will be installed
---> Package python-ipaddress.noarch 0:1.0.16-2.el7 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

==================================================================================================================
 Package                                        Arch              Version                   Repository       Size
==================================================================================================================
Installing:
 python-setuptools                              noarch            0.9.8-7.el7               base            397 k
Installing for dependencies:
 python-backports                               x86_64            1.0-8.el7                 base            5.8 k
 python-backports-ssl_match_hostname            noarch            3.5.0.1-1.el7             base             13 k
 python-ipaddress                               noarch            1.0.16-2.el7              base             34 k

Transaction Summary
==================================================================================================================
Install  1 Package (+3 Dependent packages)

Total download size: 450 k
Installed size: 2.2 M
Downloading packages:
(1/4): python-backports-1.0-8.el7.x86_64.rpm                                               | 5.8 kB  00:00:00     
(2/4): python-backports-ssl_match_hostname-3.5.0.1-1.el7.noarch.rpm                        |  13 kB  00:00:00     
(3/4): python-ipaddress-1.0.16-2.el7.noarch.rpm                                            |  34 kB  00:00:00     
(4/4): python-setuptools-0.9.8-7.el7.noarch.rpm                                            | 397 kB  00:00:00     
------------------------------------------------------------------------------------------------------------------
Total                                                                             1.4 MB/s | 450 kB  00:00:00     
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Installing : python-backports-1.0-8.el7.x86_64                                                              1/4 
  Installing : python-ipaddress-1.0.16-2.el7.noarch                                                           2/4 
  Installing : python-backports-ssl_match_hostname-3.5.0.1-1.el7.noarch                                       3/4 
  Installing : python-setuptools-0.9.8-7.el7.noarch                                                           4/4 
  Verifying  : python-ipaddress-1.0.16-2.el7.noarch                                                           1/4 
  Verifying  : python-setuptools-0.9.8-7.el7.noarch                                                           2/4 
  Verifying  : python-backports-ssl_match_hostname-3.5.0.1-1.el7.noarch                                       3/4 
  Verifying  : python-backports-1.0-8.el7.x86_64                                                              4/4 

Installed:
  python-setuptools.noarch 0:0.9.8-7.el7                                                                          

Dependency Installed:
  python-backports.x86_64 0:1.0-8.el7           python-backports-ssl_match_hostname.noarch 0:3.5.0.1-1.el7       
  python-ipaddress.noarch 0:1.0.16-2.el7       

Complete!
```



## 6. 创建 Ceph 存储集群

**如无特殊说明，本节所有操作都在 ceph-node-0 节点执行**

### 6.1. 创建集群配置目录

> **执行任务命令**

```shell
mkdir k8s-cluster
cd k8s-cluster
```

### 6.2. 初始化 Ceph 集群

> **执行任务命令**

- 创建集群

```shell
ceph-deploy new ceph-node-0
```

- 添加配置项到配置文件

```shell
# 可选添加，如果不加，后续再添加新的mon节点会报错
echo "public network = 192.168.9.0/24" >> ceph.conf
```

- 安装软件包

```shell
# 官方的正常命令是
# ceph-deploy install ceph-node-0 ceph-node-1 ceph-node-2
# 但是这种方式最多能安装到 nautilus 版，所以需要采用手工安装的方式
# 在Ansible控制服务器上批量手工安装

ansible ceph -m shell -a 'yum install -y yum-plugin-priorities'
ansible ceph -m shell -a 'yum install -y ceph'
```

- 部署初始 monitor 服务

```shell
ceph-deploy mon create-initial
```

- 复制配置文件和 admin key 到其他节点

```shell
ceph-deploy admin ceph-node-0 ceph-node-1 ceph-node-2
```

- 部署 manager daemon(可选)

```shell
ceph-deploy mgr create ceph-node-0
```

- 创建 OSD

```shell
ceph-deploy osd create --data /dev/sdb ceph-node-0
ceph-deploy osd create --data /dev/sdb ceph-node-1
ceph-deploy osd create --data /dev/sdb ceph-node-2
```

- 验证集群状态

```shell
ceph -s
ceph health

# health 有WARN信息如下
[root@ceph-node-0 k8s-cluster]# ceph health
HEALTH_WARN mon is allowing insecure global_id reclaim; Module 'restful' has failed dependency: No module named 'pecan'
```

- 解决 ceph health WARN 问题

```shell
# mon is allowing insecure global_id reclaim 
ceph config set mon auth_allow_insecure_global_id_reclaim false

# Module 'restful' has failed dependency: No module named 'pecan'
# 三个 Ceph 节点都要执行(可以使用Ansible批量执行)
pip3 install werkzeug pecan
systemctl restart ceph-mon.target
systemctl restart ceph-mgr.target
```

- 再次验证集群状态

```shell
ceph -s
```

> **正确执行输出参考**

```shell
# 创建集群
[root@ceph-node-0 k8s-cluster]# ceph-deploy new ceph-node-0
[ceph_deploy.conf][DEBUG ] found configuration file at: /root/.cephdeploy.conf
[ceph_deploy.cli][INFO  ] Invoked (2.0.1): /usr/bin/ceph-deploy new ceph-node-0
[ceph_deploy.cli][INFO  ] ceph-deploy options:
[ceph_deploy.cli][INFO  ]  username                      : None
[ceph_deploy.cli][INFO  ]  func                          : <function new at 0x7f93b59dede8>
[ceph_deploy.cli][INFO  ]  verbose                       : False
[ceph_deploy.cli][INFO  ]  overwrite_conf                : False
[ceph_deploy.cli][INFO  ]  quiet                         : False
[ceph_deploy.cli][INFO  ]  cd_conf                       : <ceph_deploy.conf.cephdeploy.Conf instance at 0x7f93b51585a8>
[ceph_deploy.cli][INFO  ]  cluster                       : ceph
[ceph_deploy.cli][INFO  ]  ssh_copykey                   : True
[ceph_deploy.cli][INFO  ]  mon                           : ['ceph-node-0']
[ceph_deploy.cli][INFO  ]  public_network                : None
[ceph_deploy.cli][INFO  ]  ceph_conf                     : None
[ceph_deploy.cli][INFO  ]  cluster_network               : None
[ceph_deploy.cli][INFO  ]  default_release               : False
[ceph_deploy.cli][INFO  ]  fsid                          : None
[ceph_deploy.new][DEBUG ] Creating new cluster named ceph
[ceph_deploy.new][INFO  ] making sure passwordless SSH succeeds
[ceph-node-0][DEBUG ] connected to host: ceph-node-0 
[ceph-node-0][DEBUG ] detect platform information from remote host
[ceph-node-0][DEBUG ] detect machine type
[ceph-node-0][DEBUG ] find the location of an executable
[ceph-node-0][INFO  ] Running command: /usr/sbin/ip link show
[ceph-node-0][INFO  ] Running command: /usr/sbin/ip addr show
[ceph-node-0][DEBUG ] IP addresses found: [u'192.168.9.85']
[ceph_deploy.new][DEBUG ] Resolving host ceph-node-0
[ceph_deploy.new][DEBUG ] Monitor ceph-node-0 at 192.168.9.85
[ceph_deploy.new][DEBUG ] Monitor initial members are ['ceph-node-0']
[ceph_deploy.new][DEBUG ] Monitor addrs are ['192.168.9.85']
[ceph_deploy.new][DEBUG ] Creating a random mon key...
[ceph_deploy.new][DEBUG ] Writing monitor keyring to ceph.mon.keyring...
[ceph_deploy.new][DEBUG ] Writing initial config to ceph.conf...

# 部署初始monitor服务
[root@ceph-node-0 k8s-cluster]# ceph-deploy mon create-initial
[ceph_deploy.conf][DEBUG ] found configuration file at: /root/.cephdeploy.conf
[ceph_deploy.cli][INFO  ] Invoked (2.0.1): /usr/bin/ceph-deploy mon create-initial
[ceph_deploy.cli][INFO  ] ceph-deploy options:
[ceph_deploy.cli][INFO  ]  username                      : None
[ceph_deploy.cli][INFO  ]  verbose                       : False
[ceph_deploy.cli][INFO  ]  overwrite_conf                : False
[ceph_deploy.cli][INFO  ]  subcommand                    : create-initial
[ceph_deploy.cli][INFO  ]  quiet                         : False
[ceph_deploy.cli][INFO  ]  cd_conf                       : <ceph_deploy.conf.cephdeploy.Conf instance at 0x7fe0d4570e18>
[ceph_deploy.cli][INFO  ]  cluster                       : ceph
[ceph_deploy.cli][INFO  ]  func                          : <function mon at 0x7fe0d4555410>
[ceph_deploy.cli][INFO  ]  ceph_conf                     : None
[ceph_deploy.cli][INFO  ]  default_release               : False
[ceph_deploy.cli][INFO  ]  keyrings                      : None
[ceph_deploy.mon][DEBUG ] Deploying mon, cluster ceph hosts ceph-node-0
[ceph_deploy.mon][DEBUG ] detecting platform for host ceph-node-0 ...
[ceph-node-0][DEBUG ] connected to host: ceph-node-0 
[ceph-node-0][DEBUG ] detect platform information from remote host
[ceph-node-0][DEBUG ] detect machine type
[ceph-node-0][DEBUG ] find the location of an executable
[ceph_deploy.mon][INFO  ] distro info: CentOS Linux 7.9.2009 Core
[ceph-node-0][DEBUG ] determining if provided host has same hostname in remote
[ceph-node-0][DEBUG ] get remote short hostname
[ceph-node-0][DEBUG ] deploying mon to ceph-node-0
[ceph-node-0][DEBUG ] get remote short hostname
[ceph-node-0][DEBUG ] remote hostname: ceph-node-0
[ceph-node-0][DEBUG ] write cluster configuration to /etc/ceph/{cluster}.conf
[ceph-node-0][DEBUG ] create the mon path if it does not exist
[ceph-node-0][DEBUG ] checking for done path: /var/lib/ceph/mon/ceph-ceph-node-0/done
[ceph-node-0][DEBUG ] done path does not exist: /var/lib/ceph/mon/ceph-ceph-node-0/done
[ceph-node-0][INFO  ] creating keyring file: /var/lib/ceph/tmp/ceph-ceph-node-0.mon.keyring
[ceph-node-0][DEBUG ] create the monitor keyring file
[ceph-node-0][INFO  ] Running command: ceph-mon --cluster ceph --mkfs -i ceph-node-0 --keyring /var/lib/ceph/tmp/ceph-ceph-node-0.mon.keyring --setuser 167 --setgroup 167
[ceph-node-0][INFO  ] unlinking keyring file /var/lib/ceph/tmp/ceph-ceph-node-0.mon.keyring
[ceph-node-0][DEBUG ] create a done file to avoid re-doing the mon deployment
[ceph-node-0][DEBUG ] create the init path if it does not exist
[ceph-node-0][INFO  ] Running command: systemctl enable ceph.target
[ceph-node-0][INFO  ] Running command: systemctl enable ceph-mon@ceph-node-0
[ceph-node-0][WARNIN] Created symlink from /etc/systemd/system/ceph-mon.target.wants/ceph-mon@ceph-node-0.service to /usr/lib/systemd/system/ceph-mon@.service.
[ceph-node-0][INFO  ] Running command: systemctl start ceph-mon@ceph-node-0
[ceph-node-0][INFO  ] Running command: ceph --cluster=ceph --admin-daemon /var/run/ceph/ceph-mon.ceph-node-0.asok mon_status
[ceph-node-0][DEBUG ] ********************************************************************************
[ceph-node-0][DEBUG ] status for monitor: mon.ceph-node-0
[ceph-node-0][DEBUG ] {
[ceph-node-0][DEBUG ]   "election_epoch": 3, 
[ceph-node-0][DEBUG ]   "extra_probe_peers": [], 
[ceph-node-0][DEBUG ]   "feature_map": {
[ceph-node-0][DEBUG ]     "mon": [
[ceph-node-0][DEBUG ]       {
[ceph-node-0][DEBUG ]         "features": "0x3f01cfb8ffedffff", 
[ceph-node-0][DEBUG ]         "num": 1, 
[ceph-node-0][DEBUG ]         "release": "luminous"
[ceph-node-0][DEBUG ]       }
[ceph-node-0][DEBUG ]     ]
[ceph-node-0][DEBUG ]   }, 
[ceph-node-0][DEBUG ]   "features": {
[ceph-node-0][DEBUG ]     "quorum_con": "4540138292840890367", 
[ceph-node-0][DEBUG ]     "quorum_mon": [
[ceph-node-0][DEBUG ]       "kraken", 
[ceph-node-0][DEBUG ]       "luminous", 
[ceph-node-0][DEBUG ]       "mimic", 
[ceph-node-0][DEBUG ]       "osdmap-prune", 
[ceph-node-0][DEBUG ]       "nautilus", 
[ceph-node-0][DEBUG ]       "octopus"
[ceph-node-0][DEBUG ]     ], 
[ceph-node-0][DEBUG ]     "required_con": "2449958747315978244", 
[ceph-node-0][DEBUG ]     "required_mon": [
[ceph-node-0][DEBUG ]       "kraken", 
[ceph-node-0][DEBUG ]       "luminous", 
[ceph-node-0][DEBUG ]       "mimic", 
[ceph-node-0][DEBUG ]       "osdmap-prune", 
[ceph-node-0][DEBUG ]       "nautilus", 
[ceph-node-0][DEBUG ]       "octopus"
[ceph-node-0][DEBUG ]     ]
[ceph-node-0][DEBUG ]   }, 
[ceph-node-0][DEBUG ]   "monmap": {
[ceph-node-0][DEBUG ]     "created": "2022-05-24T03:18:55.637040Z", 
[ceph-node-0][DEBUG ]     "epoch": 1, 
[ceph-node-0][DEBUG ]     "features": {
[ceph-node-0][DEBUG ]       "optional": [], 
[ceph-node-0][DEBUG ]       "persistent": [
[ceph-node-0][DEBUG ]         "kraken", 
[ceph-node-0][DEBUG ]         "luminous", 
[ceph-node-0][DEBUG ]         "mimic", 
[ceph-node-0][DEBUG ]         "osdmap-prune", 
[ceph-node-0][DEBUG ]         "nautilus", 
[ceph-node-0][DEBUG ]         "octopus"
[ceph-node-0][DEBUG ]       ]
[ceph-node-0][DEBUG ]     }, 
[ceph-node-0][DEBUG ]     "fsid": "a614e796-6875-4134-a735-2e4db541bba8", 
[ceph-node-0][DEBUG ]     "min_mon_release": 15, 
[ceph-node-0][DEBUG ]     "min_mon_release_name": "octopus", 
[ceph-node-0][DEBUG ]     "modified": "2022-05-24T03:18:55.637040Z", 
[ceph-node-0][DEBUG ]     "mons": [
[ceph-node-0][DEBUG ]       {
[ceph-node-0][DEBUG ]         "addr": "192.168.9.85:6789/0", 
[ceph-node-0][DEBUG ]         "name": "ceph-node-0", 
[ceph-node-0][DEBUG ]         "priority": 0, 
[ceph-node-0][DEBUG ]         "public_addr": "192.168.9.85:6789/0", 
[ceph-node-0][DEBUG ]         "public_addrs": {
[ceph-node-0][DEBUG ]           "addrvec": [
[ceph-node-0][DEBUG ]             {
[ceph-node-0][DEBUG ]               "addr": "192.168.9.85:3300", 
[ceph-node-0][DEBUG ]               "nonce": 0, 
[ceph-node-0][DEBUG ]               "type": "v2"
[ceph-node-0][DEBUG ]             }, 
[ceph-node-0][DEBUG ]             {
[ceph-node-0][DEBUG ]               "addr": "192.168.9.85:6789", 
[ceph-node-0][DEBUG ]               "nonce": 0, 
[ceph-node-0][DEBUG ]               "type": "v1"
[ceph-node-0][DEBUG ]             }
[ceph-node-0][DEBUG ]           ]
[ceph-node-0][DEBUG ]         }, 
[ceph-node-0][DEBUG ]         "rank": 0, 
[ceph-node-0][DEBUG ]         "weight": 0
[ceph-node-0][DEBUG ]       }
[ceph-node-0][DEBUG ]     ]
[ceph-node-0][DEBUG ]   }, 
[ceph-node-0][DEBUG ]   "name": "ceph-node-0", 
[ceph-node-0][DEBUG ]   "outside_quorum": [], 
[ceph-node-0][DEBUG ]   "quorum": [
[ceph-node-0][DEBUG ]     0
[ceph-node-0][DEBUG ]   ], 
[ceph-node-0][DEBUG ]   "quorum_age": 2, 
[ceph-node-0][DEBUG ]   "rank": 0, 
[ceph-node-0][DEBUG ]   "state": "leader", 
[ceph-node-0][DEBUG ]   "sync_provider": []
[ceph-node-0][DEBUG ] }
[ceph-node-0][DEBUG ] ********************************************************************************
[ceph-node-0][INFO  ] monitor: mon.ceph-node-0 is running
[ceph-node-0][INFO  ] Running command: ceph --cluster=ceph --admin-daemon /var/run/ceph/ceph-mon.ceph-node-0.asok mon_status
[ceph_deploy.mon][INFO  ] processing monitor mon.ceph-node-0
[ceph-node-0][DEBUG ] connected to host: ceph-node-0 
[ceph-node-0][DEBUG ] detect platform information from remote host
[ceph-node-0][DEBUG ] detect machine type
[ceph-node-0][DEBUG ] find the location of an executable
[ceph-node-0][INFO  ] Running command: ceph --cluster=ceph --admin-daemon /var/run/ceph/ceph-mon.ceph-node-0.asok mon_status
[ceph_deploy.mon][INFO  ] mon.ceph-node-0 monitor has reached quorum!
[ceph_deploy.mon][INFO  ] all initial monitors are running and have formed quorum
[ceph_deploy.mon][INFO  ] Running gatherkeys...
[ceph_deploy.gatherkeys][INFO  ] Storing keys in temp directory /tmp/tmpWITFug
[ceph-node-0][DEBUG ] connected to host: ceph-node-0 
[ceph-node-0][DEBUG ] detect platform information from remote host
[ceph-node-0][DEBUG ] detect machine type
[ceph-node-0][DEBUG ] get remote short hostname
[ceph-node-0][DEBUG ] fetch remote file
[ceph-node-0][INFO  ] Running command: /usr/bin/ceph --connect-timeout=25 --cluster=ceph --admin-daemon=/var/run/ceph/ceph-mon.ceph-node-0.asok mon_status
[ceph-node-0][INFO  ] Running command: /usr/bin/ceph --connect-timeout=25 --cluster=ceph --name mon. --keyring=/var/lib/ceph/mon/ceph-ceph-node-0/keyring auth get client.admin
[ceph-node-0][INFO  ] Running command: /usr/bin/ceph --connect-timeout=25 --cluster=ceph --name mon. --keyring=/var/lib/ceph/mon/ceph-ceph-node-0/keyring auth get client.bootstrap-mds
[ceph-node-0][INFO  ] Running command: /usr/bin/ceph --connect-timeout=25 --cluster=ceph --name mon. --keyring=/var/lib/ceph/mon/ceph-ceph-node-0/keyring auth get client.bootstrap-mgr
[ceph-node-0][INFO  ] Running command: /usr/bin/ceph --connect-timeout=25 --cluster=ceph --name mon. --keyring=/var/lib/ceph/mon/ceph-ceph-node-0/keyring auth get client.bootstrap-osd
[ceph-node-0][INFO  ] Running command: /usr/bin/ceph --connect-timeout=25 --cluster=ceph --name mon. --keyring=/var/lib/ceph/mon/ceph-ceph-node-0/keyring auth get client.bootstrap-rgw
[ceph_deploy.gatherkeys][INFO  ] Storing ceph.client.admin.keyring
[ceph_deploy.gatherkeys][INFO  ] Storing ceph.bootstrap-mds.keyring
[ceph_deploy.gatherkeys][INFO  ] Storing ceph.bootstrap-mgr.keyring
[ceph_deploy.gatherkeys][INFO  ] keyring 'ceph.mon.keyring' already exists
[ceph_deploy.gatherkeys][INFO  ] Storing ceph.bootstrap-osd.keyring
[ceph_deploy.gatherkeys][INFO  ] Storing ceph.bootstrap-rgw.keyring
[ceph_deploy.gatherkeys][INFO  ] Destroy temp directory /tmp/tmpWITFug


# 部署初始monitor服务
[root@ceph-node-0 k8s-cluster]# ceph-deploy admin ceph-node-0 ceph-node-1 ceph-node-2
[ceph_deploy.conf][DEBUG ] found configuration file at: /root/.cephdeploy.conf
[ceph_deploy.cli][INFO  ] Invoked (2.0.1): /usr/bin/ceph-deploy admin ceph-node-0 ceph-node-1 ceph-node-2
[ceph_deploy.cli][INFO  ] ceph-deploy options:
[ceph_deploy.cli][INFO  ]  username                      : None
[ceph_deploy.cli][INFO  ]  verbose                       : False
[ceph_deploy.cli][INFO  ]  overwrite_conf                : False
[ceph_deploy.cli][INFO  ]  quiet                         : False
[ceph_deploy.cli][INFO  ]  cd_conf                       : <ceph_deploy.conf.cephdeploy.Conf instance at 0x7f1f54cca440>
[ceph_deploy.cli][INFO  ]  cluster                       : ceph
[ceph_deploy.cli][INFO  ]  client                        : ['ceph-node-0', 'ceph-node-1', 'ceph-node-2']
[ceph_deploy.cli][INFO  ]  func                          : <function admin at 0x7f1f557e9230>
[ceph_deploy.cli][INFO  ]  ceph_conf                     : None
[ceph_deploy.cli][INFO  ]  default_release               : False
[ceph_deploy.admin][DEBUG ] Pushing admin keys and conf to ceph-node-0
[ceph-node-0][DEBUG ] connected to host: ceph-node-0 
[ceph-node-0][DEBUG ] detect platform information from remote host
[ceph-node-0][DEBUG ] detect machine type
[ceph-node-0][DEBUG ] write cluster configuration to /etc/ceph/{cluster}.conf
[ceph_deploy.admin][DEBUG ] Pushing admin keys and conf to ceph-node-1
The authenticity of host 'ceph-node-1 (192.168.9.86)' can't be established.
ECDSA key fingerprint is SHA256:/0HffAhG/oTriuqo7KjAdf/YQNoFSbwYk/SAVUYH6Fs.
ECDSA key fingerprint is MD5:34:67:af:a6:50:c0:5f:33:07:45:11:e8:b8:52:e2:ae.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added 'ceph-node-1' (ECDSA) to the list of known hosts.
[ceph-node-1][DEBUG ] connected to host: ceph-node-1 
[ceph-node-1][DEBUG ] detect platform information from remote host
[ceph-node-1][DEBUG ] detect machine type
[ceph-node-1][DEBUG ] write cluster configuration to /etc/ceph/{cluster}.conf
[ceph_deploy.admin][DEBUG ] Pushing admin keys and conf to ceph-node-2
The authenticity of host 'ceph-node-2 (192.168.9.87)' can't be established.
ECDSA key fingerprint is SHA256:/0HffAhG/oTriuqo7KjAdf/YQNoFSbwYk/SAVUYH6Fs.
ECDSA key fingerprint is MD5:34:67:af:a6:50:c0:5f:33:07:45:11:e8:b8:52:e2:ae.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added 'ceph-node-2' (ECDSA) to the list of known hosts.
[ceph-node-2][DEBUG ] connected to host: ceph-node-2 
[ceph-node-2][DEBUG ] detect platform information from remote host
[ceph-node-2][DEBUG ] detect machine type
[ceph-node-2][DEBUG ] write cluster configuration to /etc/ceph/{cluster}.conf

# 部署manager daemon
[root@ceph-node-0 k8s-cluster]# ceph-deploy mgr create ceph-node-0
[ceph_deploy.conf][DEBUG ] found configuration file at: /root/.cephdeploy.conf
[ceph_deploy.cli][INFO  ] Invoked (2.0.1): /usr/bin/ceph-deploy mgr create ceph-node-0
[ceph_deploy.cli][INFO  ] ceph-deploy options:
[ceph_deploy.cli][INFO  ]  username                      : None
[ceph_deploy.cli][INFO  ]  verbose                       : False
[ceph_deploy.cli][INFO  ]  mgr                           : [('ceph-node-0', 'ceph-node-0')]
[ceph_deploy.cli][INFO  ]  overwrite_conf                : False
[ceph_deploy.cli][INFO  ]  subcommand                    : create
[ceph_deploy.cli][INFO  ]  quiet                         : False
[ceph_deploy.cli][INFO  ]  cd_conf                       : <ceph_deploy.conf.cephdeploy.Conf instance at 0x7f39544c3998>
[ceph_deploy.cli][INFO  ]  cluster                       : ceph
[ceph_deploy.cli][INFO  ]  func                          : <function mgr at 0x7f3954d35140>
[ceph_deploy.cli][INFO  ]  ceph_conf                     : None
[ceph_deploy.cli][INFO  ]  default_release               : False
[ceph_deploy.mgr][DEBUG ] Deploying mgr, cluster ceph hosts ceph-node-0:ceph-node-0
[ceph-node-0][DEBUG ] connected to host: ceph-node-0 
[ceph-node-0][DEBUG ] detect platform information from remote host
[ceph-node-0][DEBUG ] detect machine type
[ceph_deploy.mgr][INFO  ] Distro info: CentOS Linux 7.9.2009 Core
[ceph_deploy.mgr][DEBUG ] remote host will use systemd
[ceph_deploy.mgr][DEBUG ] deploying mgr bootstrap to ceph-node-0
[ceph-node-0][DEBUG ] write cluster configuration to /etc/ceph/{cluster}.conf
[ceph-node-0][WARNIN] mgr keyring does not exist yet, creating one
[ceph-node-0][DEBUG ] create a keyring file
[ceph-node-0][DEBUG ] create path recursively if it doesn't exist
[ceph-node-0][INFO  ] Running command: ceph --cluster ceph --name client.bootstrap-mgr --keyring /var/lib/ceph/bootstrap-mgr/ceph.keyring auth get-or-create mgr.ceph-node-0 mon allow profile mgr osd allow * mds allow * -o /var/lib/ceph/mgr/ceph-ceph-node-0/keyring
[ceph-node-0][INFO  ] Running command: systemctl enable ceph-mgr@ceph-node-0
[ceph-node-0][WARNIN] Created symlink from /etc/systemd/system/ceph-mgr.target.wants/ceph-mgr@ceph-node-0.service to /usr/lib/systemd/system/ceph-mgr@.service.
[ceph-node-0][INFO  ] Running command: systemctl start ceph-mgr@ceph-node-0
[ceph-node-0][INFO  ] Running command: systemctl enable ceph.target

# 创建OSD
[root@ceph-node-0 k8s-cluster]# ceph-deploy osd create --data /dev/sdb ceph-node-0
[ceph_deploy.conf][DEBUG ] found configuration file at: /root/.cephdeploy.conf
[ceph_deploy.cli][INFO  ] Invoked (2.0.1): /usr/bin/ceph-deploy osd create --data /dev/sdb ceph-node-0
[ceph_deploy.cli][INFO  ] ceph-deploy options:
[ceph_deploy.cli][INFO  ]  verbose                       : False
[ceph_deploy.cli][INFO  ]  bluestore                     : None
[ceph_deploy.cli][INFO  ]  cd_conf                       : <ceph_deploy.conf.cephdeploy.Conf instance at 0x7fb25a9cbf38>
[ceph_deploy.cli][INFO  ]  cluster                       : ceph
[ceph_deploy.cli][INFO  ]  fs_type                       : xfs
[ceph_deploy.cli][INFO  ]  block_wal                     : None
[ceph_deploy.cli][INFO  ]  default_release               : False
[ceph_deploy.cli][INFO  ]  username                      : None
[ceph_deploy.cli][INFO  ]  journal                       : None
[ceph_deploy.cli][INFO  ]  subcommand                    : create
[ceph_deploy.cli][INFO  ]  host                          : ceph-node-0
[ceph_deploy.cli][INFO  ]  filestore                     : None
[ceph_deploy.cli][INFO  ]  func                          : <function osd at 0x7fb25a98d8c0>
[ceph_deploy.cli][INFO  ]  ceph_conf                     : None
[ceph_deploy.cli][INFO  ]  zap_disk                      : False
[ceph_deploy.cli][INFO  ]  data                          : /dev/sdb
[ceph_deploy.cli][INFO  ]  block_db                      : None
[ceph_deploy.cli][INFO  ]  dmcrypt                       : False
[ceph_deploy.cli][INFO  ]  overwrite_conf                : False
[ceph_deploy.cli][INFO  ]  dmcrypt_key_dir               : /etc/ceph/dmcrypt-keys
[ceph_deploy.cli][INFO  ]  quiet                         : False
[ceph_deploy.cli][INFO  ]  debug                         : False
[ceph_deploy.osd][DEBUG ] Creating OSD on cluster ceph with data device /dev/sdb
[ceph-node-0][DEBUG ] connected to host: ceph-node-0 
[ceph-node-0][DEBUG ] detect platform information from remote host
[ceph-node-0][DEBUG ] detect machine type
[ceph-node-0][DEBUG ] find the location of an executable
[ceph_deploy.osd][INFO  ] Distro info: CentOS Linux 7.9.2009 Core
[ceph_deploy.osd][DEBUG ] Deploying osd to ceph-node-0
[ceph-node-0][DEBUG ] write cluster configuration to /etc/ceph/{cluster}.conf
[ceph-node-0][WARNIN] osd keyring does not exist yet, creating one
[ceph-node-0][DEBUG ] create a keyring file
[ceph-node-0][DEBUG ] find the location of an executable
[ceph-node-0][INFO  ] Running command: /usr/sbin/ceph-volume --cluster ceph lvm create --bluestore --data /dev/sdb
[ceph-node-0][WARNIN] Running command: /usr/bin/ceph-authtool --gen-print-key
[ceph-node-0][WARNIN] Running command: /usr/bin/ceph --cluster ceph --name client.bootstrap-osd --keyring /var/lib/ceph/bootstrap-osd/ceph.keyring -i - osd new a8238416-bd31-466f-b3ac-d10f497bd323
[ceph-node-0][WARNIN] Running command: vgcreate --force --yes ceph-9c8b5a7b-eb15-4737-9df4-48fcfd00d95b /dev/sdb
[ceph-node-0][WARNIN]  stdout: Physical volume "/dev/sdb" successfully created.
[ceph-node-0][WARNIN]  stdout: Volume group "ceph-9c8b5a7b-eb15-4737-9df4-48fcfd00d95b" successfully created
[ceph-node-0][WARNIN] Running command: lvcreate --yes -l 25599 -n osd-block-a8238416-bd31-466f-b3ac-d10f497bd323 ceph-9c8b5a7b-eb15-4737-9df4-48fcfd00d95b
[ceph-node-0][WARNIN]  stdout: Logical volume "osd-block-a8238416-bd31-466f-b3ac-d10f497bd323" created.
[ceph-node-0][WARNIN] Running command: /usr/bin/ceph-authtool --gen-print-key
[ceph-node-0][WARNIN] Running command: /usr/bin/mount -t tmpfs tmpfs /var/lib/ceph/osd/ceph-0
[ceph-node-0][WARNIN] Running command: /usr/bin/chown -h ceph:ceph /dev/ceph-9c8b5a7b-eb15-4737-9df4-48fcfd00d95b/osd-block-a8238416-bd31-466f-b3ac-d10f497bd323
[ceph-node-0][WARNIN] Running command: /usr/bin/chown -R ceph:ceph /dev/dm-2
[ceph-node-0][WARNIN] Running command: /usr/bin/ln -s /dev/ceph-9c8b5a7b-eb15-4737-9df4-48fcfd00d95b/osd-block-a8238416-bd31-466f-b3ac-d10f497bd323 /var/lib/ceph/osd/ceph-0/block
[ceph-node-0][WARNIN] Running command: /usr/bin/ceph --cluster ceph --name client.bootstrap-osd --keyring /var/lib/ceph/bootstrap-osd/ceph.keyring mon getmap -o /var/lib/ceph/osd/ceph-0/activate.monmap
[ceph-node-0][WARNIN]  stderr: 2022-05-24T13:34:17.775+0800 7f1b743d4700 -1 auth: unable to find a keyring on /etc/ceph/ceph.client.bootstrap-osd.keyring,/etc/ceph/ceph.keyring,/etc/ceph/keyring,/etc/ceph/keyring.bin,: (2) No such file or directory
[ceph-node-0][WARNIN] 2022-05-24T13:34:17.775+0800 7f1b743d4700 -1 AuthRegistry(0x7f1b6c0592f0) no keyring found at /etc/ceph/ceph.client.bootstrap-osd.keyring,/etc/ceph/ceph.keyring,/etc/ceph/keyring,/etc/ceph/keyring.bin,, disabling cephx
[ceph-node-0][WARNIN]  stderr: got monmap epoch 1
[ceph-node-0][WARNIN] Running command: /usr/bin/ceph-authtool /var/lib/ceph/osd/ceph-0/keyring --create-keyring --name osd.0 --add-key AQBYboxi1LjtCRAAp+JD29cGvks4UNQftMqYCQ==
[ceph-node-0][WARNIN]  stdout: creating /var/lib/ceph/osd/ceph-0/keyring
[ceph-node-0][WARNIN] added entity osd.0 auth(key=AQBYboxi1LjtCRAAp+JD29cGvks4UNQftMqYCQ==)
[ceph-node-0][WARNIN] Running command: /usr/bin/chown -R ceph:ceph /var/lib/ceph/osd/ceph-0/keyring
[ceph-node-0][WARNIN] Running command: /usr/bin/chown -R ceph:ceph /var/lib/ceph/osd/ceph-0/
[ceph-node-0][WARNIN] Running command: /usr/bin/ceph-osd --cluster ceph --osd-objectstore bluestore --mkfs -i 0 --monmap /var/lib/ceph/osd/ceph-0/activate.monmap --keyfile - --osd-data /var/lib/ceph/osd/ceph-0/ --osd-uuid a8238416-bd31-466f-b3ac-d10f497bd323 --setuser ceph --setgroup ceph
[ceph-node-0][WARNIN]  stderr: 2022-05-24T13:34:19.048+0800 7f13c3f1cbc0 -1 bluestore(/var/lib/ceph/osd/ceph-0/) _read_fsid unparsable uuid
[ceph-node-0][WARNIN]  stderr: 2022-05-24T13:34:19.108+0800 7f13c3f1cbc0 -1 freelist read_size_meta_from_db missing size meta in DB
[ceph-node-0][WARNIN] --> ceph-volume lvm prepare successful for: /dev/sdb
[ceph-node-0][WARNIN] Running command: /usr/bin/chown -R ceph:ceph /var/lib/ceph/osd/ceph-0
[ceph-node-0][WARNIN] Running command: /usr/bin/ceph-bluestore-tool --cluster=ceph prime-osd-dir --dev /dev/ceph-9c8b5a7b-eb15-4737-9df4-48fcfd00d95b/osd-block-a8238416-bd31-466f-b3ac-d10f497bd323 --path /var/lib/ceph/osd/ceph-0 --no-mon-config
[ceph-node-0][WARNIN] Running command: /usr/bin/ln -snf /dev/ceph-9c8b5a7b-eb15-4737-9df4-48fcfd00d95b/osd-block-a8238416-bd31-466f-b3ac-d10f497bd323 /var/lib/ceph/osd/ceph-0/block
[ceph-node-0][WARNIN] Running command: /usr/bin/chown -h ceph:ceph /var/lib/ceph/osd/ceph-0/block
[ceph-node-0][WARNIN] Running command: /usr/bin/chown -R ceph:ceph /dev/dm-2
[ceph-node-0][WARNIN] Running command: /usr/bin/chown -R ceph:ceph /var/lib/ceph/osd/ceph-0
[ceph-node-0][WARNIN] Running command: /usr/bin/systemctl enable ceph-volume@lvm-0-a8238416-bd31-466f-b3ac-d10f497bd323
[ceph-node-0][WARNIN]  stderr: Created symlink from /etc/systemd/system/multi-user.target.wants/ceph-volume@lvm-0-a8238416-bd31-466f-b3ac-d10f497bd323.service to /usr/lib/systemd/system/ceph-volume@.service.
[ceph-node-0][WARNIN] Running command: /usr/bin/systemctl enable --runtime ceph-osd@0
[ceph-node-0][WARNIN]  stderr: Created symlink from /run/systemd/system/ceph-osd.target.wants/ceph-osd@0.service to /usr/lib/systemd/system/ceph-osd@.service.
[ceph-node-0][WARNIN] Running command: /usr/bin/systemctl start ceph-osd@0
[ceph-node-0][WARNIN] --> ceph-volume lvm activate successful for osd ID: 0
[ceph-node-0][WARNIN] --> ceph-volume lvm create successful for: /dev/sdb
[ceph-node-0][INFO  ] checking OSD status...
[ceph-node-0][DEBUG ] find the location of an executable
[ceph-node-0][INFO  ] Running command: /bin/ceph --cluster=ceph osd stat --format=json
[ceph_deploy.osd][DEBUG ] Host ceph-node-0 is now ready for osd use.

# 验证集群状态
[root@ceph-node-0 k8s-cluster]# ceph health
HEALTH_WARN mon is allowing insecure global_id reclaim; Module 'restful' has failed dependency: No module named 'pecan'

[root@ceph-node-0 k8s-cluster]# ceph -s
  cluster:
    id:     a614e796-6875-4134-a735-2e4db541bba8
    health: HEALTH_WARN
            mon is allowing insecure global_id reclaim
            Module 'restful' has failed dependency: No module named 'pecan'
 
  services:
    mon: 1 daemons, quorum ceph-node-0 (age 2h)
    mgr: ceph-node-0(active, since 2h)
    osd: 3 osds: 3 up (since 4m), 3 in (since 4m)
 
  data:
    pools:   1 pools, 1 pgs
    objects: 0 objects, 0 B
    usage:   3.0 GiB used, 297 GiB / 300 GiB avail
    pgs:     1 active+clean
 
 # 解决ceph health WARN问题
 [root@ceph-node-0 k8s-cluster]# pip3 install werkzeug pecan
WARNING: Running pip install with root privileges is generally not a good idea. Try `pip3 install --user` instead.
Collecting werkzeug
  Retrying (Retry(total=4, connect=None, read=None, redirect=None, status=None)) after connection broken by 'NewConnectionError('<pip._vendor.urllib3.connection.VerifiedHTTPSConnection object at 0x7f6053068898>: Failed to establish a new connection: [Errno 101] Network is unreachable',)': /simple/werkzeug/
  Retrying (Retry(total=3, connect=None, read=None, redirect=None, status=None)) after connection broken by 'NewConnectionError('<pip._vendor.urllib3.connection.VerifiedHTTPSConnection object at 0x7f6053068748>: Failed to establish a new connection: [Errno 101] Network is unreachable',)': /simple/werkzeug/
  Retrying (Retry(total=2, connect=None, read=None, redirect=None, status=None)) after connection broken by 'NewConnectionError('<pip._vendor.urllib3.connection.VerifiedHTTPSConnection object at 0x7f60530686a0>: Failed to establish a new connection: [Errno 101] Network is unreachable',)': /simple/werkzeug/
  Retrying (Retry(total=1, connect=None, read=None, redirect=None, status=None)) after connection broken by 'NewConnectionError('<pip._vendor.urllib3.connection.VerifiedHTTPSConnection object at 0x7f6053068b70>: Failed to establish a new connection: [Errno 101] Network is unreachable',)': /simple/werkzeug/
  Downloading https://files.pythonhosted.org/packages/f4/f3/22afbdb20cc4654b10c98043414a14057cd27fdba9d4ae61cea596000ba2/Werkzeug-2.0.3-py3-none-any.whl (289kB)
    100% |████████████████████████████████| 296kB 116kB/s 
Collecting pecan
  Downloading https://files.pythonhosted.org/packages/2a/cc/d7c9c62b7af117d803ed1441191a2297fd8ee0f4a6fbedaefb46c736ba52/pecan-1.4.1.tar.gz (124kB)
    100% |████████████████████████████████| 133kB 17kB/s 
Collecting dataclasses; python_version < "3.7" (from werkzeug)
  Downloading https://files.pythonhosted.org/packages/fe/ca/75fac5856ab5cfa51bbbcefa250182e50441074fdc3f803f6e76451fab43/dataclasses-0.8-py3-none-any.whl
Collecting WebOb>=1.8 (from pecan)
  Downloading https://files.pythonhosted.org/packages/62/9c/e94a9982e9f31fc35cf46cdc543a6c2c26cb7174635b5fd25b0bbc6a7bc0/WebOb-1.8.7-py2.py3-none-any.whl (114kB)
    100% |████████████████████████████████| 122kB 22kB/s 
Collecting Mako>=0.4.0 (from pecan)
  Downloading https://files.pythonhosted.org/packages/b4/4d/e03d08f16ee10e688bde9016bc80af8b78c7f36a8b37c7194da48f72207e/Mako-1.1.6-py2.py3-none-any.whl (75kB)
    100% |████████████████████████████████| 81kB 58kB/s 
Collecting WebTest>=1.3.1 (from pecan)
  Downloading https://files.pythonhosted.org/packages/41/c7/3897bd62366cb4a50bfb411d37efca9fa33bf07a7c1c22fce8f6ad2664ff/WebTest-3.0.0-py3-none-any.whl
Requirement already satisfied: setuptools in /usr/lib/python3.6/site-packages (from pecan)
Requirement already satisfied: six in /usr/lib/python3.6/site-packages (from pecan)
Collecting logutils>=0.3 (from pecan)
  Downloading https://files.pythonhosted.org/packages/49/b2/b57450889bf73da26027f8b995fd5fbfab258ec24ef967e4c1892f7cb121/logutils-0.3.5.tar.gz
Collecting MarkupSafe>=0.9.2 (from Mako>=0.4.0->pecan)
  Downloading https://files.pythonhosted.org/packages/fc/d6/57f9a97e56447a1e340f8574836d3b636e2c14de304943836bd645fa9c7e/MarkupSafe-2.0.1-cp36-cp36m-manylinux1_x86_64.whl
Collecting beautifulsoup4 (from WebTest>=1.3.1->pecan)
  Downloading https://files.pythonhosted.org/packages/9c/d8/909c4089dbe4ade9f9705f143c9f13f065049a9d5e7d34c828aefdd0a97c/beautifulsoup4-4.11.1-py3-none-any.whl (128kB)
    100% |████████████████████████████████| 133kB 9.7kB/s 
Collecting waitress>=0.8.5 (from WebTest>=1.3.1->pecan)
  Downloading https://files.pythonhosted.org/packages/a8/cf/a9e9590023684dbf4e7861e261b0cfd6498a62396c748e661577ca720a29/waitress-2.0.0-py3-none-any.whl (56kB)
    100% |████████████████████████████████| 61kB 10kB/s 
Collecting soupsieve>1.2 (from beautifulsoup4->WebTest>=1.3.1->pecan)
  Downloading https://files.pythonhosted.org/packages/16/e3/4ad79882b92617e3a4a0df1960d6bce08edfb637737ac5c3f3ba29022e25/soupsieve-2.3.2.post1-py3-none-any.whl
Installing collected packages: dataclasses, werkzeug, WebOb, MarkupSafe, Mako, soupsieve, beautifulsoup4, waitress, WebTest, logutils, pecan
  Running setup.py install for logutils ... done
  Running setup.py install for pecan ... done
Successfully installed Mako-1.1.6 MarkupSafe-2.0.1 WebOb-1.8.7 WebTest-3.0.0 beautifulsoup4-4.11.1 dataclasses-0.8 logutils-0.3.5 pecan-1.4.1 soupsieve-2.3.2.post1 waitress-2.0.0 werkzeug-2.0.3

# 再次验证，集群状态正常
[root@ceph-node-0 k8s-cluster]# ceph -s
  cluster:
    id:     a614e796-6875-4134-a735-2e4db541bba8
    health: HEALTH_OK
 
  services:
    mon: 1 daemons, quorum ceph-node-0 (age 36s)
    mgr: ceph-node-0(active, since 25s)
    osd: 3 osds: 3 up (since 14m), 3 in (since 14m)
 
  data:
    pools:   1 pools, 1 pgs
    objects: 0 objects, 0 B
    usage:   3.0 GiB used, 297 GiB / 300 GiB avail
    pgs:     1 active+clean
```



## 7. 扩展集群

### 7.1. 添加 MONITORS

> **执行任务命令**

```shell
ceph-deploy mon add ceph-node-1
ceph-deploy mon add ceph-node-2
```

> **正确执行输出参考**

```shell
[root@ceph-node-0 k8s-cluster]# ceph-deploy mon add ceph-node-1
[ceph_deploy.conf][DEBUG ] found configuration file at: /root/.cephdeploy.conf
[ceph_deploy.cli][INFO  ] Invoked (2.0.1): /usr/bin/ceph-deploy mon add ceph-node-1
[ceph_deploy.cli][INFO  ] ceph-deploy options:
[ceph_deploy.cli][INFO  ]  username                      : None
[ceph_deploy.cli][INFO  ]  verbose                       : False
[ceph_deploy.cli][INFO  ]  overwrite_conf                : False
[ceph_deploy.cli][INFO  ]  subcommand                    : add
[ceph_deploy.cli][INFO  ]  quiet                         : False
[ceph_deploy.cli][INFO  ]  cd_conf                       : <ceph_deploy.conf.cephdeploy.Conf instance at 0x7f718b7c3e18>
[ceph_deploy.cli][INFO  ]  cluster                       : ceph
[ceph_deploy.cli][INFO  ]  mon                           : ['ceph-node-1']
[ceph_deploy.cli][INFO  ]  func                          : <function mon at 0x7f718b7a8410>
[ceph_deploy.cli][INFO  ]  address                       : None
[ceph_deploy.cli][INFO  ]  ceph_conf                     : None
[ceph_deploy.cli][INFO  ]  default_release               : False
[ceph_deploy.mon][INFO  ] ensuring configuration of new mon host: ceph-node-1
[ceph_deploy.admin][DEBUG ] Pushing admin keys and conf to ceph-node-1
[ceph-node-1][DEBUG ] connected to host: ceph-node-1 
[ceph-node-1][DEBUG ] detect platform information from remote host
[ceph-node-1][DEBUG ] detect machine type
[ceph-node-1][DEBUG ] write cluster configuration to /etc/ceph/{cluster}.conf
[ceph_deploy.mon][DEBUG ] Adding mon to cluster ceph, host ceph-node-1
[ceph_deploy.mon][DEBUG ] using mon address by resolving host: 192.168.9.86
[ceph_deploy.mon][DEBUG ] detecting platform for host ceph-node-1 ...
[ceph-node-1][DEBUG ] connected to host: ceph-node-1 
[ceph-node-1][DEBUG ] detect platform information from remote host
[ceph-node-1][DEBUG ] detect machine type
[ceph-node-1][DEBUG ] find the location of an executable
[ceph_deploy.mon][INFO  ] distro info: CentOS Linux 7.9.2009 Core
[ceph-node-1][DEBUG ] determining if provided host has same hostname in remote
[ceph-node-1][DEBUG ] get remote short hostname
[ceph-node-1][DEBUG ] adding mon to ceph-node-1
[ceph-node-1][DEBUG ] get remote short hostname
[ceph-node-1][DEBUG ] write cluster configuration to /etc/ceph/{cluster}.conf
[ceph-node-1][DEBUG ] create the mon path if it does not exist
[ceph-node-1][DEBUG ] checking for done path: /var/lib/ceph/mon/ceph-ceph-node-1/done
[ceph-node-1][DEBUG ] done path does not exist: /var/lib/ceph/mon/ceph-ceph-node-1/done
[ceph-node-1][INFO  ] creating keyring file: /var/lib/ceph/tmp/ceph-ceph-node-1.mon.keyring
[ceph-node-1][DEBUG ] create the monitor keyring file
[ceph-node-1][INFO  ] Running command: ceph --cluster ceph mon getmap -o /var/lib/ceph/tmp/ceph.ceph-node-1.monmap
[ceph-node-1][WARNIN] got monmap epoch 1
[ceph-node-1][INFO  ] Running command: ceph-mon --cluster ceph --mkfs -i ceph-node-1 --monmap /var/lib/ceph/tmp/ceph.ceph-node-1.monmap --keyring /var/lib/ceph/tmp/ceph-ceph-node-1.mon.keyring --setuser 167 --setgroup 167
[ceph-node-1][INFO  ] unlinking keyring file /var/lib/ceph/tmp/ceph-ceph-node-1.mon.keyring
[ceph-node-1][DEBUG ] create a done file to avoid re-doing the mon deployment
[ceph-node-1][DEBUG ] create the init path if it does not exist
[ceph-node-1][INFO  ] Running command: systemctl enable ceph.target
[ceph-node-1][INFO  ] Running command: systemctl enable ceph-mon@ceph-node-1
[ceph-node-1][WARNIN] Created symlink from /etc/systemd/system/ceph-mon.target.wants/ceph-mon@ceph-node-1.service to /usr/lib/systemd/system/ceph-mon@.service.
[ceph-node-1][INFO  ] Running command: systemctl start ceph-mon@ceph-node-1
[ceph-node-1][INFO  ] Running command: ceph --cluster=ceph --admin-daemon /var/run/ceph/ceph-mon.ceph-node-1.asok mon_status
[ceph-node-1][WARNIN] ceph-node-1 is not defined in `mon initial members`
[ceph-node-1][WARNIN] monitor ceph-node-1 does not exist in monmap
[ceph-node-1][INFO  ] Running command: ceph --cluster=ceph --admin-daemon /var/run/ceph/ceph-mon.ceph-node-1.asok mon_status
[ceph-node-1][DEBUG ] ********************************************************************************
[ceph-node-1][DEBUG ] status for monitor: mon.ceph-node-1
[ceph-node-1][DEBUG ] {
[ceph-node-1][DEBUG ]   "election_epoch": 0, 
[ceph-node-1][DEBUG ]   "extra_probe_peers": [], 
[ceph-node-1][DEBUG ]   "feature_map": {
[ceph-node-1][DEBUG ]     "mon": [
[ceph-node-1][DEBUG ]       {
[ceph-node-1][DEBUG ]         "features": "0x3f01cfb8ffedffff", 
[ceph-node-1][DEBUG ]         "num": 1, 
[ceph-node-1][DEBUG ]         "release": "luminous"
[ceph-node-1][DEBUG ]       }
[ceph-node-1][DEBUG ]     ]
[ceph-node-1][DEBUG ]   }, 
[ceph-node-1][DEBUG ]   "features": {
[ceph-node-1][DEBUG ]     "quorum_con": "0", 
[ceph-node-1][DEBUG ]     "quorum_mon": [], 
[ceph-node-1][DEBUG ]     "required_con": "2449958197560098820", 
[ceph-node-1][DEBUG ]     "required_mon": [
[ceph-node-1][DEBUG ]       "kraken", 
[ceph-node-1][DEBUG ]       "luminous", 
[ceph-node-1][DEBUG ]       "mimic", 
[ceph-node-1][DEBUG ]       "osdmap-prune", 
[ceph-node-1][DEBUG ]       "nautilus", 
[ceph-node-1][DEBUG ]       "octopus"
[ceph-node-1][DEBUG ]     ]
[ceph-node-1][DEBUG ]   }, 
[ceph-node-1][DEBUG ]   "monmap": {
[ceph-node-1][DEBUG ]     "created": "2022-05-24T03:18:55.637040Z", 
[ceph-node-1][DEBUG ]     "epoch": 1, 
[ceph-node-1][DEBUG ]     "features": {
[ceph-node-1][DEBUG ]       "optional": [], 
[ceph-node-1][DEBUG ]       "persistent": [
[ceph-node-1][DEBUG ]         "kraken", 
[ceph-node-1][DEBUG ]         "luminous", 
[ceph-node-1][DEBUG ]         "mimic", 
[ceph-node-1][DEBUG ]         "osdmap-prune", 
[ceph-node-1][DEBUG ]         "nautilus", 
[ceph-node-1][DEBUG ]         "octopus"
[ceph-node-1][DEBUG ]       ]
[ceph-node-1][DEBUG ]     }, 
[ceph-node-1][DEBUG ]     "fsid": "a614e796-6875-4134-a735-2e4db541bba8", 
[ceph-node-1][DEBUG ]     "min_mon_release": 15, 
[ceph-node-1][DEBUG ]     "min_mon_release_name": "octopus", 
[ceph-node-1][DEBUG ]     "modified": "2022-05-24T03:18:55.637040Z", 
[ceph-node-1][DEBUG ]     "mons": [
[ceph-node-1][DEBUG ]       {
[ceph-node-1][DEBUG ]         "addr": "192.168.9.85:6789/0", 
[ceph-node-1][DEBUG ]         "name": "ceph-node-0", 
[ceph-node-1][DEBUG ]         "priority": 0, 
[ceph-node-1][DEBUG ]         "public_addr": "192.168.9.85:6789/0", 
[ceph-node-1][DEBUG ]         "public_addrs": {
[ceph-node-1][DEBUG ]           "addrvec": [
[ceph-node-1][DEBUG ]             {
[ceph-node-1][DEBUG ]               "addr": "192.168.9.85:3300", 
[ceph-node-1][DEBUG ]               "nonce": 0, 
[ceph-node-1][DEBUG ]               "type": "v2"
[ceph-node-1][DEBUG ]             }, 
[ceph-node-1][DEBUG ]             {
[ceph-node-1][DEBUG ]               "addr": "192.168.9.85:6789", 
[ceph-node-1][DEBUG ]               "nonce": 0, 
[ceph-node-1][DEBUG ]               "type": "v1"
[ceph-node-1][DEBUG ]             }
[ceph-node-1][DEBUG ]           ]
[ceph-node-1][DEBUG ]         }, 
[ceph-node-1][DEBUG ]         "rank": 0, 
[ceph-node-1][DEBUG ]         "weight": 0
[ceph-node-1][DEBUG ]       }
[ceph-node-1][DEBUG ]     ]
[ceph-node-1][DEBUG ]   }, 
[ceph-node-1][DEBUG ]   "name": "ceph-node-1", 
[ceph-node-1][DEBUG ]   "outside_quorum": [], 
[ceph-node-1][DEBUG ]   "quorum": [], 
[ceph-node-1][DEBUG ]   "rank": -1, 
[ceph-node-1][DEBUG ]   "state": "probing", 
[ceph-node-1][DEBUG ]   "sync_provider": []
[ceph-node-1][DEBUG ] }
[ceph-node-1][DEBUG ] ********************************************************************************
[ceph-node-1][INFO  ] monitor: mon.ceph-node-1 is currently at the state of probing
```

### 7.2. 添加  MANAGERS

> **执行任务命令**

```shell
ceph-deploy mgr create ceph-node-1
ceph-deploy mgr create ceph-node-2
```

> **正确执行输出参考**

```shell
[root@ceph-node-0 k8s-cluster]# ceph-deploy mgr create ceph-node-1
[ceph_deploy.conf][DEBUG ] found configuration file at: /root/.cephdeploy.conf
[ceph_deploy.cli][INFO  ] Invoked (2.0.1): /usr/bin/ceph-deploy mgr create ceph-node-1
[ceph_deploy.cli][INFO  ] ceph-deploy options:
[ceph_deploy.cli][INFO  ]  username                      : None
[ceph_deploy.cli][INFO  ]  verbose                       : False
[ceph_deploy.cli][INFO  ]  mgr                           : [('ceph-node-1', 'ceph-node-1')]
[ceph_deploy.cli][INFO  ]  overwrite_conf                : False
[ceph_deploy.cli][INFO  ]  subcommand                    : create
[ceph_deploy.cli][INFO  ]  quiet                         : False
[ceph_deploy.cli][INFO  ]  cd_conf                       : <ceph_deploy.conf.cephdeploy.Conf instance at 0x7fe995bef998>
[ceph_deploy.cli][INFO  ]  cluster                       : ceph
[ceph_deploy.cli][INFO  ]  func                          : <function mgr at 0x7fe996461140>
[ceph_deploy.cli][INFO  ]  ceph_conf                     : None
[ceph_deploy.cli][INFO  ]  default_release               : False
[ceph_deploy.mgr][DEBUG ] Deploying mgr, cluster ceph hosts ceph-node-1:ceph-node-1
[ceph-node-1][DEBUG ] connected to host: ceph-node-1 
[ceph-node-1][DEBUG ] detect platform information from remote host
[ceph-node-1][DEBUG ] detect machine type
[ceph_deploy.mgr][INFO  ] Distro info: CentOS Linux 7.9.2009 Core
[ceph_deploy.mgr][DEBUG ] remote host will use systemd
[ceph_deploy.mgr][DEBUG ] deploying mgr bootstrap to ceph-node-1
[ceph-node-1][DEBUG ] write cluster configuration to /etc/ceph/{cluster}.conf
[ceph-node-1][WARNIN] mgr keyring does not exist yet, creating one
[ceph-node-1][DEBUG ] create a keyring file
[ceph-node-1][DEBUG ] create path recursively if it doesn't exist
[ceph-node-1][INFO  ] Running command: ceph --cluster ceph --name client.bootstrap-mgr --keyring /var/lib/ceph/bootstrap-mgr/ceph.keyring auth get-or-create mgr.ceph-node-1 mon allow profile mgr osd allow * mds allow * -o /var/lib/ceph/mgr/ceph-ceph-node-1/keyring
[ceph-node-1][INFO  ] Running command: systemctl enable ceph-mgr@ceph-node-1
[ceph-node-1][WARNIN] Created symlink from /etc/systemd/system/ceph-mgr.target.wants/ceph-mgr@ceph-node-1.service to /usr/lib/systemd/system/ceph-mgr@.service.
[ceph-node-1][INFO  ] Running command: systemctl start ceph-mgr@ceph-node-1
[ceph-node-1][INFO  ] Running command: systemctl enable ceph.target
```

### 7.3. 集群状态查看

> **执行任务命令 (含输出)**

```shell
[root@ceph-node-0 k8s-cluster]# ceph -s
  cluster:
    id:     a614e796-6875-4134-a735-2e4db541bba8
    health: HEALTH_OK
 
  services:
    mon: 3 daemons, quorum ceph-node-0,ceph-node-1,ceph-node-2 (age 3m)
    mgr: ceph-node-0(active, since 23m), standbys: ceph-node-2, ceph-node-1
    osd: 3 osds: 3 up (since 37m), 3 in (since 37m)
 
  task status:
 
  data:
    pools:   1 pools, 1 pgs
    objects: 0 objects, 0 B
    usage:   3.0 GiB used, 297 GiB / 300 GiB avail
    pgs:     1 active+clean
```



## 8. 常见问题

### 8.1. 报错 No module named pkg_resources

> **报错信息**

```shell
[root@ceph-node-0 k8s-cluster]# ceph-deploy new 192.168.9.65
Traceback (most recent call last):
  File "/usr/bin/ceph-deploy", line 18, in <module>
    from ceph_deploy.cli import main
  File "/usr/lib/python2.7/site-packages/ceph_deploy/cli.py", line 1, in <module>
    import pkg_resources
ImportError: No module named pkg_resources
```

> **解决方案**

```shell
yum install python-setuptools
```

### 8.2. 报错 ceph-deploy...must be a hostname not an IP

> **报错信息**

```shell
[root@ceph-node-0 k8s-cluster]# ceph-deploy new 192.168.9.65
usage: ceph-deploy new [-h] [--no-ssh-copykey] [--fsid FSID]
                       [--cluster-network CLUSTER_NETWORK]
                       [--public-network PUBLIC_NETWORK]
                       MON [MON ...]
ceph-deploy new: error: 192.168.9.65 must be a hostname not an IP
```

> **解决方案**

Ceph-deploy 部署时不支持节点使用 IP 的形式，必须使用主机名，因此还要注意 /etc/hosts 配置文件一定要做解析。

---

## 9. 总结

本文简单介绍了 Ceph 分布式存储集群常用的安装部署方式，重点详细讲解了通过 Ceph-deploy 部署 Ceph Octopus 的全部过程。

本文仅适用于学习测试环境，生产环境建议使用 Rook 的方式。

> **参考文档**

- [官网 Ceph 安装过程](https://docs.ceph.com/en/octopus/install/ceph-deploy/quick-ceph-deploy/)

> **Get 文档**

- Github https://github.com/devops/z-notes
- Gitee https://gitee.com/zdevops/z-notes

> **Get 代码**

- Github https://github.com/devops/ansible-zdevops
- Gitee https://gitee.com/zdevops/ansible-zdevops

> **B 站**

- [老 Z 手记](https://space.bilibili.com/1039301316) https://space.bilibili.com/1039301316

> **版权声明** 

- 所有内容均属于原创，整理不易，感谢收藏，转载请标明出处。

> **About Me**

- 昵称：老 Z
- 坐标：山东济南
- 职业：运维架构师 / 高级运维工程师 =**运维**
- 微信：zdevops
- 关注的领域：云计算 / 云原生技术运维，自动化运维
- 技能标签：OpenStack、Ansible、K8S、Python、Go、CNCF
