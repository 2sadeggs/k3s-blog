### 关于ctr加载镜像crictl查询不到

* 关联问题：ctr导入了镜像 (ctr image import demo-image.tar )，但是crictl images查询不到，且在k3s里也像也加载不到该镜像

* 先说结论：docker 和 k3s 混用，docker 自带了ctr命令，且k3s本身也自带的ctr命令（其实是k3s本身，后有说明）

* 因为默认自带了docker，后来又安装了k3s，这种情况直接使用ctr命令实际上是用的docker自带的ctr，这种情况k3s的命名空间其实识别不到加载的镜像，解决方式就是使用k3s ctr代替ctr导入镜像文件

* ```bash
  #docker自带的ctr
  [root@insuitedzda ~]# which ctr
  /usr/bin/ctr
  [root@insuitedzda ~]# ls -la /usr/bin/ctr 
  -rwxr-xr-x 1 root root 27099648 Feb 17  2023 /usr/bin/ctr
  [root@insuitedzda ~]# 
  #k3s自带的ctr 本身就是k3s命令的软连接
  root@nicaicai:~# which ctr
  /usr/local/bin/ctr
  root@nicaicai:~# ls -la /usr/local/bin/ctr 
  lrwxrwxrwx 1 root root 3 Dec 19 09:58 /usr/local/bin/ctr -> k3s
  root@nicaicai:~# 
  
  ```

* 小结：尽量不要docker和k3s混用，虽然k3s也支持docker容器运行时，但是当前情况下因为混用引起的奇葩问题实在太多了