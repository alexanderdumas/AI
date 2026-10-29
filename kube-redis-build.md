# 基于KubeSphere 部署Redis集群
**概述:** Redis集群的部署是现代系统的必备组件，只有很传统的公司还依然使用的是单点Redis。经过发现，网上大部分都不太准确，很多都是有问题。基本操作按照这个地址：https://kubesphere.io/zh/blogs/kubesphere-redis/
这里面有两个问题：
1）容器和宿主机要配置的时候，不要按照上面的模板进行。应该配置HostPath。否则一旦配置了模板，直接该yaml文件重启不会改变映射关系
2）上面说到的是关于不用IP绑定，而是用域名。我下载过那个工具包，看了源码。本质还是IP绑定。使用的是dig命令解释出来IP，一旦POD重启集群就失效了。应该用这个参数：--cluster-announce-ip（使用这个参数，尽量用6以上的版本。笔者使用的是6.2.3）
3）redis持久化的问题。就是要配置容器和宿主机之间的目录映射关系。保证rdb数据不丢失。笔者是放在一个共享磁盘上。对每个pod建立一个目录。然后在redis启动添加指定dir，在指定dir的时候通过获取POD的name达到目的，以下是核心部分：

## 容器配置
```yaml
containers:
        - name: container-pfpgvb
          image: 'redis:6.2.3'
          command:
            - redis-server
          args:
            - /etc/redis/redis.conf
            - '--protected-mode'
            - 'no'
            - '--cluster-announce-ip'
            - $(POD_IP)
            - '--dir'
            - /data/$(POD_NAME)
          ports:
            - name: tcp-6379
              containerPort: 6379
              protocol: TCP
          env:
            - name: POD_IP
              valueFrom:
                fieldRef:
                  apiVersion: v1
                  fieldPath: status.podIP
            - name: POD_NAME
```

## 卷的配置
```yaml
volumes:
        - name: redis-pvc
          hostPath:
            path: /home/data/data_ai_rediscluster
            type: ''
        - name: volume-ngntn5
          configMap:
            name: redis-conf
            defaultMode: 420
volumeMounts:
            - name: redis-pvc
              mountPath: /data
            - name: volume-ngntn5
              readOnly: true
              mountPath: /etc/redis
```
