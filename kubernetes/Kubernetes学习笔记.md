## Kubernetes 学习笔记



### 1.ipvs模式

<img src="https://cdn.jsdelivr.net/gh/lj408226003/java-leaning@main/images/27936455-be7a111953afecb1.png" alt="img" style="zoom:80%;" align="left"/>



### 2.iptables模式

<img src="https://cdn.jsdelivr.net/gh/lj408226003/java-leaning@main/images/27936455-81ab44fa30cf9491.png" alt="img" style="zoom:80%;" align="left"/>







### 3.pod创建流程

<img src="https://cdn.jsdelivr.net/gh/lj408226003/java-leaning@main/images/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5rC46L-c5piv5bCR5bm05ZWK,size_20,color_FFFFFF,t_70,g_se,x_16.png" alt="img" style="zoom:30%;" align="left"/>





<img src="https://cdn.jsdelivr.net/gh/lj408226003/java-leaning@main/images/3346f18bb94e4ec28afc9a02fba11994.png" alt="img" style="zoom:70%;" align="left"/>



> 1. 用户通过kubectl或者其他API客户端向apiserver发起创建pod请求。
> 2. apiserver通过对应的kubeconfig进行认证，认证通过后将yaml中的po信息存到etcd。
> 3. Controller-Manager通过apiserver的watch接口发现了pod信息的更新，执行该资源所依赖的拓扑结构整合，整合后将对应的信息交给apiserver，apiserver再写到etcd，此时pod已经可以被调度。
> 4. Scheduler同样通过apiserver的watch接口更新到pod可以被调度，通过算法给pod分配节点，并将pod和对应节点绑定的信息交给apiserver，apiserver再写到etcd，然后将pod交给kubelet。
> 5. kubelet收到pod后，调用CNI接口给pod创建pod网络，调用CRI接口去启动容器，调用CSI进行存储卷的挂载。
> 6. 网络、容器、存储创建完成后，pod创建完成，等业务进程启动后，pod运行成功。



### 4.架构图[本地大图]

### 5.configmap、secret、volumes、pv、pvc

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-test
spec:
  containers:
  volumes:
    - name: volume-def
    
spec:
  containers:
  - name: nginx-nfs
    image: nginx
    volumeMounts:
    - name: nfs-test
      mountPath: /usr/share/nginx/html/
  volumes:
  - name: nfs-test
    nfs:
      server: 192.168.9.185
      path: /nfs/data
```



