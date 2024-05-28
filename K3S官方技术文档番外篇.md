#### K3S官方技术文档番外篇

* 缘由
  作为K3S重度使用者，免不了的要经常翻看K3S官方技术文档，官方文档也很给力，可以说是应有尽有，常见问题在文档里都能找到解决办法，但是也有官方文档覆盖不到的，或者说笔者使用过程的一些经验总结，了以分享，以飨同好。

* 官方技术文档之真假美猴王
  官方技术文档一般情况下用搜索引擎搜索k3s的话，来自rancher官方首页的连接比较靠前，其次才是真正的K3S官方首页，如下：

  [K3S文档 | Rancher | Rancher文档](https://docs.rancher.cn/k3s/)
  https://docs.k3s.io/zh/

  其实K3S是rancher旗下的一款开源产品，类似的情况还有Longhorn。那么说回前文的这两个连接究竟有什么异同，笔者肉眼对比，发现大致相同，但是K3S官网(https://docs.k3s.io/zh/)的这个更详尽、更全面、更新的更及时，举例说明：

  * 集群数据存储-备份恢复中，k3s中给了SQLite的备份还原方式，rancher页面里没有

  * 高级选项和配置中，k3s给出了配置 HTTP 代理的方式，rancher页面没有

  * 使用 Docker 作为容器运行时中，k3s给出的docker版本是20，而rancher给出的版本是19
    ```bash
    #K3S
    curl https://releases.rancher.com/install-docker/20.10.sh | sh
    #rancher
    curl https://releases.rancher.com/install-docker/19.03.sh | sh
    ```

  * 参考中，k3s给出了资源给出了资源分析，用于确定 K3s 的最低资源要求，但是rancher没有
    综上，选择技术文档时，优先选择https://docs.k3s.io/zh/

* 国内源
  ```bash
  #官方安装方式
  curl -sfL https://get.k3s.io | sh -
  #官方给出的中国用户安装方式
  curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn sh -
  #阿里源--实际使用中用的最多的方式
  curl -sfL https://rancher-mirror.oss-cn-beijing.aliyuncs.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn sh -
  ```

  官方给定的安装方式，基本无法安装成功，会卡在文件下载页面，推测是因为下载的文件部署在国外导致，要么下载慢，要么下载超时，要么直接无法下载，总之基本没法用；

  官方给定的中国用户安装方式，为中国用户设置了单独的访问页面，经验证也可用，但是在2023年X月的一段时间，该链接一直无法正常打开，导致k3s无法正常安装，所以导致笔者对此时区了信心；

  K3S阿里源安装，实际使用过程中用的最多的方式，成功率也是最高的，虽然成功率高，但也不能保证百分百，某些情况虽然部署主机也能正常访问外网，但是阿里源安装k3s的时候还是不成功，表现为安装很快完成，但是无报错信息，这种情况需要考虑DNS问题，比如将/etc/resolv.conf里的namesever由127.0.0.53改为114.114.114.114，看看能不能管用，当然更复杂的情况需要进一步分析，暂不在这里展开。

* containerd与docker不兼容
  在使用K3S旧版本1.22.x、1.23.x时，彼时用docker作为容器引擎一起使用并未发现很明显的问题，但是到1.26.x、1.27.x时，再用docker作为容器引擎已经出现很多明显的问题，比如kubectl exec -it无法正常进入容器，但是docker exec -it却可以，比如ctr或crictl命令与docker冲突等等，总之不再建议在新的k3s版本中使用docker，而是使用自带的containerd，避免一些额外问题。

* coreDNS纯内网环境启动报错
  典型报错：CoreDNS Error：[FATAL] plugin/loop: Loop

  ```shell
   [FATAL] plugin/loop: Loop (127.0.0.1:49443 -> :53) detected for zone ".", see https://coredns.io/plugins/loop#troubleshooting. Query: "HINFO 5688354173550604804.8931943943623004701."
  ```

  关键字搜索，总能找到答案，问题的核心是DNS解析形成loop环路，也就是自己解析自己导致的服务crash，网络上总结比较好的一遍博文：https://zhuanlan.zhihu.com/p/476611162，怕年老体衰脑子不好使，以为记。

* traefix强制HTTPS转换
  特定情况下需要为K3S里的ingress配置HTTPS访问，这本身没什么问题，但是好多ingress在HTTPS能访问的时候HTTP也能访问，再加上WEB应用的随意跳转，就会出现HTTPS跳转到HTTP的情况，所以需要HTTP强制转为HTTPS，详见：[K3s的traefik配置http强制跳转https - K3s - Rancher 中文论坛](https://forums.rancher.cn/t/k3s-traefik-http-https/450)

* pvc锚定自定义节点
  K3S默认使用local-path的SC，local-path本质上就是使用的节点本地磁盘，虽然该SC基本能完成pvc管理任务，但是有一点，当多节点部署K3S的时候，pvc飘落在哪一个节点完全有SC说了算，这样就无法根据主机特征匹配pvc，这个是local-path的限制。
  但是也有办法可以规避这个策略，那就是在pvc声明的时候，在注释里添加节点选择器，这样就可以根据主机名将pvc锚定在该主机名的节点上，该方法能拯救k3s的一个致命问题，那就是最小单元的pod及pod的持久化卷必须分配在同一节点，不然无法正常拉起pod。此处就显出Longhorn的重要性了。

  ```yaml
  
  ---
  kind: PersistentVolumeClaim
  apiVersion: v1
  metadata:
    name: pgdata2
    namespace: default
    annotations:
      #通过此选项可以调整pvc所在node节点
      #volume.kubernetes.io/selected-node: xingnengceshi
      volume.kubernetes.io/selected-node: VM-3-36-ubuntu
  spec:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 30Gi
    storageClassName: local-path
    volumeMode: Filesystem
  
  ```

  

* 调整节点pod上限110
  当做性能测试的时候，性能节点pod需求量大于110，但是默认情况下K3S节点pod上限就是110，这个需要调整，详见另一篇文档：kubernetes修改集群节点默认Pod数量上限

* k3s节点更换IP
  一些不可抗力，K3S-master单节点更换IP总是避免不了，但是K3S本身更换IP的复杂程度没有想象中那么复杂，如果牵扯到业务IP那么另当别论了，github上有相关说明：https://github.com/k3s-io/k3s/issues/5880，总的说如果是master单节点更换IP，其实也简单，更换后删除旧node然后添加新node，然后重启k3s服务，基本可以解决；如果主机名没有变的话，其实只要重启下K3S服务就能解决