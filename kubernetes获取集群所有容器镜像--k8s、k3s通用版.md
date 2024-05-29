#### kubernetes获取集群所有容器镜像--k8s、k3s通用版

###### 需求--获取集群所有镜像然后导出镜像到文件

利用kubectl的json输出即可获取所有镜像信息，然后再配合其他命令去重和格式整理即可完美输出集群所有镜像

```
root@zhangdeshuai:~# kubectl get pods --all-namespaces -o jsonpath="{..image}" | tr -s '[[:space:]]' '\n' | sort | uniq 
docker.io/rancher/klipper-helm:v0.8.0-build20230510
docker.io/rancher/klipper-lb:v0.4.4
docker.io/rancher/local-path-provisioner:v0.0.24
docker.io/rancher/mirrored-coredns-coredns:1.10.1
docker.io/rancher/mirrored-library-traefik:2.9.10
docker.io/rancher/mirrored-metrics-server:v0.6.3
rancher/klipper-helm:v0.8.0-build20230510
rancher/klipper-lb:v0.4.4
rancher/local-path-provisioner:v0.0.24
rancher/mirrored-coredns-coredns:1.10.1
rancher/mirrored-library-traefik:2.9.10
rancher/mirrored-metrics-server:v0.6.3
```

###### 特殊情况处理--一些重复的镜像--某些dockerhub的镜像可以不写镜像仓库地址docker.io

```
#获取的镜像列表其实有重复 这会导致ctr 导出的时候识别不到下边的镜像
#当然 忽略也没问题，因为是重复的 前边已经导出过了

docker.io/rancher/klipper-helm:v0.8.0-build20230510
docker.io/rancher/klipper-lb:v0.4.4
docker.io/rancher/local-path-provisioner:v0.0.24
docker.io/rancher/mirrored-coredns-coredns:1.10.1
docker.io/rancher/mirrored-library-traefik:2.9.10
docker.io/rancher/mirrored-metrics-server:v0.6.3

rancher/klipper-helm:v0.8.0-build20230510
rancher/klipper-lb:v0.4.4
rancher/local-path-provisioner:v0.0.24
rancher/mirrored-coredns-coredns:1.10.1
rancher/mirrored-library-traefik:2.9.10
rancher/mirrored-metrics-server:v0.6.3
```

###### 借助grep再去一次重复获得最终镜像列表

```
###特殊情况特殊处理 不需要的行是以rancher开头的 那么用grep过滤rancher开头的行 然后grep -v取反
root@zhangdeshuai:~# kubectl get pods --all-namespaces -o jsonpath="{..image}" | tr -s '[[:space:]]' '\n' | sort | uniq | grep ^rancher
rancher/klipper-helm:v0.8.0-build20230510
rancher/klipper-lb:v0.4.4
rancher/local-path-provisioner:v0.0.24
rancher/mirrored-coredns-coredns:1.10.1
rancher/mirrored-library-traefik:2.9.10
rancher/mirrored-metrics-server:v0.6.3
root@zhangdeshuai:~# kubectl get pods --all-namespaces -o jsonpath="{..image}" | tr -s '[[:space:]]' '\n' | sort | uniq | grep -v ^rancher
docker.io/rancher/klipper-helm:v0.8.0-build20230510
docker.io/rancher/klipper-lb:v0.4.4
docker.io/rancher/local-path-provisioner:v0.0.24
docker.io/rancher/mirrored-coredns-coredns:1.10.1
docker.io/rancher/mirrored-library-traefik:2.9.10
docker.io/rancher/mirrored-metrics-server:v0.6.3
```

###### 导出镜像列表为本地文件之处理文件名字

因为文件里一般不会出现斜线和冒号，Linux虽然支持但是可阅读性终究差点，所以要处理导出文件里的文件名，以冒号作为分隔，分别去前后两部分，这样没有了冒号，再将前半部分的斜线换成下划线，相关脚本如下

```
  #docker save $LINE  >${LINE//\//_}.tar
  #这样的文件名称里有冒号 虽然Linux支持 但是担心引起一些别的问题 所以不用这种方式 下面是新的方式
  #举例 harbor.demo.com/library/mario:v3
  #先取tag v3 关键字冒号: 因为一般情况下只会出现一次冒号 所以从左从右匹配都行
  #从左往右匹配会保留匹配后的右侧部分 格式{string#*key} 或{string##*key} 区别在首次匹配和最后匹配后的保留部分--##错误注释
  #担心有多个冒号情况的出现 选择深匹配 也就是最后一次匹配--##错误注释
  #正确注释应该是最短匹配和最长匹配 也就是关键字配合通用匹配符* 完成最短最长匹配 然后把匹配的部分删除
  #*: 就是从左侧开始匹配任何字符串到冒号: 结束
  #双井号##表示最长匹配 
  #harbor.demo.com/library/mario:v3
  #--------------------------:
  #只有一个冒号看不出效果 如果是harbor.demo.com/library/mario:v3:subtag
  #那么*:最长匹配理论结果应该为subtag 最短匹配结果应该为v3:subtag
  #harbor.demo.com/library/mario:v3:subtag
  #--------------------------:  最短匹配
  #--------------------------------------:  最长匹配
  #当然用:* 也能匹配 推测下匹配结果
  #harbor.demo.com/library/mario:v3:subtag
  #                          :------------------
  #                                      :------
  #验证结果显示星号* 只能在关键字左侧 在右侧无效果
  #推测是因为本来就是从左侧匹配 如果把通配符放在右边 然后本身就有冲突
  #https://tldp.org/LDP/abs/html/string-manipulation.html
  #根据上文教程 推测g*p
  #经验证 从右侧匹配的时候 通配符型号*需要放在右边（左侧匹配时 型号放在左边）否则也会失去效果
  #所以左侧匹配 右侧匹配实际上应该是有方向的 关键字冒号:
  #从左侧匹配时*:  从右侧匹配时:*
  #harbor.demo.com/library/mario:v3:subtag
  #------------------------->:
  #------------------------------------->:
  #                                      :----->
  #                          :----------------->
  #root@zhangdeshuai:~# echo ${LINE#*:}
  #v3:subtag
  #root@zhangdeshuai:~# echo ${LINE##*:}
  #subtag
  #root@zhangdeshuai:~# echo ${LINE%:*}
  #harbor.demo.com/library/mario:v3
  #root@zhangdeshuai:~# echo ${LINE%%:*}
  #harbor.demo.com/library/mario
  #总结 虽然匹配方向不同 但是匹配的关键字序列都是从左往右 不会出现从右往左的匹配关键字
  TAG=${LINE##*:}
  #然后去镜像的名字 harbor.demo.com/library/mario 关键字还是冒号 但是这次选择从右往左匹配
  #从右往左匹配会保留匹配后的左侧部分 格式{string%key*} 或{string%%key*} 两者区别也是首次匹配和最后匹配后的保留部分
  #从右往左匹配和从左往右匹配语法格式的不同是 关键字key放在中间 而不是最后边
  #取镜像名称URL 从右往左匹配 关键字冒号: 保留首次匹配成功左侧部分 harbor.demo.com/library/mario
  URL=${LINE%:*}
  #URL里有斜线/ 无法用作文件名 将斜线转换为下划线_ harbor.demo.com/library/mario
  #大括号扩展 替换字符串格式 ${var/old_char/new_char} 或 ${var//old_char/new_char}
  #两者区别是仅替换第一个和替换所有
  #我们选择替换所有 并且斜线/ 需要转义为\/
  IMAGE=${URL//\//_}
  docker save $LINE > ${IMAGE}_${TAG}.tar &
  #不放后台运行时间558秒 放后台为并发执行时间326秒
  #如果没有docker 可以选择用ctr导出镜像
  #ctr image export ${IMAGE}_${TAG}.tar $LINE &
  #注意 ctr处理docker hub镜像的时候 需要加前缀docker.io/  
  #原样镜像 docker.io/rancher/local-path-provisioner:v0.0.24
  #kubectl 和 docker 会识别成 rancher/local-path-provisioner:v0.0.24
  #但是ctr 会识别成 docker.io/rancher/local-path-provisioner:v0.0.24
```

###### 完成获取镜像列表且导出镜像列表为本地文件脚本如下

```
#!/bin/bash
set -ex

###获取k8s集群所有镜像


#定义脚本运行的开始时间
start_time=`date +%s`


#获取k8s集群所有镜像并保存到文件images.txt
#uniq 不加c 不统计重复行数
kubectl get pods --all-namespaces -o jsonpath="{..image}" | tr -s '[[:space:]]' '\n' | sort | uniq > images.txt
#遍历镜像文件列表 并将列表里的镜像保存成本地文件
while read LINE
do
  #docker save $LINE  >${LINE//\//_}.tar
  #这样的文件名称里有冒号 虽然Linux支持 但是担心引起一些别的问题 所以不用这种方式 下面是新的方式

  #举例 harbor.demo.com/library/mario:v3
  #先取tag v3 关键字冒号: 因为一般情况下只会出现一次冒号 所以从左从右匹配都行
  #从左往右匹配会保留匹配后的右侧部分 格式{string#*key} 或{string##*key} 区别在首次匹配和最后匹配后的保留部分--##错误注释
  #担心有多个冒号情况的出现 选择深匹配 也就是最后一次匹配--##错误注释
  #正确注释应该是最短匹配和最长匹配 也就是关键字配合通用匹配符* 完成最短最长匹配 然后把匹配的部分删除
  #*: 就是从左侧开始匹配任何字符串到冒号: 结束
  #双井号##表示最长匹配 
  #harbor.demo.com/library/mario:v3
  #--------------------------:
  #只有一个冒号看不出效果 如果是harbor.demo.com/library/mario:v3:subtag
  #那么*:最长匹配理论结果应该为subtag 最短匹配结果应该为v3:subtag
  #harbor.demo.com/library/mario:v3:subtag
  #--------------------------:  最短匹配
  #--------------------------------------:  最长匹配
  #当然用:* 也能匹配 推测下匹配结果
  #harbor.demo.com/library/mario:v3:subtag
  #                          :------------------
  #                                      :------
  #验证结果显示星号* 只能在关键字左侧 在右侧无效果
  #推测是因为本来就是从左侧匹配 如果把通配符放在右边 然后本身就有冲突
  #https://tldp.org/LDP/abs/html/string-manipulation.html
  #根据上文教程 推测g*p
  #经验证 从右侧匹配的时候 通配符型号*需要放在右边（左侧匹配时 型号放在左边）否则也会失去效果
  #所以左侧匹配 右侧匹配实际上应该是有方向的 关键字冒号:
  #从左侧匹配时*:  从右侧匹配时:*
  #harbor.demo.com/library/mario:v3:subtag
  #------------------------->:
  #------------------------------------->:
  #                                      :----->
  #                          :----------------->
  #root@zhangdeshuai:~# echo ${LINE#*:}
  #v3:subtag
  #root@zhangdeshuai:~# echo ${LINE##*:}
  #subtag
  #root@zhangdeshuai:~# echo ${LINE%:*}
  #harbor.demo.com/library/mario:v3
  #root@zhangdeshuai:~# echo ${LINE%%:*}
  #harbor.demo.com/library/mario
  #总结 虽然匹配方向不同 但是匹配的关键字序列都是从左往右 不会出现从右往左的匹配关键字

  TAG=${LINE##*:}
  #然后去镜像的名字 harbor.demo.com/library/mario 关键字还是冒号 但是这次选择从右往左匹配
  #从右往左匹配会保留匹配后的左侧部分 格式{string%key*} 或{string%%key*} 两者区别也是首次匹配和最后匹配后的保留部分
  #从右往左匹配和从左往右匹配语法格式的不同是 关键字key放在中间 而不是最后边
  #取镜像名称URL 从右往左匹配 关键字冒号: 保留首次匹配成功左侧部分 harbor.demo.com/library/mario
  URL=${LINE%:*}
  #URL里有斜线/ 无法用作文件名 将斜线转换为下划线_ harbor.demo.com/library/mario
  #大括号扩展 替换字符串格式 ${var/old_char/new_char} 或 ${var//old_char/new_char}
  #两者区别是仅替换第一个和替换所有
  #我们选择替换所有 并且斜线/ 需要转义为\/
  IMAGE=${URL//\//_}
  docker save $LINE > ${IMAGE}_${TAG}.tar &
  #不放后台运行时间558秒 放后台为并发执行时间326秒
  #如果没有docker 可以选择用ctr导出镜像
  #ctr image export ${IMAGE}_${TAG}.tar $LINE &
  #注意 ctr处理docker hub镜像的时候 需要加前缀docker.io/  
  #原样镜像 docker.io/rancher/local-path-provisioner:v0.0.24
  #kubectl 和 docker 会识别成 rancher/local-path-provisioner:v0.0.24
  #但是ctr 会识别成 docker.io/rancher/local-path-provisioner:v0.0.24
done < images.txt

#等待save的后台任务全部结束
wait

#定义脚本运行的结束时间
stop_time=`date +%s`


#输出脚本运行时间
echo "脚本运行时间：`expr ${stop_time} - ${start_time}`"
```

以上脚本导出时依赖docker,但是当前k8s或k3s集群往往不在用docker作为容器引擎而是直接用containerd,所以改为ctr作为导出命令--但是ctr无法处理docker.io/rancher/local-path-provisio这样的镜像，所以需要在列表里单独处理，因为是重复的所以用grep过滤掉，完成脚本如下

```
#!/bin/bash
set -ex

###获取k8s集群所有镜像


#v2打算放弃使用外部文件 而是使用数组


#定义脚本运行的开始时间
start_time=`date +%s`

IMAGES=($(kubectl get pods --all-namespaces -o jsonpath="{..image}" | tr -s '[[:space:]]' '\n' | sort | uniq | grep -v ^rancher))
#如果为了验证脚本需要把命令写在一行 注意循环体do的内容用大括号抱起来 不然后台进程& 会被识别报错
#for I in ${IMAGES[@]}; do { TAG=${I##*:};  URL=${I%:*};  IMAGE=${URL//\//_};  docker save $I > ${IMAGE}_${TAG}.tar &}; done
for I in ${IMAGES[@]}; do { TAG=${I##*:};  URL=${I%:*};  IMAGE=${URL//\//_};  ctr i export ${IMAGE}_${TAG}.tar ${I} &}; done

#for I in ${IMAGES[@]}; do 
#  echo "current image $I"
#  TAG=${I##*:}
#  URL=${I%:*}
#  IMAGE=${URL//\//_}
#  docker save $I > ${IMAGE}_${TAG}.tar &
#  ctr i export ${IMAGE}_${TAG}.tar ${I} &
#done


#等待save的后台任务全部结束
wait

#定义脚本运行的结束时间
stop_time=`date +%s`


#输出脚本运行时间
echo "脚本运行时间：`expr ${stop_time} - ${start_time}`"
```

部分数据结果如下

```
root@zhangdeshuai:/opt/mario #定义脚本运行的开始时间
root@zhangdeshuai:/opt/mario start_time=`date +%s`
root@zhangdeshuai:/opt/mario 
root@zhangdeshuai:/opt/mario IMAGES=($(kubectl get pods --all-namespaces -o jsonpath="{..image}" | tr -s '[[:space:]]' '\n' | sort | uniq))
root@zhangdeshuai:/opt/mario for I in ${IMAGES[@]}; do { TAG=${I##*:};  URL=${I%:*};  IMAGE=${URL//\//_};  docker save $I > ${IMAGE}_${TAG}.tar &}; done
[1] 6651
[2] 6652
[3] 6653
[4] 6654
[5] 6655
[6] 6656
[7] 6657
[8] 6658
[9] 6659
[10] 6660
[11] 6661
[12] 6662
[13] 6663
[14] 6664
[15] 6665
[16] 6666
[17] 6668
[18] 6669
[19] 6670
[20] 6671
[21] 6672
[22] 6673
[23] 6674
[24] 6675
[25] 6676
root@zhangdeshuai:/opt/mario 
root@zhangdeshuai:/opt/mario #for I in ${IMAGES[@]}; do 
root@zhangdeshuai:/opt/mario #  echo "current image $I"
root@zhangdeshuai:/opt/mario #  TAG=${I##*:}
root@zhangdeshuai:/opt/mario #  URL=${I%:*}
root@zhangdeshuai:/opt/mario #  IMAGE=${URL//\//_}
root@zhangdeshuai:/opt/mario #  docker save $I > ${IMAGE}_${TAG}.tar &
root@zhangdeshuai:/opt/mario #done
root@zhangdeshuai:/opt/mario 
root@zhangdeshuai:/opt/mario 
root@zhangdeshuai:/opt/mario #等待save的后台任务全部结束
root@zhangdeshuai:/opt/mario wait
[1]   Done                    docker save $I > ${IMAGE}_${TAG}.tar
[4]   Done                    docker save $I > ${IMAGE}_${TAG}.tar
[5]   Done                    docker save $I > ${IMAGE}_${TAG}.tar
[6]   Done                    docker save $I > ${IMAGE}_${TAG}.tar
[7]   Done                    docker save $I > ${IMAGE}_${TAG}.tar
[8]   Done                    docker save $I > ${IMAGE}_${TAG}.tar
[9]   Done                    docker save $I > ${IMAGE}_${TAG}.tar
[10]   Done                    docker save $I > ${IMAGE}_${TAG}.tar
[11]   Done                    docker save $I > ${IMAGE}_${TAG}.tar
[12]   Done                    docker save $I > ${IMAGE}_${TAG}.tar
[14]   Done                    docker save $I > ${IMAGE}_${TAG}.tar
[15]   Done                    docker save $I > ${IMAGE}_${TAG}.tar
[16]   Done                    docker save $I > ${IMAGE}_${TAG}.tar
[17]   Done                    docker save $I > ${IMAGE}_${TAG}.tar
[18]   Done                    docker save $I > ${IMAGE}_${TAG}.tar
[19]   Done                    docker save $I > ${IMAGE}_${TAG}.tar
[20]   Done                    docker save $I > ${IMAGE}_${TAG}.tar
[21]   Done                    docker save $I > ${IMAGE}_${TAG}.tar
[22]   Done                    docker save $I > ${IMAGE}_${TAG}.tar
[23]   Done                    docker save $I > ${IMAGE}_${TAG}.tar
[24]-  Done                    docker save $I > ${IMAGE}_${TAG}.tar
[25]+  Done                    docker save $I > ${IMAGE}_${TAG}.tar
[2]   Done                    docker save $I > ${IMAGE}_${TAG}.tar
[3]-  Done                    docker save $I > ${IMAGE}_${TAG}.tar
[13]+  Done                    docker save $I > ${IMAGE}_${TAG}.tar
root@zhangdeshuai:/opt/mario 
root@zhangdeshuai:/opt/mario #定义脚本运行的结束时间
root@zhangdeshuai:/opt/mario stop_time=`date +%s`
root@zhangdeshuai:/opt/mario 
root@zhangdeshuai:/opt/mario #输出脚本运行时间
root@zhangdeshuai:/opt/mario echo "脚本运行时间：`expr ${stop_time} - ${start_time}`"
脚本运行时间：327
root@zhangdeshuai:/opt/mario 



root@zhangdeshuai:/opt/mario ###获取k8s集群所有镜像
root@zhangdeshuai:/opt/mario 
root@zhangdeshuai:/opt/mario 
root@zhangdeshuai:/opt/mario #v2打算放弃使用外部文件 而是使用数组
root@zhangdeshuai:/opt/mario 
root@zhangdeshuai:/opt/mario 
root@zhangdeshuai:/opt/mario #定义脚本运行的开始时间
root@zhangdeshuai:/opt/mario start_time=`date +%s`
root@zhangdeshuai:/opt/mario 
root@zhangdeshuai:/opt/mario IMAGES=($(kubectl get pods --all-namespaces -o jsonpath="{..image}" | tr -s '[[:space:]]' '\n' | sort | uniq | grep -v ^rancher))
root@zhangdeshuai:/opt/mario #如果为了验证脚本需要把命令写在一行 注意循环体do的内容用大括号抱起来 不然后台进程& 会被识别报错
root@zhangdeshuai:/opt/mario #for I in ${IMAGES[@]}; do { TAG=${I##*:};  URL=${I%:*};  IMAGE=${URL//\//_};  docker save $I > ${IMAGE}_${TAG}.tar &}; done
root@zhangdeshuai:/opt/mario for I in ${IMAGES[@]}; do { TAG=${I##*:};  URL=${I%:*};  IMAGE=${URL//\//_};  ctr i export ${IMAGE}_${TAG}.tar ${I} &}; done
[1] 23714
[2] 23715
[3] 23716
[4] 23717
[5] 23723
[6] 23728
[7] 23730
[8] 23733
[9] 23741
[10] 23748
[11] 23752
[12] 23758
[13] 23763
[14] 23768
[15] 23771
[16] 23773
[17] 23774
[18] 23775
[19] 23776
[20] 23777
[21] 23779
[22] 23785
[23] 23786
[24] 23790
[25] 23791
[26] 23794
root@zhangdeshuai:/opt/mario 
root@zhangdeshuai:/opt/mario #for I in ${IMAGES[@]}; do 
root@zhangdeshuai:/opt/mario #  echo "current image $I"
root@zhangdeshuai:/opt/mario #  TAG=${I##*:}
root@zhangdeshuai:/opt/mario #  URL=${I%:*}
root@zhangdeshuai:/opt/mario #  IMAGE=${URL//\//_}
root@zhangdeshuai:/opt/mario #  docker save $I > ${IMAGE}_${TAG}.tar &
root@zhangdeshuai:/opt/mario #  ctr i export ${IMAGE}_${TAG}.tar ${I} &
root@zhangdeshuai:/opt/mario #done
root@zhangdeshuai:/opt/mario 
root@zhangdeshuai:/opt/mario 
root@zhangdeshuai:/opt/mario #等待save的后台任务全部结束
root@zhangdeshuai:/opt/mario wait
[1]   Done                    ctr i export ${IMAGE}_${TAG}.tar ${I}
[2]   Done                    ctr i export ${IMAGE}_${TAG}.tar ${I}
[3]   Done                    ctr i export ${IMAGE}_${TAG}.tar ${I}
[4]   Done                    ctr i export ${IMAGE}_${TAG}.tar ${I}
[5]   Done                    ctr i export ${IMAGE}_${TAG}.tar ${I}
[6]   Done                    ctr i export ${IMAGE}_${TAG}.tar ${I}
[9]   Done                    ctr i export ${IMAGE}_${TAG}.tar ${I}
[10]   Done                    ctr i export ${IMAGE}_${TAG}.tar ${I}
[13]   Done                    ctr i export ${IMAGE}_${TAG}.tar ${I}
[16]   Done                    ctr i export ${IMAGE}_${TAG}.tar ${I}
[17]   Done                    ctr i export ${IMAGE}_${TAG}.tar ${I}
[20]   Done                    ctr i export ${IMAGE}_${TAG}.tar ${I}
[21]   Done                    ctr i export ${IMAGE}_${TAG}.tar ${I}
[23]   Done                    ctr i export ${IMAGE}_${TAG}.tar ${I}
[24]   Done                    ctr i export ${IMAGE}_${TAG}.tar ${I}
[26]+  Done                    ctr i export ${IMAGE}_${TAG}.tar ${I}
[7]   Done                    ctr i export ${IMAGE}_${TAG}.tar ${I}
[8]   Done                    ctr i export ${IMAGE}_${TAG}.tar ${I}
[11]   Done                    ctr i export ${IMAGE}_${TAG}.tar ${I}
[12]   Done                    ctr i export ${IMAGE}_${TAG}.tar ${I}
[14]   Done                    ctr i export ${IMAGE}_${TAG}.tar ${I}
[15]   Done                    ctr i export ${IMAGE}_${TAG}.tar ${I}
[18]   Done                    ctr i export ${IMAGE}_${TAG}.tar ${I}
[19]   Done                    ctr i export ${IMAGE}_${TAG}.tar ${I}
[22]-  Done                    ctr i export ${IMAGE}_${TAG}.tar ${I}
[25]+  Done                    ctr i export ${IMAGE}_${TAG}.tar ${I}
root@zhangdeshuai:/opt/mario 
root@zhangdeshuai:/opt/mario #定义脚本运行的结束时间
root@zhangdeshuai:/opt/mario stop_time=`date +%s`
root@zhangdeshuai:/opt/mario 
root@zhangdeshuai:/opt/mario 
root@zhangdeshuai:/opt/mario #输出脚本运行时间
root@zhangdeshuai:/opt/mario echo "脚本运行时间：`expr ${stop_time} - ${start_time}`"
脚本运行时间：78
root@zhangdeshuai:/opt/mario 
```

###### 补充

镜像个数少的情况下可以用docker将多个镜像导出到一个本地文件

ctr也能将多个镜像导出到一个本地文件

如果选择导出一个文件，少数镜像的话可行，镜像数量多的话建议分开，一来镜像明确，二来可以用并发提高导出效率

```
root@zhangdeshuai:~# docker save -o ingress-nginx.tar registry.k8s.io/ingress-nginx/controller:v1.8.1 registry.k8s.io/ingress-nginx/kube-webhook-certgen:v20230407 
root@zhangdeshuai:~# ls 
acme-certs  ingress-nginx.tar  nginx-ingress-images-1.8.1.tar  snap
root@zhangdeshuai:/home/heimdall/app/inSuite/images-bak# IMAGES=($(kubectl get pods --all-namespaces -o jsonpath="{..image}" | tr -s '[[:space:]]' '\n' | sort | uniq | grep -v ^rancher))
root@zhangdeshuai:~#/home/heimdall/app/inSuite/images-bak# echo ${IMAGES[@]}
docker.io/rancher/klipper-helm:v0.8.0-build20230510 docker.io/rancher/klipper-lb:v0.4.4 docker.io/rancher/local-path-provisioner:v0.0.24 docker.io/rancher/mirrored-coredns-coredns:1.10.1 docker.io/rancher/mirrored-library-traefik:2.9.10 docker.io/rancher/mirrored-metrics-server:v0.6.3 harbor.demo.com/library/mario:v3 
root@zhangdeshuai:/home/heimdall/app/inSuite/images-bak# ctr i export images_all.tag ${IMAGES[@]}###### 再补充--docker导出镜像tag丢失问题
```

如果导出的时候，也就是docker save的时候用的镜像名+镜像tag，那么docker load的时候一般不会丢失，如果save的时候用的镜像完整id，那么还原出来的镜像没有tag

其实docker pull的时候如果加了@那么镜像已经没有tag了

```
root@zhangdeshuai:~# docker pull registry.k8s.io/ingress-nginx/kube-webhook-certgen:v20230407@sha256:543c40fd093964bc9ab509d3e791f9989963021f1e9e4c9c7b6700b02bfb227b && docker pull registry.k8s.io/ingress-nginx/controller:v1.8.1@sha256:e5c4824e7375fcf2a393e1c03c293b69759af37a9ca6abdb91b13d78a93da8bd
registry.k8s.io/ingress-nginx/kube-webhook-certgen@sha256:543c40fd093964bc9ab509d3e791f9989963021f1e9e4c9c7b6700b02bfb227b: Pulling from ingress-nginx/kube-webhook-certgen
Digest: sha256:543c40fd093964bc9ab509d3e791f9989963021f1e9e4c9c7b6700b02bfb227b
Status: Image is up to date for registry.k8s.io/ingress-nginx/kube-webhook-certgen@sha256:543c40fd093964bc9ab509d3e791f9989963021f1e9e4c9c7b6700b02bfb227b
registry.k8s.io/ingress-nginx/kube-webhook-certgen:v20230407@sha256:543c40fd093964bc9ab509d3e791f9989963021f1e9e4c9c7b6700b02bfb227b
registry.k8s.io/ingress-nginx/controller@sha256:e5c4824e7375fcf2a393e1c03c293b69759af37a9ca6abdb91b13d78a93da8bd: Pulling from ingress-nginx/controller
Digest: sha256:e5c4824e7375fcf2a393e1c03c293b69759af37a9ca6abdb91b13d78a93da8bd
Status: Image is up to date for registry.k8s.io/ingress-nginx/controller@sha256:e5c4824e7375fcf2a393e1c03c293b69759af37a9ca6abdb91b13d78a93da8bd
registry.k8s.io/ingress-nginx/controller:v1.8.1@sha256:e5c4824e7375fcf2a393e1c03c293b69759af37a9ca6abdb91b13d78a93da8bd
root@zhangdeshuai:~# docker images 
REPOSITORY                                           TAG            IMAGE ID       CREATED        SIZE
registry.k8s.io/ingress-nginx/controller             <none>         825aff16c20c   6 weeks ago    284MB
registry.k8s.io/ingress-nginx/kube-webhook-certgen   <none>         7e7451bb7042   4 months ago   47.2MB
openjdk                                              8-jre-alpine   f7a292bbb70c   4 years ago    84.9MB
```
