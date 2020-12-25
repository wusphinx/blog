---
title: IaC工具之Ansible
date: 2020-12-25 11:28:08
categories: 
- DevOps
tags:
- DevOps
---
# Ansible是什么？
>Ansible是一种开源软件配置，配置管理和应用程序部署工具，可将基础结构作为代码启用。它可以在许多类Unix系统上运行，并且可以配置类Unix系统和Mi​​crosoft Windows。它包含自己的声明性语言来描述系统配置。Ansible由Michael DeHaan编写，并于2015年被Red Hat收购

如上所述，Ansible是一种开源软件**配置**工具，那它可以用来做什么

# Ansible可以用来做什么？

## 安装Ansible
在Mac下直接安装即可
```
brew install ansible
```
版本
```
ansible --version
ansible 2.10.3
```
## Ansible主机管理
自己手头有两台虚拟机，一台是本地使用[multipass](https://multipass.run/)的虚拟机，一台是自己在腾讯去上买的虚拟机，先配置好ssh名密登录，然后配置好Ansible管理的主机，新建配置文件`/etc/ansible/hosts`，文件配置如下
```
[webservers]
42.192.129.128 ansible_ssh_user=ubuntu
[node]
192.168.64.12 ansible_ssh_user=ubuntu
```
这份配置还是非常清晰易懂的：
- `[webservers]`是可以理解分组，`webservers`就是组名
- `42.192.129.128`自然主机的ip地址了
- `ansible_ssh_user`，故名思意，ssh登录用户名

运行以下命令：
```
➜  ~ ansible all -m ping
192.168.64.12 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
[DEPRECATION WARNING]: Distribution Ubuntu 18.04 on host 42.192.129.128 should use /usr/bin/python3,
 but is using /usr/bin/python for backward compatibility with prior Ansible releases. A future
Ansible release will default to using the discovered platform python for this host. See
https://docs.ansible.com/ansible/2.10/reference_appendices/interpreter_discovery.html for more
information. This feature will be removed in version 2.12. Deprecation warnings can be disabled by
setting deprecation_warnings=False in ansible.cfg.
42.192.129.128 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false,
    "ping": "pong"
}
```
这条`ansible all -m ping`中`-m`的含义是
```
-m MODULE_NAME, --module-name MODULE_NAME
                    module name to execute (default=command)
```
也就是运行ansible的模块`ping`一下所有分组（注意，这里的ping并不是linux命令ping，面是ansible的一个模块实现了与linux下ping类似的功能），也可以指定分组如指定**node**分组
```
➜  ~ ansible node -m ping
192.168.64.12 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
```
通过这几条命令使用下来，可以知道ansible能通过ssh建立连接批量管理远程主机，试想一下，1000台主机的情况下，如果没有ansible，上述ping操作可能要重复敲击1000次，或者写脚本配置完成ansible的类似功能，将操作抽象成配置，减少重复，这或许就是**ansible**这类工具诞生的其中一个动机吧。

## Ansible的Playbook功能
>Ansible脚本的名字叫Playbook，使用的是YAML的格式，文件以yml结尾

实现上例中的相同功能，可编写`demo.yml`如下
```yml
---
- hosts: node
  remote_user: ubuntu
  tasks:
  - name: ping test
    ping: 

```
运行之
```
➜  ~ ansible-playbook demo.yml

PLAY [node] *****************************************************************************************

TASK [Gathering Facts] ******************************************************************************
ok: [192.168.64.12]

TASK [ping test] ************************************************************************************
ok: [192.168.64.12]

PLAY RECAP ******************************************************************************************
192.168.64.12              : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```
很简单的样子，事实上也确实简单，因为yml配置的描述也相当清晰。虽然此处只是一个简的例子，不过也可以看出Playbook其实就是ansible的的一种"脚本"形式，这种声明式配置足够友好（k8s的声明式对象配置倒是与此非常相似，应该是同宗同源）。

## Host Inventory
上术示例主机都是使用默认路径的配置，其实有可以动态自定义主机及分组配置，而且使用yml格式（我真是喜欢yml格式）定义文件`host.yml`如下所示
```yml
--- 
all:
  children:
    webservers:
      hosts:
        42.192.129.128:
          ansible_ssh_user: ubuntu
    node:
      hosts:
        192.168.64.12:
          ansible_ssh_user: ubuntu
```
然后使用命令
```
ansible all -i host.yml -m ping
```
结果与上例无异，个人更喜欢yml这种写法，结构更清晰，当然这是要在熟悉文档及其用法的基础上。

## ansible-console
ansible为用户提供的交互式工具，可以方便地连接远程主机使用shell
```
➜ ansible-console node
Welcome to the ansible console.
Type help or ? to list commands.

whoami@node (1)[f:5]$ lsb_release -a
192.168.64.12 | CHANGED | rc=0 >>
Distributor ID:	Ubuntu
Description:	Ubuntu 18.04.3 LTS
Release:	18.04
Codename:	bionicNo LSB modules are available.
```
发现ansible真挺好用，如果平台用多台虚拟机的话，也可以使用ansible来管理与交互，整体下来学习成本不高，使用却非常方便。

## 条件控制
就像写代码一样，有时候做一些交互操作的时候希望有满足某些条件再执行
```
---
- hosts: node
  remote_user: ubuntu
  tasks:
  - name: ping test
    ping: 
    when: ansible_os_family == "RedHat"
```
用ansible-playbook运行
```
ansible-playbook demo.yml

PLAY [node] ***********************************************************************************************************************************************

TASK [Gathering Facts] ************************************************************************************************************************************
ok: [192.168.64.12]

TASK [ping test] ******************************************************************************************************************************************
skipping: [192.168.64.12]

PLAY RECAP ************************************************************************************************************************************************
192.168.64.12              : ok=1    changed=0    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0
```
因为我的虚拟机是Ubuntu，所以在`ansible_os_family == "RedHat"`为false, 任务就跳过执行了，改为`Debian`就能正确执行了。

## 安装软件
这个功能很有用，特别是需要批量操作
```
---
- hosts: node
  remote_user: ubuntu
  tasks:
  - name: Install the package "nmap"
    become: true
    apt:
      name: nmap
```
注意此处`become: true`是为了权限升级，可理解为`sudo`操作。

---

此篇为开坑之作，还有一些高级用法Roles、`ansible-galaxy`、`ansible-vault`等就不一次展开了，通过简单的使用，ansible解决了批量操作、运维操作重用等功能，描述性配置非常强大，基本等同于编程，这与一些CI/CD平台如gitlab等倒是非常相似的，文档也很丰富且全面。

未完，待续……

参考：
- [Ansible wiki](https://en.wikipedia.org/wiki/Ansible_(software))
- [Ansible中文权威指南](https://ansible-tran.readthedocs.io/en/latest/docs/intro.html)
- [ssh免密登录](https://juejin.cn/post/6844903734233792519)
- [Ansible模块](https://getansible.com/begin/ansiblemo_kuai_module)