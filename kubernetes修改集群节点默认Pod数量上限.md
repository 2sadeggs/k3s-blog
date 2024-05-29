### kubernetes修改集群节点默认Pod数量上限

###### 背景

默认k8s集群节点pod数量上限为110，某些特殊情况下单节点所需的pod数量会超过110，这个时候需要修改这个上限

###### 核心参数

修改节点pod数量上限，核心思路是给kubelet传入max-pods参数，实例如下：

--kubelet-arg max-pods=300

###### k3s中修改节点pod数量上限

k3s中的kubelet与k8s中稍有不同，k3s中的kubelet集成到了k3s可执行文件中，而k3s可执行二进制文件被托管成了系统后台service

查看K3S后台服务找到服务配置文件

```
#查看K3S后台服务找到服务配置文件
root@xingnengceshi:~# systemctl status k3s 
● k3s.service - Lightweight Kubernetes
   Loaded: loaded (/etc/systemd/system/k3s.service; enabled; vendor preset: enabled)
   Active: active (running) since Tue 2023-07-25 11:42:39 CST; 1 day 2h ago
     Docs: https://k3s.io
  Process: 1056 ExecStartPre=/sbin/modprobe overlay (code=exited, status=0/SUCCESS)
  Process: 1055 ExecStartPre=/sbin/modprobe br_netfilter (code=exited, status=0/SUCCESS)
  Process: 1053 ExecStartPre=/bin/sh -xc ! /usr/bin/systemctl is-enabled --quiet nm-cloud-setup.service (code=exited, status=0/SUCCESS)
 Main PID: 1057 (k3s-server)
```

#修改K3S服务配置文件添加max-pods参数

```
root@xingnengceshi:~# cat /etc/systemd/system/k3s.service
[Unit]
Description=Lightweight Kubernetes
Documentation=https://k3s.io
Wants=network-online.target
After=network-online.target

[Install]
WantedBy=multi-user.target

[Service]
Type=notify
EnvironmentFile=-/etc/default/%N
EnvironmentFile=-/etc/sysconfig/%N
EnvironmentFile=-/etc/systemd/system/k3s.service.env
KillMode=process
Delegate=yes
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNOFILE=1048576
LimitNPROC=infinity
LimitCORE=infinity
TasksMax=infinity
TimeoutStartSec=0
Restart=always
RestartSec=5s
ExecStartPre=/bin/sh -xc '! /usr/bin/systemctl is-enabled --quiet nm-cloud-setup.service'
ExecStartPre=-/sbin/modprobe br_netfilter
ExecStartPre=-/sbin/modprobe overlay
ExecStart=/usr/local/bin/k3s \
    server \
--kubelet-arg max-pods=300
```

<mark>注意不要把--kubelet-arg max-pods=300和server写在同一行，容易出错</mark>

最后重新加载服务和重启K3S服务

systemctl daemon-reload && systemctl restart k3s

下面的方式写在同一行不会报错，<mark>如果同一行内出现两个空格就会报错</mark>

```
ExecStart=/usr/local/bin/k3s \
    server --kubelet-arg \
    max-pods=300
```

下面的写法就会报错，并且最后一行字体会全部变成绿色

```
ExecStart=/usr/local/bin/k3s \
    server --kubelet-arg max-pods=300 
```

<mark>还有，如果是K3S节点时worker节点，那么服务名称为k3s-agent</mark>
