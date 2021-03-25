# Stone Lee test for envoy-contro-plane in Redis

首先编译control plane，这里copy了网上一个[envoy control plane example](https://github.com/mnaboka/envoy-control-plane-example)，因为它的cli工具很方便
```
git clone <本repo>
cd <本repo>
cd cmd/control-plan
go build -o controlplane main.go callbacks.go 
./controlplane
```

You can see the following output
```
INFO[0000] Starting API server on 0.0.0.0:8000
INFO[0000] Starting a server on 0.0.0.0:5678
```

controlplane提供两个服务接口，一个是REST，端口8000，一个是GRPC，端口5678

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

下面我们要启动一个envoy座位proxy，让其支持Redis协议，并连接到dynamic resource，i.e., the above control plane

请参考本project里的b.yaml，是一个为envoy配置的支持dynamic control plane 和 Redis协议的yaml

由于envoy(1.17)在2021年初已经全面启动V3协议，所以很多事情很麻烦（我们这个project是用的V2协议），但还好当前envoy是兼容V2的
```
envoy --bootstrap-version 2 -c b.yaml
```

然后，你可以用浏览器连到envoy看一下当前从control plane获得的配置，具体如下
```
http://192.168.64.4:8001/
```
这里的ip是你的envoy的ip，NOTE: 这个IP和control plane没有关系，如果让envoy连到其他地址的control plane，修改b.yaml中```port_value: 5678```上面一行ip地址

然后，可以在浏览器里点看clusters
```
http://192.168.64.4:8001/clusters
```

然后，你需要用本工程的cli.sh加入cluster和endpoint这些运行参数数据，具体如下
```
./cli.sh cluster add redis_cluster 127.0.0.1 anything
./cli.sh endpoint add redis_cluster 127.0.0.1 6001
./cli.sh commit
```

如果每条命令成功，你都可以看到HTTP的response是200， ```HTTP/1.1 200 OK```

我们就设置了一个redis_cluster和一个Redis Server endpoint监听在6001口

如果此时我们用redis-cli连接envoy，由于envoy已经监听在6379端口（详细见b.yaml），我们发现连接可以连上
```
redis-cli 192.168.64.4
```
上面的IP是你的envoy启动的IP，一般情况下，应该和你的客户端, i.e., redis-cli，放在一起（这样，redis-cli命令就是```redis-cli```）

然后你试着在redis-cli执行命令，例如：get key_abc

结果如下：
```
(error) no upstream host
```

这是因为envoy对应的Redis Server还没有启动，你需要根据你上面的./cli.sh命令，在合适的IP上启动redis-server，如下
```
redis-server --port 6001
```

这时，你再在redis-cli里执行```get key_abc```，结果会是```(nil)```（或有效值，取决你的redis-server是否缓存），这说明Redis命令执行成功

NOTE: 真正的生产环境，你应该先启动redis-server，然后再用cli.sh去做配置，上面的示范只是让你更了解整个体系

你可以继续用上面的方式，建立M个envoy，和N个Redis-server(形成M*N的矩阵)，都在redis-cluster这个集群下（有兴趣，还可以建立多个redis clusters），这样，就组成了一个mesh service for Redis

以上的所有都基于Envoy的V2协议，新的Envoy，包括[control plane](https://github.com/envoyproxy/go-control-plane), [Envoy config](https://www.envoyproxy.io/docs/envoy/latest/configuration/configuration)都升级为V3协议（而且强制，当前其文档也比较混乱，比如：Redis如何配置），很多资料网上不足，更重要的是，go control plane example没有支持REST，所以操作起来很不方便，未来等相关社区的工具、文档都针对V3齐备后，情况会好些。

# envoy-control-plane-example

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