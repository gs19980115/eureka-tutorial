# Eureka-tutorial


[Microservices with Spring](https://spring.io/blog/2015/07/14/microservices-with-spring#load-balanced-resttemplate)这个例子对于学习用Eureka实现微服务确实不错，但代码中融入了很多业务逻辑的内容和Spring MVC的代码，就快速上手Eureka而言对新手来说不太友好，我选择的是[Guides](https://spring.io/guides)里的[Service Registration and Discovery](https://spring.io/guides/gs/service-registration-and-discovery/)来入门。

针对用Eureka模拟实现HDFS，我把它修改了一下，方便理解，见[我的Github](https://github.com/gs19980115/eureka-tutorial)（对gradle和maven的用法不太熟，只改了个大概orz，下文会说明运行方法）

## Discovery Server

如同课上所讲，首先需要一个Discovery Server，所有的NameNode和DataNode向Discovery Server发起服务注册，每隔30s续约。

- ### 如何配置启动Discovery Server？

由于我使用的是Eureka做服务发现，可以通过**@EnableEurekaServer**让Spring Boot启动一个 Discovery Server，或者叫Service Registry。如下所示(见eureka-service/src/main/java/EurekaServiceApplication.java)

```Java
@EnableEurekaServer
@SpringBootApplication
public class EurekaServiceApplication {
    
    public static void main(String[] args) {
        SpringApplication.run(EurekaServiceApplication.class, args);
    }
}
```

- ### Discovery Server 如何配置？

对于Eureka Service 可以简单配置如下(见eureka-service/src/main/java/resources/application.properties)：

```properties
# Eureka Server的默认端口
server.port=8761

# 使Eureka Server不会向自己发起注册请求
eureka.client.register-with-eureka=false
eureka.client.fetch-registry=false

logging.level.com.netflix.eureka=OFF
logging.level.com.netflix.discovery=OFF
```

按照上面这样简单配置完，可以按照下面的方式运行

```Shell
$ cd eureka-service
$ gradle build
$ java -jar build/libs/eureka-service-0.0.1-SNAPSHOT.jar
```

一个简单的Eureka Server就可以提供服务了。

- ### Discovery Server如何监听Client的注册请求或下线呢？

找了很久，找到这个解决方法,主要提供几个Listener函数，监听服务注册和服务下线

```Java
	@EventListener
    public void listen(EurekaInstanceCanceledEvent event) throws IOException {
        String uri = "http://" + event.getServerId() + "/";
        System.err.println(uri + " 服务下线");
    }

    @EventListener
    public void listen(EurekaInstanceRegisteredEvent event) {
        InstanceInfo instanceInfo = event.getInstanceInfo();
        String uri = instanceInfo.getHomePageUrl();
        System.err.println(uri + "进行注册");
    }

    @EventListener
    public void listen(EurekaInstanceRenewedEvent event) {
        String uri = "http://" + event.getServerId() + "/";
        System.err.println(uri + " 服务进行续约");

    }

    @EventListener
    public void listen(EurekaRegistryAvailableEvent event) {
        System.err.println("注册中心 启动");
    }

    @EventListener
    public void listen(EurekaServerStartedEvent event) {
        System.err.println("Eureka Server 启动");
    }
```



## NameNode和DataNode

- ### 如何配置启动NameNode和DataNode？

在这个基于微服务架构的分布式系统中，NameNode和DataNode都是Discovery Client，仿照Discovery Server，在对应的EurekaClientApplication.java中加上**@EnableDiscoveryClient**即可，如下所示

```Java
@EnableDiscoveryClient
@SpringBootApplication
public class EurekaClientApplication {

    public static void main(String[] args) {
        SpringApplication.run(EurekaClientApplication.class, args);
    }
}
```

- ### 既然NameNode和DataNode都是Discovery Client，处于对等的地位，那该如何区分这两者呢？

Eureka中是通过application name来区分这两者。针对application name 可以在properties文件中分别配置，如下所示：

1. DataNode中(eureka-client-namenode/src/main/java/resources/bootstrap.properties)

```Pro
spring.application.name=datanode
```

2. NameNode中(eureka-client-datanode/src/main/java/resources/bootstrap.properties)

```
spring.application.name=namenode
```

- ### 如何启动服务？（如何启动多个DataNode实例？）

下面可以启动这两个服务

NameNode

```Shell
$ cd eureka-client-namenode
$ gradle build
$ java -jar build/libs/eureka-client-0.0.1-SNAPSHOT.jar --server.port=8080
```

DataNode
```Shell
# 进入datanode的目录
$ cd eureka-client-datanode
$ gradle build

# 分别占用8081和8082两个端口启动两个DataNode
$ java -jar build/libs/eureka-client-0.0.1-SNAPSHOT.jar --server.port=8081
$ java -jar build/libs/eureka-client-0.0.1-SNAPSHOT.jar --server.port=8082
```

- ### NameNode如何获取所有DataNode节点信息？

可能需要等待30s左右，Eureka才能将这几个服务注册成功，访问[http://localhost:8761](http://localhost:8761),可以看到这几个服务已经启动，大致如下（原本是图形化界面，以下只截取部分相关信息）：

```
# Instances currently registered with Eureka
Application---Status
DATANODE------UP (2) - 192.168.31.250:datanode:8081 , 192.168.31.250:datanode:8082
NAMENODE------UP (1) - 192.168.31.250:namenode:8080
```

可以看到，Datanode和NameNode可以通过Applicationde的名字区分开来，可以很容易想到，NameNode也是可以通过DataNode的Application Name来找到所有DataNode 的信息。继续看NameNode中的代码(见eureka-client-namenode/src/main/java/EurekaClientApplication.java)

```Java
@RestController
class ServiceInstanceRestController {

    // Autowired帮助我们自动注入
    @Autowired
    private DiscoveryClient discoveryClient;

    @RequestMapping("/service-instances/{applicationName}")
    public List<ServiceInstance> serviceInstancesByApplicationName(
            @PathVariable String applicationName) {
        // 核心代码，可以获取所有的Application名为applicationName的所有client信息
        return this.discoveryClient.getInstances(applicationName);
    }
}
```

上述代码中，discoveryClient是由Spring框架自动注入的，能够获取该Server管理的所有client信息，通过**this.discoveryClient.getInstances("datanode");**方法，可以很容易获取所有DataNode的信息。

我们使用[Postman工具](https://www.getpostman.com)来测试一下，由于之前NameNode端口号是8080，向NameNode发起GET请求的链接是http://127.0.0.1:8080/service-instances/datanode，这个类用@RestController注解，所以该请求返回的内容是json格式的，返回结果如下所示：

```json
[
    {
        "host": "192.168.31.250",
        "port": 8082,
        "metadata": {
            "management.port": "8082"
        },
        "secure": false,
        "uri": "http://192.168.31.250:8082",
        "instanceInfo": {
            "instanceId": "192.168.31.250:datanode:8082",
            "app": "DATANODE",
            "appGroupName": null,
            "ipAddr": "192.168.31.250",
            "sid": "na",
            "homePageUrl": "http://192.168.31.250:8082/",
            "statusPageUrl": "http://192.168.31.250:8082/actuator/info",
            "healthCheckUrl": "http://192.168.31.250:8082/actuator/health",
            "secureHealthCheckUrl": null,
            "vipAddress": "datanode",
            "secureVipAddress": "datanode",
            "countryId": 1,
            "dataCenterInfo": {
                "@class": "com.netflix.appinfo.InstanceInfo$DefaultDataCenterInfo",
                "name": "MyOwn"
            },
            "hostName": "192.168.31.250",
            "status": "UP",
            "overriddenStatus": "UNKNOWN",
            "leaseInfo": {
                "renewalIntervalInSecs": 30,
                "durationInSecs": 90,
                "registrationTimestamp": 1529781764747,
                "lastRenewalTimestamp": 1529781764747,
                "evictionTimestamp": 0,
                "serviceUpTimestamp": 1529781764236
            },
            "isCoordinatingDiscoveryServer": false,
            "metadata": {
                "management.port": "8082"
            },
            "lastUpdatedTimestamp": 1529781764747,
            "lastDirtyTimestamp": 1529781764169,
            "actionType": "ADDED",
            "asgName": null
        },
        "serviceId": "DATANODE",
        "scheme": null
    },
    {
        "host": "192.168.31.250",
        "port": 8081,
        "metadata": {
            "management.port": "8081"
        },
        "secure": false,
        "uri": "http://192.168.31.250:8081",
        "instanceInfo": {
            "instanceId": "192.168.31.250:datanode:8081",
            "app": "DATANODE",
            "appGroupName": null,
            "ipAddr": "192.168.31.250",
            "sid": "na",
            "homePageUrl": "http://192.168.31.250:8081/",
            "statusPageUrl": "http://192.168.31.250:8081/actuator/info",
            "healthCheckUrl": "http://192.168.31.250:8081/actuator/health",
            "secureHealthCheckUrl": null,
            "vipAddress": "datanode",
            "secureVipAddress": "datanode",
            "countryId": 1,
            "dataCenterInfo": {
                "@class": "com.netflix.appinfo.InstanceInfo$DefaultDataCenterInfo",
                "name": "MyOwn"
            },
            "hostName": "192.168.31.250",
            "status": "UP",
            "overriddenStatus": "UNKNOWN",
            "leaseInfo": {
                "renewalIntervalInSecs": 30,
                "durationInSecs": 90,
                "registrationTimestamp": 1529781738244,
                "lastRenewalTimestamp": 1529781858198,
                "evictionTimestamp": 0,
                "serviceUpTimestamp": 1529781737737
            },
            "isCoordinatingDiscoveryServer": false,
            "metadata": {
                "management.port": "8081"
            },
            "lastUpdatedTimestamp": 1529781738244,
            "lastDirtyTimestamp": 1529781737682,
            "actionType": "ADDED",
            "asgName": null
        },
        "serviceId": "DATANODE",
        "scheme": null
    }
]
```

可以看到返回了两个DataNode的ServiceInstance转化成Json格式信息，和前面启动的两个DataNode实例相吻合，那么我们只需要提取其中的uri就可以了，ServiceInstance 提供了getUri()方法，直接提取即可。有了URI就可以通过DataNode提供的Restful API进行交互



## 我的实现

以上是我对Eureka的理解，我觉得每次都用"discoveryClient.getInstances(applicationName)"获取所有DataNode节点信息不方便维护管理（可能是我了解的不够全面，不知道有没有别的"Magic"），其实NameNode只需要知道哪个DataNode上线或下线的信息就可以维护所有的DataNode。

1. 一种解决方法是Discovery Server在监听到DataNode上线或下线时主动告诉NameNode（在上述Listener函数中添加相应代码即可），同时NameNode也需要添加接口以处理Discovery Server的通知
2. 在这次实验中，我偷了点懒，试着**把Discovery Server和NameNode结合起来**，NameNode同时也是Discovery Server也就是说所有DataNode直接向NameNode进行服务注册、续约和下线。

- ### 怎么把Discovery Server和NameNode结合起来？

很简单，在NameNodeApplication类里同时加上两个注解**@EnableDiscoveryClient**和**@EnableEurekaServer**

如下所示：

```Java
@SpringBootApplication
@EnableDiscoveryClient
@EnableEurekaServer
public class NameNodeApplication {

    public static void main(String[] args) {
        SpringApplication.run(NameNodeApplication.class, args);
    }
}
```

application.properties文件的内容也全都要，由于这里NameNode作为Discovery Server中的配置不会将自己注册进去，所以也没必要用application的名字区分DataNode和NameNode,如下

```properties
# Eureka Server的默认端口
server.port=8761

# 使Eureka Server不会向自己发起注册请求
eureka.client.register-with-eureka=false
eureka.client.fetch-registry=false

logging.level.com.netflix.eureka=OFF
logging.level.com.netflix.discovery=OFF
```

- ### DataNode 如何配置?

DataNodeApplication只需要加**@EnableEurekaClient**，不多解释

application.properties文件做了一点修改（参考的网上的教程，感觉并不sexy）

```properties
# 不设spring.application.name，是因为在服务下线时
# server能获得到的serverId中间多一个这个名字(大概长这样172.26.23.53:spring-datanode-client:8081)
# 我嫌处理起来麻烦，就，，，就把这个名字删了orz，很暴力hhh
#spring.application.name=spring-datanode-client

# 下面是抄的配置
eureka.client.service-url.defaultZone=${EUREKA_URI:http://localhost:8761/eureka}
eureka.instance.prefer-ip-address=true
```

- ### NameNode如何获取DataNode服务上线和下线的信息?

百度了一下，发现有@EventListener注解可以帮我监听DataNode服务的各种请求，我是在NameNode的Controller里加了以下代码,注释应该详尽，如下

```Java

    // namenode启动
    @EventListener
    public void listen(EurekaServerStartedEvent event) {
        System.err.println("Eureka Server 启动");
    }

	// namenode可以开始接受datanode注册
    @EventListener
    public void listen(EurekaRegistryAvailableEvent event) {
        System.err.println("注册中心 启动");
    }

	// datanode下线，进行数据迁移，保证负载均衡
    @EventListener
    public void listen(EurekaInstanceCanceledEvent event) throws IOException {
        String dataNodeUrl = "http://" + event.getServerId() + "/";
        System.out.println("我要开始删除这个节点"+dataNodeUrl+"，并且迁移数据");
        // nameNodeService.cancelDataNode(dataNodeUrl);
        System.err.println(event.getServerId() + "\t" + event.getAppName() + " 服务下线");
    }

    // datanode 注册，纳入整个系统
    @EventListener
    public void listen(EurekaInstanceRegisteredEvent event) {
        InstanceInfo instanceInfo = event.getInstanceInfo();
        String dataNodeUrl = instanceInfo.getHomePageUrl();
        System.err.println(dataNodeUrl + "进行注册");
        nameNodeService.registerNewDataNode(dataNodeUrl);
    }

    // datanode 续约，告诉namenode他还在
    @EventListener
    public void listen(EurekaInstanceRenewedEvent event) {
        System.err.println(event.getServerId() + "\t" + event.getAppName() + " 服务进行续约");
    }

```
