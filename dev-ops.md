# DevOp系统环境搭建

<img src="https://cdn.jsdelivr.net/gh/lj408226003/java-leaning@main/images/7602.1513404277.png" alt="img" style="zoom:15%;" />



## 0.DevOps是什么？



是面向企业的一站式研发效能平台，

覆盖研发、测试、部署、运维、监控全流程，以求提升研发效能，降低研发成本，支撑技术团队实现真正的CI/CD和敏捷交付为目标。



<img src="https://cdn.jsdelivr.net/gh/lj408226003/java-leaning@main/images/1*57__j14aNQfmPZyFoS1yRg.png" alt="img" style="zoom:60%;" align="left"/>



<img src="https://cdn.jsdelivr.net/gh/lj408226003/java-leaning@main/images/image-20221103003518087.png" alt="image-20221103003518087" style="zoom:30%;" align="left"/>



## 1.运行环境：OMV



本文采用omv nas服务器作为运行环境，当然有条件或根据自己喜好可以选择Proxmox、OpenStack等环境均可。

omv的安装很简单，官网下载iso镜像写入到U盘，引导安装即可。



## 2.Docker环境



因为omv是一个底层linux系统，而他可以直接安装docker服务（通过插件的方式安装）并同时安装portainer图形界面来管理docker容器。具体插件安装就不细说了。

当然如果使用proxmox等虚拟服务器的，可以单台虚拟机安装或组建k8s安装devops的各个组件，这是都可以的，而我这里使用的是omv，所以都在docker中以容器的方式安装各个组件。



> 注意：docker安装完后，避免拉取镜像较慢，需要更换成国内镜像源，或使用阿里云镜像加速，本人使用的是阿里云镜像加速。
>
> https://cr.console.aliyun.com/cn-beijing/instances/mirrors
>
> 我的加速地址是：https://mtu7rhzd.mirror.aliyuncs.com
>
> 将地址给docker的配置即可：/etc/docker/daemon.json
>
> ```shell
> sudo mkdir -p /etc/docker
> sudo tee /etc/docker/daemon.json <<-'EOF'
> {
>   "registry-mirrors": ["https://mtu7rhzd.mirror.aliyuncs.com"]
> }
> EOF
> sudo systemctl daemon-reload
> sudo systemctl restart docker
> ```





## 3.OMV的docker目录规划



因为我的机器是1块128G固态+1块1T机械硬盘，而通过df -h命令可以知道磁盘挂载情况

<img src="../../Library/Application Support/typora-user-images/image-20221101145511333.png" alt="image-20221101145511333" style="zoom:33%;" align="left"/>

如图可知，1T硬盘挂载在”“目录，为了方便我本想将docker的根据目录设置到/srv/dev-disk-by-uuid-27aed415-2dc4-4511-ae70-e0ec850787dd/下的docker目录，在把这个docker通过软连接的方式链接到/docker目录，但是实际操作在omv安装docker指定根目录时，如果用/srv/dev-disk-by-uuid-27aed415-2dc4-4511-ae70-e0ec850787dd/docker会报错，无奈只能使用原始目录作为docker的根目录，即/var/lib/docker，所以只能将这个docker映射到/docker目录，后续有时间可以研究下将docker服务的跟目录改到1T硬盘上。

> 以上都是前期准备知道就行。 



## 4.Gitlab安装：/docker/gitlab



> gitlab就不多说了，这个东西现在大多数公司内部都在使用，它分为社区和企业版本，社区版本ce是免费的，当然也可以选择gitee或github，但由于一些情况gitlab还是最受欢迎的。
>
> 首先在docker服务器根目录创建gitlab目录，表示将所有gitlab容器的数据都会映射到/docker/gitlab目录

### ① 拉取gitlab最新镜像

```sh
docker pull gitlab/gitlab-ce:latest
```



### ② 运行gitlab容器：两种方式



> #### 方式一：通过docker run启动

```
# 启动容器
docker run \
 -itd  \
 -p 8929:80 \
 -p 2224:22 \
 -v /docker/gitlab/config:/etc/gitlab  \
 -v /docker/gitlab/logs:/var/log/gitlab \
 -v /docker/gitlab/data:/var/opt/gitlab \
 -v /etc/timezone:/etc/timezone:ro \
 -v /etc/localtime:/etc/localtime:ro \
 --network bloom-net \
 --shm-size 256m \
 --restart always \
 --privileged=true \
 --name gitlab \
 gitlab/gitlab-ce:latest
```

| 参数                                 | 描述                                                         |
| ------------------------------------ | ------------------------------------------------------------ |
| -i                                   | 以交互模式运行容器，通常与 -t 同时使用                       |
| -t                                   | 为容器重新分配一个伪输入终端，通常与 -i 同时使用             |
| -d                                   | 后台运行容器，并返回容器ID                                   |
| -p 8929:80                           | 将容器内80端口映射至宿主机8929端口，这是访问gitlab的端口     |
| -p 2224:22                           | 将容器内22端口映射至宿主机2224端口，这是访问ssh的端口        |
| -v /docker/gitlab/config:/etc/gitlab | 将容器/etc/gitlab目录挂载到宿主机/docker/gitlab/config目录下，若宿主机内此目录不存在将会自动创建，其他两个挂载同理 |
| -v /etc/timezone:/etc/timezone:ro    | 修正容器内时钟差8小时问题                                    |
| -v /etc/localtime:/etc/localtime:ro  | 修正容器内时钟差8小时问题                                    |
| --shm-size                           | 设置docker占用宿主机内存大小                                 |
| --restart always                     | 容器在出现异常或宿主机重启，该容器随宿主机的docker服务自动重启 |
| --privileged=true                    | 让容器获取宿主机root权限                                     |
| --name gitlab                        | 设置容器名称为gitlab                                         |
| gitlab/gitlab-ce:latest              | 镜像的名称，这里也可以写镜像ID                               |



> #### 方式二：通过docker-compose.yml运行
>
> `优势：该方式一步到位，省去了《方式一》运行后还需要单独配置对外开放的host和port。`

我们知道docker-compose是可以编排复杂容器关系的工具，该文件中可以通过设置docker容器的多维属性并能同时创建运行多个容器，可以和kubernetes的yaml类比。

```shell
# 创建vi docker-compose.yml文件，保存到宿主机的/docker/gitlab目录。

version: '3.8'
services:
  web:
    image: 'gitlab/gitlab-ce:latest'  # 使用的gitlab镜像版本[其他参数见方案一解释]
    restart: always
    container_name: 'gitlab'
    hostname: 'gitlab'
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'http://10.10.1.199:8929'
        gitlab_rails['gitlab_shell_ssh_port'] = 2224
    ports:
      - '8929:80'
      - '2224:22'
      # - '9443:443' # 如果需要https访问则需要映射443端口
    networks:
      - 'exist-net-bloom' # 当前创建的容器加入到外部的bloom网络
    volumes:
      - '/docker/gitlab/config:/etc/gitlab'
      - '/docker/gitlab/logs:/var/log/gitlab'
      - '/docker/gitlab/data:/var/opt/gitlab'
      - '/etc/timezone:/etc/timezone:ro'
      - '/etc/localtime:/etc/localtime:ro'
    shm_size: '256m'
networks:
  exist-net-bloom: # 改名字为当前docker-compose中要引用的名字
    external: # 指定已经存在的网络名字
      name: bloom-net
```

```shell
# 命令行执行，运行docker编排内容
docker-compose up -d
# 观察docker运行日志
docker logs -f gitlab
```



> #### 统一配置部分：

<font size='2px'>`-> 修改docker容器内的gitlab配置，完成gitlab容器的高级映射[避免修改宿主机映射的目录文件]`</font>

```shell
1.修改容器中vi /etc/gitlab/gitlab.rb配置，增加对外映射的ip，可以认为指定对外开放哪个ip可以访问

# 进容器内部
docker exec -it gitlab /bin/bash
 
# 修改gitlab.rb
vi /etc/gitlab/gitlab.rb
 
# 加入如下配置内容
# gitlab访问地址，可以写域名。如果端口不写的话默认为80端口，10.10.1.199是我omv(宿主机)的ip地址，大家可以根据自己的情况设置
external_url 'http://10.10.1.199'

#ssh主机ip
gitlab_rails['gitlab_ssh_host'] = '10.10.1.199'
#ssh连接端口
gitlab_rails['gitlab_shell_ssh_port'] = 2224
 
# 让配置生效
gitlab-ctl reconfigure
```

```shell
2.修改容器中vi /opt/gitlab/embedded/service/gitlab-rails/config/gitlab.yml
 
  gitlab:
    host: 10.10.1.199 # 因为上面的修改生效后会自动更新到改文件，所以host不用改
    port: 8929 # 这里需要改成外部映射的8929端口
    https: false # 如果想通过https访问，此处设置成true
    
# 让配置生效
gitlab-ctl restart

3.安装完后进入容器查看默认gitlab密码，用户名: root
sudo docker exec -it gitlab grep 'Password:' /etc/gitlab/initial_root_password
```

> `注意：`安装完后查看容器日志，可能会发现如下错误
>
> ```java
> err="create memberlist: Failed to get final advertise address: No private IP address found, and explicit IP not provided"
> ```
>
> 错误原因是因为alertmanager配置的监听地址是localhost，此处需要指定127.0.0.1，修改vi /etc/gitlab/gitlab.rb如下：
>
> ```json
> alertmanager['flags'] = {
>   'cluster.advertise-address' => "127.0.0.1:9093"
> }
> 
> # 指定后使配置生效即可
> gitlab-ctl reconfigure
> ```



### ③ 其他配置

> `邮箱配置：`用于用户操作时推送关键性信息
>
> ```
> # 1.修改容器中配置文件vi /etc/gitlab/gitlab.rb
> ### GitLab email server settings                                                          
> ###! Docs: https://docs.gitlab.com/omnibus/settings/smtp.html                      
> ###! **Use smtp instead of sendmail/postfix.**                                     
>                                                                                               
> gitlab_rails['smtp_enable'] = true                                                 
> gitlab_rails['smtp_address'] = "smtp.126.com"                                      
> gitlab_rails['smtp_port'] = 465                                                                
> gitlab_rails['smtp_user_name'] = "xxx@126.com"                             
> gitlab_rails['smtp_password'] = "xxx"                                         
> gitlab_rails['smtp_domain'] = "smtp.126.com"                                                             
> gitlab_rails['smtp_authentication'] = "login"                                                                 
> gitlab_rails['smtp_enable_starttls_auto'] = true                                           
> gitlab_rails['smtp_tls'] = true               
> 
> gitlab_rails['gitlab_email_from'] = 'xxx@126.com'                          
> gitlab_rails['gitlab_email_display_name'] = 'gitlab'
> 
> user['git_user_email'] = "xxx@126.com"  
> 
> # 2.重启配置
> gitlab-ctl reconfigure
> gitlab-ctl restart 
> 
> # 3.测试：首先进入控制台
> gitlab-rails console
> # 执行测试命令
> Notify.test_email("接收邮箱","标题","内容").deliver_now
> 
> # 4.输出如下内容就是成功了
> root@gitlab:/# gitlab-rails console
> --------------------------------------------------------------------------------
>  Ruby:         ruby 2.7.5p203 (2021-11-24 revision f69aeb8314) [x86_64-linux]
>  GitLab:       14.6.1 (661d663ab2b) FOSS
>  GitLab Shell: 13.22.1
>  PostgreSQL:   12.7
> --------------------------------------------------------------------------------
> Loading production environment (Rails 6.1.4.1)
> irb(main):001:0> Notify.test_email("xxx@126.com","ceshi","内容").deliver_now
> Delivered mail 636286d3cea5d_35a95adc58939@gitlab.mail (1436.8ms)
> => #<Mail::Message:154820, Multipart: false, Headers: <Date: Wed, 02 Nov 2022 23:03:47 +0800>, <From: gitlab <xxx@126.com>>, <Reply-To: gitlab <noreply@10.10.1.199>>, <To: xxx@126.com>, <Message-ID: <636286d3cea5d_35a95adc58939@gitlab.mail>>, <Subject: ceshi>, <Mime-Version: 1.0>, <Content-Type: text/html; charset=UTF-8>, <Content-Transfer-Encoding: 7bit>, <Auto-Submitted: auto-generated>, <X-Auto-Response-Suppress: All>>
> ```

> `【坑】`按照以上方式启动docker会遇到一个坑，就是注册用户后，给新用户发邮件需要新用户点击邮件中的链接去重置密码，而链接点击后却是空白页？
>
> ==> 这是因为邮件的链接是指向了10.10.1.199:80，并没有使用我们映射到宿主机的8929端口，这是因为我们在配置external_url 'http://10.10.1.199'时用的是默认的80，
>
> 那就简单了，直接指定external_url 'http://10.10.1.199:8929'不就行了吗？
>
> 这样改问题更大，直接通过http://10.10.1.199:8929都不能访问gitlab了，那怎么办？
>
> 
>
> 其实问题出在docker创建时的端口映射，我们把容器的80->映射到了宿主的8929，而宿主内部使用的是80，而external_url 'http://10.10.1.199'对外的地址也是80，
>
> 所以改起来也很简单，就是让宿主机内部也是用8929，这样在external_url这里就可以使用external_url 'http://10.10.1.199:8929'了。
>
> `更正后的docker-compose.yml如下：`
>
> ```shell
> version: '3.8'
> services:
>   web:
>     image: 'gitlab/gitlab-ce:latest'  # 使用的gitlab镜像版本[其他参数见方案一解释]
>     restart: always
>     container_name: 'gitlab'
>     hostname: 'gitlab'
>     environment:
>       GITLAB_OMNIBUS_CONFIG: |
>         external_url 'http://10.10.1.199:8929'
>         gitlab_rails['gitlab_shell_ssh_port'] = 2224
>     ports:
>       - '8929:8929'
>       - '2224:22'
>       # - '9443:443' # 如果需要https访问则需要映射443端口
>     networks:
>       - 'exist-net-bloom' # 当前创建的容器加入到外部的bloom网络
>     volumes:
>       - '/docker/gitlab/config:/etc/gitlab'
>       - '/docker/gitlab/logs:/var/log/gitlab'
>       - '/docker/gitlab/data:/var/opt/gitlab'
>       - '/etc/timezone:/etc/timezone:ro'
>       - '/etc/localtime:/etc/localtime:ro'
>     shm_size: '256m'
> networks:
>   exist-net-bloom: # 改名字为当前docker-compose中要引用的名字
>     external: # 指定已经存在的网络名字
>       name: bloom-net
> ```
>
> `疑惑：`有的小伙伴可能注意了，你的GITLAB_OMNIBUS_CONFIG环境变量中不是指定了external_url 'http://10.10.1.199:8929'吗？怎么实际设置vi /etc/gitlab/gitlab.rb的时候却使用了external_url 'http://10.10.1.199'不带端口的？
>
> --> 根据官方的解释，GITLAB_OMNIBUS_CONFIG这个变量可以指定所有gitlab.rb配置文件中的属性，但这个不会写入到容器的gitlab.rb配置文件，所以他只是一个临时的配置，实际运行是还是使用的gitlab.rb中的配置。
>
> `疑惑：`既然不生效，为什么要在docker-compose.yml中指定呢？
>
> --> 其实直接说不生效并不准确，通过GITLAB_OMNIBUS_CONFIG指定的参数，在我不重新读gitlab.rb配置时确实是生效的，但是当重新执行gitlab-ctl reconfigure后就会以gitlab.rb中的配置为准了。而我们以上操作确实在启动docker容器后又改了gitlab.rb并重启加载，这样就导致GITLAB_OMNIBUS_CONFIG是失效的。
>
> `细节：`其实不知道大家有没有注意，如果按照我上边的操作，执行完docker-compose.yml启动容器后，直接访问http://10.10.1.199:8926是无响应的，就是因为这个时候GITLAB_OMNIBUS_CONFIG的设置生效了，因为这里我指定的external_url 'http://10.10.1.199:8929'是带端口的，而宿主和容器的映射却是8929->80。
>
> `最后一步：`所以按照正确的修正，修改了docker-compose.yml中的端口映射后，还需要修改一下vi /etc/gitlab/gitlab.rb才算完整：
>
> ```
> external_url 'http://10.10.1.199:8929'
> ```
>
> `重新部署gitlab：`因为修改了docker-compose.yml，所以需要更新一下配置并构建
>
> ```shell
> # 关闭并移除现有gitlab容器
> docker-compose down
> # 重构并启动：【之前的配置并不会丢失】
> docker-compose up -d --build
> ```



### ④ 常用命令

| 命令                    | 描述                                                         |
| ----------------------- | ------------------------------------------------------------ |
| gitlab-ctl reconfigure  | 重载配置                                                     |
| gitlab-ctl check-config | 检查配置并启动                                               |
| gitlab-ctl diff-config  | 将用户配置与包可用配置进行比较                               |
| gitlab-ctl status       | 查看所有启动组件的进程和状态                                 |
| gitlab-ctl service-list | 查看所有服务                                                 |
| gitlab-ctl stop         | 停止GitLab服务                                               |
| gitlab-ctl start        | 启动GitLab服务                                               |
| gitlab-ctl restart      | 重启GitLab服务                                               |
| gitlab-ctl once         | 如果GitLab服务已停止则启动服务，如果GitLab服务已启动则重启GitLab服务 |

> 以上参考官方文档：https://docs.gitlab.cn/jh/install/docker.html



## 5.Jenkins安装



根据jenkins官网对自己的描述，他是一个可集成有上百个插件的自动化服务，提供构建、部署和自动化的工程，可以说是opsdev的大总管，将开发的代码工程与环境紧密结合起来。

以实现持续开发、持续集成、持续发布的能力。



### ① 拉取jenkins lts长期稳定版本

```
docker pull jenkins/jenkins:2.361.2-lts
```



### ② 运行jenkins容器

为了方便维护，后续统一使用docker-compose.yml

```
# 1.首先在宿主机创建jenkins的容器目录/docker/jenkins
mkdir /docker/jenkins

# 2.因为jenkins容器启动后会自动使用jenkins用户(所属组为1000)来进行操作，所以需要给宿主机的/docker/jenkins目录授权，此处为简单我们直接使用777授权
chmod -R 777 /docker/jenkins

# 3.编写docker-compose.yml文件：vi docker-compose.yml

```

