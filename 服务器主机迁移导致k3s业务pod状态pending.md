今天碰到一个情况：
表面现象：服务器主机迁移、导致部署在k3s中的业务服务不可用
简单排查：有几个挂载持久化卷的pod全部pending
再进一步describe显示：0/1 nodes are available: 1 node(s) had volume node affinity conflict.
于是进一步排查发现现有pvc中自动生成的annotation里有主机名相关字段
因为云主机迁移导致的主机名变更，而pvc又跟主机名绑定，
所以出现了上述无法调度情况，导致了pod状态pending，导致了业务不可用
解决方法：既然主机名变了 那么再改回来 经验证可行 改完pod自动全拉起
hostnamectl set-hostname old_hostname
service k3s restart
将pvc中绑定主机名的annotation删除也是一个解决思路，但是没有验证可行性
解决项目问题有时候就是这样，那边等着用，这边解决了赶紧反馈，但是丧失了另一种思路验证的机会


