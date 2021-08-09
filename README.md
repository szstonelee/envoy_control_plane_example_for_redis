# Tony test for envoy-contro-plane in Redis

## 前言

这里是基于[Envoy](https://envoyproxy.io/)这个代理工具，然后用一个最简模式的[Control Plane](https://hub.fastgit.org/envoyproxy/go-control-plane)，实现Redis（以及BunnyRedis）的Auto Rebalance和Auto Failover。

这涉及下面几个知识点：

1. 在一台专用机器上，编译和运行一个conntrol plane
2. 在多个你的Redis client机器上，安装Envoy proxy
3. 配至整个集群，本文以[BunnyRedis](https://hub.fastgit.org/szstonelee/bunnyredis/wiki)为例，Redis类似
4. 运行集群服务器
5. 相关测试

## 下载Control Plane和编译

### 下载源码和编译（需要Golang支持）

参考：[Golang下载和安装](https://golang.org/dl/)

首先编译control plane，这里copy了网上一个[envoy control plane example](https://github.com/mnaboka/envoy-control-plane-example)，因为它的**cli.sh**工具很方便

在control plane专用机器上，比如：192.168.0.11
```
git clone https://github.com/szstonelee/envoy_control_plane_example_for_redis
cd envoy_control_plane_example_for_redis
cd cmd/control-plane
go build -o controlplane main.go callbacks.go 
./controlplane
```

上面也可以用https://hub.fastgit.org/szstonelee/envoy_control_plane_example_for_redis，这个镜像站点

You can see the following output
```
INFO[0000] Starting API server on 0.0.0.0:8000
INFO[0000] Starting a server on 0.0.0.0:5678
```

controlplane提供两个服务接口，一个是REST，端口8000，一个是gRPC，端口5678

### Golang中国编译技巧

Go编译注意事项，特别是在中国。Go编译时，需要动态到GitHub下载一些dependency，但中国的防火墙有封锁，[不过一些人给出了解决方案](https://github.com/golang/go/issues/31755)，具体如下（你可能需要购买到国外的proxy服务）

首先设置go environment
```
go env -w GOPROXY=direct
go env -w GOSUMDB=sum.golang.google.cn
```

然后设置两个环境变量，为了方便，在你的Linux账户环境里设置，建议设置~/.bashrc，因为这个对VS Code有效，而~/.bash_profile只对SSH session有效
```
sudo vi ~/.bashrc
在文件里加入下面两行
export GOPROXY=https://goproxy.cn
export GOSUMDB=sum.golang.org
存盘退出后
source ~/.bashrc
```

## Envoy代理服务器和相关配置

### 安装Envoy

在各个Redis client机器上（比如：192.168.0.22， 192.168.0.33）上，安装Envoy proxy

我们将Envoy proxy和Redis client装在同一台机器上，做成side-car模式，这也是推荐的模式（分开也可以，但不推荐）

请参考[Envoy官网的Envoy安装说明](https://www.envoyproxy.io/docs/envoy/latest/start/install)

https://www.envoyproxy.io/docs/envoy/latest/start/install

对于CentOS
```
sudo yum install yum-utils
sudo rpm --import 'https://rpm.dl.getenvoy.io/public/gpg.CF716AF503183491.key'
curl -sL 'https://rpm.dl.getenvoy.io/public/config.rpm.txt?distro=el&codename=7' > /tmp/tetrate-getenvoy-rpm-stable.repo
sudo yum-config-manager --add-repo '/tmp/tetrate-getenvoy-rpm-stable.repo'
sudo yum makecache --disablerepo='*' --enablerepo='tetrate-getenvoy-rpm-stable'
sudo yum install getenvoy-envoy
```

### 启动Envoy proxy和相关配置

下面我们要启动一个Envoy作为proxy代理服务器，让其支持Redis，并连接到dynamic resource，i.e., the above control plane

Envony prox的配至文件
```
请参考：本工程根目录下的b.yaml文件
```

同时，需要修改control plane的地址和端口，具体如下：

在b.yaml文件里，请找到
```
                address: 127.0.0.1
                port_value: 5678
```

然后，请在各个Envoy proxy机器上（本例中的192.168.0.22和192.168.0.33）的，修改这个b.yaml文件这两行，如上例中，control plane装在192.168.0.11这机器上，请修改如下
```
                address: 192.168.0.11
                port_value: 5678
```

请参考本project里根目录下的b.yaml，是一个为Envoy proxy配置的支持dynamic control plane 和 Redis协议的yaml

由于Envoy(1.17)在2021年初已经全面启动V3协议，所以很多事情很麻烦（我们这个project是用的V2协议），但还好当前Envoy proxy是兼容V2的。所以，我们启动Envoy proxy用如下的命令
```
envoy --bootstrap-version 2 -c b.yaml
```

### 测试Envoy proxy

然后，可以在浏览器里点看clusters
```
http://192.168.0.22:8001/clusters
http://192.168.0.33:8001/clusterss
```

## control plane自带的cli.sh工具

### 说明

这个工具将用于control plane里对集群的管理，包括：设置cluster、bunny-redis server endpoint

请参考本工程的根目录下的cli.sh文件

同时，我们这个cli.sh是假设和control plane运行在同一机器上（即192.168.0.11）。如果想用cli.sh用于其他机器上，请修改cli.sh文件。

### 操作范例

为了简单，我们假设布置BunnyRedis中的bunny-redis集群也运行在192.168.0.22和192.168.0.33机器上，即Redis client、Envoy proxy、bunny-redis都位于同一机器上。由于Envoy proxy已经监听于6379端口，所以，我们需要给bunny-redis一个新的端口6380。

实际生产环境中，bunny-redis很可能位于单独的机器上，这样，bunny-redis也可以监听于6379这个缺省Redis端口上。大家可以修改下面相应的参数。

然后，你需要用本工程的cli.sh加入cluster和endpoint这些运行参数数据，具体如下
```
./cli.sh cluster add bunny_redis_cluster
./cli.sh endpoint add bunny_redis_cluster 192.168.0.22 6380
./cli.sh endpoint add bunny_redis_cluster 192.168.0.33 6380
./cli.sh commit
```

如果每条命令成功，你都可以看到HTTP的response是200， ```HTTP/1.1 200 OK```

我们就设置了一个redis_cluster和一个Redis server（或者[BunnyRedis](github.com/szstonelee/bunnyredis/wiki)的bunny-redis进程） endpoint监听在6001口

## 测试Envoy proxy以及control plane

### 先测试Envoy proxy

如果此时我们用redis-cli连接Envoy proxy，由于Envoy proxy已经监听在6379端口（详细见b.yaml），我们发现连接可以连上
```
redis-cli 192.168.64.4
```
上面的IP是你的Envoy proxy启动的IP。

一般情况下，应该Envoy proxy和你的Redis客户端程序（例如：redis-cli）放在一台机器上，成为side-car模式, 这样，redis-cli命令就可以简化为
```
redis-cli
```

### 再测试control plane

然后你试着在redis-cli执行命令，例如：get key_abc

结果如下：
```
(error) no upstream host
```

这是因为Envoy系统里对应的Redis Server endpoint还没有启动，你需要根据你上面的./cli.sh命令，在合适的IP上启动redis-server，如下
```
redis-server --port 6001
```

这时，你再在redis-cli里执行```get key_abc```，结果会是```(nil)```（或有效值，取决你的redis-server是否缓存），这说明Redis命令执行成功

NOTE: 真正的生产环境，你应该先启动redis-server，然后再用cli.sh去做配置，上面的示范只是让你更了解整个体系

你可以继续用上面的方式，建立M个envoy，和N个Redis-server(形成M*N的矩阵)，都在redis-cluster这个集群下（有兴趣，还可以建立多个redis clusters），这样，就组成了一个mesh service for Redis

以上的所有都基于Envoy的V2协议，新的Envoy，包括[control plane](https://github.com/envoyproxy/go-control-plane), [Envoy config](https://www.envoyproxy.io/docs/envoy/latest/configuration/configuration)都升级为V3协议（而且强制，当前其文档也比较混乱，比如：Redis如何配置），很多资料网上不足，更重要的是，go control plane example没有支持REST，所以操作起来很不方便，未来等相关社区的工具、文档都针对V3齐备后，情况会好些。

## Other reference

Example of the control plane server for envoy.

This is an example implementation of envoy control plane written in golang.
Envoy is extremely powerful, but learning curve might be steep.

In this example we are dynamically updating Clusters, Endpoints and Routes.
More details [here](https://www.envoyproxy.io/docs/envoy/latest/configuration/overview/v2_overview)

docker-compose contains the following containers:
  - `control-plane` - implementation of xDS server. It listens for xDS DiscoveryRequests on port `:5678`
    and exposes REST API on port `8000` to dynamically add/remove clusters and endpoints(upstreams).
    CLI script contains examples how to interact with REST API [cli.sh](https://github.com/mnaboka/envoy-control-plane-example/blob/master/cli.sh)

  - `envoy` - is an official image of envoy v1.11.1, it exposes admin interface on port `:9901` and HTTP1 listener on `:3000`
  - `test-server-dev` and `test-server-prod` simple HTTP servers listens on `:8080` and returns info about server (hostname, ip address etc.)

### How to use CLI
```bash
Usage:
./cli.sh cluster add <name> <url_prefix>
./cli.sh cluster remove <name>
./cli.sh endpoint add <cluster_name> <ip_address> <port>
./cli.sh endpoint remove <cluster_name> <ip_address> <port>
./cli.sh commit
```

Examples:

To add a new cluster with a URL prefix `/api/v1` and 2 upstream hosts. When changes are made, they must be committed with
`commit` subcommand.

```bash
cli.sh cluster add backend-cluster /api/v1
cli.sh endpoint add backend-cluster 10.10.0.1 8080
cli.sh endpoint add backend-cluster 10.10.0.2 8080
cli.sh commit
```

To verify changes, you can navigate to envoy admin interface:
  - [config_dump](http://127.0.0.1:9901/config_dump)
  - [clusters](http://127.0.0.1:9901/clusters)



### How to use this repo
  - start with cloning this repo on a local machine

```bash
git clone git@github.com:mnaboka/envoy-control-plane-example.git
```

  - start up docker-compose and scale the `test-server-dev` and `test-server-prod` to 3 instances

```bash
docker-compose up -d --scale test-server-prod=3 --scale test-server-dev=3 --no-recreate
```

  - run `docker-compose ps` to see all the running instances, we should have 8 containers running
  
```bash
$ docker-compose ps
                     Name                                   Command               State                             Ports
---------------------------------------------------------------------------------------------------------------------------------------------------
envoy-control-plane-example_control-plane_1      /go/bin/control-plane            Up      0.0.0.0:5678->5678/tcp, 0.0.0.0:8000->8000/tcp, 8080/tcp
envoy-control-plane-example_envoy_1              /docker-entrypoint.sh /usr ...   Up      10000/tcp, 0.0.0.0:3000->3000/tcp, 0.0.0.0:9901->9901/tcp
envoy-control-plane-example_test-server-dev_1    /go/bin/server                   Up      5678/tcp, 8000/tcp, 8080/tcp
envoy-control-plane-example_test-server-dev_2    /go/bin/server                   Up      5678/tcp, 8000/tcp, 8080/tcp
envoy-control-plane-example_test-server-dev_3    /go/bin/server                   Up      5678/tcp, 8000/tcp, 8080/tcp
envoy-control-plane-example_test-server-prod_1   /go/bin/server                   Up      5678/tcp, 8000/tcp, 8080/tcp
envoy-control-plane-example_test-server-prod_2   /go/bin/server                   Up      5678/tcp, 8000/tcp, 8080/tcp
envoy-control-plane-example_test-server-prod_3   /go/bin/server                   Up      5678/tcp, 8000/tcp, 8080/tcp

```
  
  - find ip addresses for our `test-server-dev` instances
  
```bash
$ docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' envoy-control-plane-example_test-server-dev_1 envoy-control-plane-example_test-server-dev_2 envoy-control-plane-example_test-server-dev_3
192.168.240.7
192.168.240.4
192.168.240.8
```

  - create a new envoy cluster called "dev" with provided `cli.sh`
```bash
$ ./cli.sh cluster add dev /dev

HTTP/1.1 200 OK
Date: Wed, 18 Sep 2019 00:46:09 GMT
Content-Length: 0

$ ./cli.sh endpoint add dev 192.168.240.7 8080

HTTP/1.1 200 OK
Date: Wed, 18 Sep 2019 00:46:27 GMT
Content-Length: 0

$ ./cli.sh endpoint add dev 192.168.240.4 8080

HTTP/1.1 200 OK
Date: Wed, 18 Sep 2019 00:46:32 GMT
Content-Length: 0

$ ./cli.sh endpoint add dev 192.168.240.8 8080

HTTP/1.1 200 OK
Date: Wed, 18 Sep 2019 00:46:38 GMT
Content-Length: 0

$ ./cli.sh commit

HTTP/1.1 200 OK
Date: Wed, 18 Sep 2019 00:47:59 GMT
Content-Length: 0
```

  - curl envoy to see if it proxies the requests
  
```bash
17:49 $ curl 127.0.0.1:3000/dev 2>/dev/null| jq '.'
{
  "env": "development",
  "hostname": "16a1355951b2",
  "ips": [
    "127.0.0.1/8",
    "192.168.240.8/20"
  ]
}
```

  - verify changes, you can navigate to envoy admin interface:
    - [config_dump](http://127.0.0.1:9901/config_dump)
    - [clusters](http://127.0.0.1:9901/clusters)
  - optionally repeat steps for test-server-prod
  - great! looks like envoy proxies to the correct upstream host, congrats!