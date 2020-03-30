---
title: 记一次ETCD灾难恢复
date: 2020-03-30 15:33:46
tags:
- Kubernetes
- ETCD
---
# 记一次ETCD的故障恢复

事情的起因是，前两天给客户新搭建了一个K8S的集群，今天客户问我说exec有点问题让我给看一下。

那就看看呗。

上去看看，的确不能exec，报错：

    Error from server: error dialing backend: dial tcp 10.17.77.29:10250: i/o timeout

kubectl get svc一下吧，发现cluster ip 是 10.16段的，这不对啊，我一看配置文件是直接复制的dev集群的，没改，得，重来吧，删了集群重拉！

然后就在本地更新配置，清空S3，重新上传，删除主机，等待重拉。

这时候还出了一个小插曲，本来要删除三个Master主机，结果手滑吧跳板机也删了，还好这个跳板机上没啥东西，赶紧恢复吧，把之前的工作量有做了一遍。

折腾了半天，美滋滋的拉起跳板机，重新get svc，不对啊，怎么还是16，下意识看了一下S3，发现那个S3还是空的，不对我上传了啊，我扫了一眼桶名，是dev环境的桶，我心一凉，赶紧上传dev桶里的数据。

我登录dev跳板机一看，k8s里啥也没了，只剩几滴尿在我的裤子上。

告诉自己别慌，冷静

先看看能不能恢复，不行就重建吧，毕竟还没投入生产，没啥数据是万幸。

这里吐槽一下，为啥我把S3上的etcd backups删了，主库会空？？？？

先看看本机ETCD能否回滚吧，看了一下本机没有备份。

那咋办呢，心里更慌了，研究一下S3有没有办法恢复，发现S3的bucket开了版本控制，不幸中的万幸，AWS牛逼~

恢复最后一个S3的备份，然后看看怎么恢复ETCD

因为集群是用Kops为核心，自己魔改的，ETCD用的还是kops的

从备份里恢复吧

    ./etcd-manager-ctl --backup-store=s3://cf-uws/cluster.luqel-water.k8s.local/backups/etcd/main list-backups
    ./etcd-manager-ctl --backup-store=s3://cf-uws/cluster.luqel-water.k8s.local/backups/etcd/main restore-backup 2020-03-23T06:49:19Z-007671

恢复完，需要对每个etcd容器重启：

    docker restart xxxxxx

解决了~AWS 牛逼!

恢复了之后就去修复CIDR重复的问题，我说ETCD怎么总和之前Dev那套环境的ETCD组成集群呢。

修改了cluster.spec，等AWS Autoscaling group拉起EC2即可。

集群配置更新完之后，发现之前的问题仍存在。

对比发现Master的安全组没有添加Node的access，加上就解决了