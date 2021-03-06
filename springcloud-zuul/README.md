# Spring Cloud Eureka 服务治理

**1.父pom依赖, 使用的Boot-1.59 Cloud-Edgware**
```
    <!--Spring Boot 父依赖-->
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>1.5.9.RELEASE</version>
        <relativePath/>
    </parent>
      
      
    <dependencies>
      <!--Spring Boot 测试模块-->
      <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
      </dependency>
  
      <!--Spring Boot 项目管理-->
      <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
      </dependency>
    </dependencies>
    
    <!--Spring Cloud 引用 各模块的版本等-->
    <dependencyManagement>
        <dependencies>
          <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>Edgware.RELEASE</version>
            <type>pom</type>
            <scope>import</scope>
          </dependency>
        </dependencies>
    </dependencyManagement>
```

**2.使用Feign继承特性, 创建API模块管理接口和DTO对象**

- pom中引入web模块

```
    <dependencies>
        <!--引入Web模块-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
    </dependencies>
```

- 维护DTO对象
```
    //注意实现序列化接口
    public class UserDTO implements Serializable {}
```

- 创建发布消费的REST接口, 需要注意在Feign中@RequestParam必须指定后面的参数名

```
    public interface UserRest {
    
        /**
         * 获取通过ID查询用户
         * @param id
         * @return
         */
        @GetMapping("/user/{id}")
        UserDTO findById (@PathVariable("id") Long id);
    }
```


**3.配置注册中心：eureka-server**

- pom依赖
```
    <dependencies>
        <!--Eureka Server 注册中心依赖-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-eureka-server</artifactId>
        </dependency>
    </dependencies>
```

- application.yml配置

```
    server:
      port: 8081  #注册中心端口
    
    spring:
      application:
        name: eureka-server #服务名称
    
    eureka:
      instance:
        hostname: localhost #实例地址
      client:
        serviceUrl:
          defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/  #访问地址
        register-with-eureka: false #不注册自己
        fetch-registry: false #注册中心职责就是维护实例, 不需要检索服务
```

- EurekaServerApplication：boot启动类

```
    //Spring Boot 应用-启动入口
    @SpringBootApplication
    //自动启动Eureka注册中心
    @EnableEurekaServer
    public class EurekaServerApplication {
    
        public static void main(String[] args) {
            SpringApplication.run(EurekaServerApplication.class, args);
        }
    }
```

**4.注册服务**

- pom依赖-引入服务发布相关依赖

```
    <dependencies>
        <!--Eureka服务发布组件 web服务 ribbon负载均衡等-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-eureka</artifactId>
        </dependency>
        
        <!--通用API 服务注册发现共同依赖-->
        <dependency>
            <groupId>org.ko</groupId>
            <artifactId>api</artifactId>
            <version>1.0-SNAPSHOT</version>
        </dependency>
    </dependencies>
```

- application.yml配置

```
    # 端口号
    server.port: 8082
    # 定义服务名
    spring.application.name: user
    # 定义注册中心地址
    eureka.client.serviceUrl.defaultZone: http://localhost:8081/eureka/
```

- 创建Controller对应发布接口, 使用DiscoveryClient对象获取服务相关信息

```
    @RestController
    @RequestMapping("services")
    public class ServicesController {
    
        //获取服务相关信息
        @Autowired private DiscoveryClient client;
    
        @GetMapping
        public String getServices() {
            String desc = client.description();
            return desc;
        }
    
    }
```

- 创建UserController实现UserRest接口, 暴露给外部调用
```
    @RestController
    public class UserController implements UserRest {
    
        @Autowired private UserService userService;
    
    }
```

- 创建UserService, 处理业务, 设置当插入用户大于2时发生异常
```
    @Service
    public class UserService {
        /**
         * 模拟数据库
         */
        private static Map<Long, User> data = new ConcurrentHashMap<>();
        
        /**
         * 查询全部用户
         * @return
         */
        public List<UserDTO> findAll() {
            Collection<User> users = data.values();
            //当users数量大于2时发生异常
            if (users.size() > 2) {
                throw new RuntimeException("User overflow.");
            }
    
            List<UserDTO> userDTOS = new ArrayList<>();
            if (!CollectionUtils.isEmpty(users)) {
                userDTOS = users.stream().map(user -> {
                    UserDTO userDTO = new UserDTO();
                    BeanUtils.copyProperties(user, userDTO);
                    return userDTO;
                }).collect(Collectors.toList());
            }
            return userDTOS;
        }
    }
```

- ComputeServiceApplication: boot启动类

```
    //Spring Boot 程序
    @SpringBootApplication
    //向Eureka注册中心发布服务
    @EnableEurekaClient
    public class ComputeServiceApplication {
    
        public static void main(String[] args) {
            SpringApplication.run(ComputeServiceApplication.class, args);
        }
    }
```

**5.使用Feign消费服务**

- pom依赖
```
    <dependencies>
        <!--Feign客户端依赖-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-feign</artifactId>
        </dependency>

        <!--Eureka服务治理依赖-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-eureka</artifactId>
        </dependency>

        <!--服务维护通用API模块-->
        <dependency>
            <groupId>org.ko</groupId>
            <artifactId>api</artifactId>
            <version>1.0-SNAPSHOT</version>
        </dependency>
    </dependencies>
```

- application.yml配置
```
    # 端口号
    server.port: 8083
    # 定义服务名
    spring.application.name: ribbon-consumer
    # 定义注册中心地址
    eureka.client:
      serviceUrl.defaultZone: http://localhost:8081/eureka/
      register-with-eureka: false #客户端不发布服务
  
    # 开启Feign客户端Hystrix熔断
    feign.hystrix.enabled: true
    
    # Ribbon全局配置
    ribbon:
      ConnectTimeout: 250               #ribbon请求连接的超时时间
      ReadTimeout: 1000                 #请求处理的超时时间
      OkToRetryOnAllOperations: true    #对所有操作请求都进行重试
      MaxAutoRetries: 1                 #对当前实例的重试次数
      MaxAutoRetriesNextServer: 1       #对下个实例的重试次数
    
    # Hystrix全局超时时间
    hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds: 2000
```

- 创建Controller

```
    //REST
    @RestController
    public class UserController {
        @Autowired private UserService userService;
    }
```

- 创建Service接口集成API模块UserRest, 使用@FeignClient注解开启Feign客户端, fallback配置降级处理

```
    /**
     * 开启Feign客户端
     * #{@link FeignClient}
     * value 服务名称
     * fallback Hystrix 熔断后降级处理
     * configuration Feign客户端初始化 不配置使用默认
     */
    @FeignClient(value = "user", fallback = UserClientHystrix.class, configuration = FeignLoggerConfig.class)
    public interface UserClientService extends UserRest {
    
    }
    
```

- 创建Hystrix断路器降级处理, 创建UserClientHystrix类实现UserClientService
```
    @Component
    public class UserClientHystrix implements UserClientService {}
```

- FeignApplication: 启动类
```
    //SpringBoot启动
    @SpringBootApplication
    //开启Eureka注册中心连接
    @EnableDiscoveryClient
    //开启Feign客户端
    @EnableFeignClients
    public class FeignApplication {
    
        public static void main(String[] args) {
            SpringApplication.run(ConsumerApplication.class, args);
        }
    }
```

**6.Zuul网关配置**

- yml依赖
```
    <dependencies>
        <!--Zuul 网关依赖-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-zuul</artifactId>
        </dependency>

        <!--开启面向服务路由-Eureka依赖-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-eureka</artifactId>
        </dependency>
    </dependencies>
```
- application.yml配置
```
    spring:
      application:
        name: api-gateway #Spring应用名称-SOA服务名
    server:
      port: 8084 #应用端口号
    
    
    #Zuul路由简单配置-缺点: path和url比较难以维护
    
    #zuul:
    #  routes:
    #    user:
    #      path: /feign/**              #匹配Url格式
    #      url: http://localhost:8083/  #映射Url地址
    #----------------------------------------
    
    
    #Zuul面向服务路由
    
    # 定义注册中心地址
    eureka.client.serviceUrl.defaultZone: http://localhost:8081/eureka/
    # 定义Zuul路由
    zuul:
      routes:
        feign:    #eureka-feign实例
          path: /feign/**
          serviceId: feign-consumer
        compute:  #compute-service实例
          path: /compute/**
          serviceId: user
```
- Zuul网关, 请求过滤. 继承ZuulFilter 实现抽象方法. 

```
    /**
     * 创建过滤器, 实现不是GET请求参数必须填写AccessToken
     */
    public class AccessFilter extends ZuulFilter {
    
        private static Logger log = LoggerFactory.getLogger(AccessFilter.class);
    
        private static final String GET = "GET";
    
        /**
         * 过滤器的类型
         * 决定过滤器在请求那个生命周期执行
         * pre: 请求在路由之前调用
         * routing: 路由请求时调用
         * post: 在routing和error过滤器之后被调用
         * error: 处理请求发生错误时调用
         * @return
         */
        @Override
        public String filterType() {
            return "pre";
        }
    
        /**
         * 过滤器执行顺序
         * @return
         */
        @Override
        public int filterOrder() {
            return 0;
        }
    
        /**
         * 过滤器是否要被执行
         * @return
         */
        @Override
        public boolean shouldFilter() {
            return true;
        }
    
        /**
         * 过滤器的具体逻辑
         * @return
         */
        @Override
        public Object run() {
            RequestContext ctx = RequestContext.getCurrentContext();
            HttpServletRequest request = ctx.getRequest();
    
            log.info(String.format("%s request to %s", request.getMethod(), request.getRequestURL().toString()));
    
            Object accessToken = request.getParameter("accessToken");
    
            //不是GET请求必须有accessToken
            if(accessToken == null && !GET.equals(request.getMethod())) {
                log.warn("access token is empty");
                ctx.setSendZuulResponse(false);
                ctx.setResponseStatusCode(401);
                return null;
            }
            log.info("access token ok");
            return null;
        }
    }
```
- ApiGatewayApplication：启动类
```
    //SpringCloud应用启动(Boot, Eureka, Hystrix启动)
    @SpringCloudApplication
    //开启Zuul API网关服务
    @EnableZuulProxy
    public class ApiGatewayApplication {
    
        public static void main(String[] args) {
            SpringApplication.run(ApiGatewayApplication.class, args);
        }
    
        /**
         * 配置Zuul过滤器
         * @return
         */
        @Bean
        public AccessFilter accessFilter () {
            return new AccessFilter();
        }
    }
```



**7.启动**

- 启动EurekaServerApplication-注册中心
- 启动ComputeServiceApplication-发布服务
- 启动FeignApplication-服务消费
- 启动ApiGatewayApplication-服务网关

**8.测试**

- /json目录PostMan Collection请求脚本-导入后测试

**9.结束**