微服务架构
=========
## 说明
     学习笔记，持续更新中
## 目录
* [微服务架构](#微服务架构)
	* [一、架构设计](#一架构设计)
		* [1、拆分原则](#1拆分原则)
		* [2、自治原则](#2自治原则)
		* [3、交互原则](#3交互原则)
		* [4、架构迁移](#4架构迁移)
		* [5、微服务优缺点](#5微服务优缺点)
	* [二、SpringBoot](#二SpringBoot)
		* [1、SpringBoot整合swagger2](#1SpringBoot整合swagger2)		
	* [三、SpringCloud](#三SpringCloud)
	    * [1、Feign熔断降级](#1Feign熔断降级)		
	* [四、服务治理Eureka与负载均衡Ribbon](#四服务治理Eureka与负载均衡Ribbon)
	    * [1、搭建治理服务器-Eureka服务器](#1搭建治理服务器-Eureka服务器)	
	    * [2、搭建服务提供者-注册服务](#2搭建服务提供者-注册服务)
	    * [3、搭建服务消费者-获取服务](#2搭建服务消费者-获取服务)
	    * [4、客户端负载均衡Ribbon](#4客户端负载均衡Ribbon)	
	    * [5、Eureka高可用集群](#5Eureka高可用集群)			
	* [五、容错保护Hystrix](#五容错保护Hystrix)
	* [六、API服务网关Zuul](#六API服务网关Zuul)
	    * [1、搭建Zuul网关](#1搭建Zuul网关)	
	* [七、统一配置中心Config](#七统一配置中心Config)
	* [八、分布式服务跟踪Sleuth](#八分布式服务跟踪Sleuth)
	* [九、消息驱动Stream](#九消息驱动Stream)
	* [十、应用安全Security](#十应用安全Security)
	    * [1、OAuth认证](#1OAuth认证)	
	    * [2、JWT认证](#2JWT认证)	
	* [十一、容器Docker与Jenkins](#十一容器Docker与Jenkins)
	
### 一、架构设计
### 1、拆分原则
- 单一职责原则，一个微服务仅拥有一个业务
- 共同封闭原则，同一性质的类封装到同一个包中
### 2、自治原则
- 你构建，你运行。
### 3、交互原则
- 使用REST协议，JSON数据格式，HTTP标准状态码等
### 4、架构迁移
- 围绕传统应用开发出新的微服务应用，并逐渐替代传统应用中的部分业务功能
### 5、微服务优缺点
- 优点：松耦合，抽象，独立，应对用户需求多样性，有更高可用性和弹性
- 缺点：处理分布式较棘手，学习难度加大
### 二、SpringBoot
### 1、SpringBoot整合swagger2
- swagger是一个方便后端编写接口文档的开源项目，并提供界面化测试。
- maven依赖
```
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger-ui</artifactId>
    <version>${swagger.version}</version>
</dependency>
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger2</artifactId>
    <version>${swagger.version}</version>
</dependency>
```
- 配置类Swagger2Config
```
@Configuration
@EnableSwagger2
public class Swagger2Config {
     /**
     * 通过 createRestApi函数来构建一个DocketBean
     * 函数名,可以随意命名,喜欢什么命名就什么命名
     */
    @Bean
    public Docket createRestApi() {
        List<ApiListingReference> apiListingReferenceList = new ArrayList<>();
        apiListingReferenceList.add(new ApiListingReference("", "", 0));
        return new Docket(DocumentationType.SWAGGER_2)
                .apiInfo(apiInfo())//调用apiInfo方法,创建一个ApiInfo实例,里面是展示在文档页面信息内容
                .select()
                //控制暴露出去的路径下的实例
                //如果某个接口不想暴露,可以使用以下注解
                //@ApiIgnore 这样,该接口就不会暴露在 swagger2 的页面下
                .apis(RequestHandlerSelectors.basePackage("com.cd826dong"))
                .paths(PathSelectors.any())
                .build();
    }
    private ApiInfo apiInfo() {
        return new ApiInfoBuilder()
                .title("Spring Boot Swagger2 构建RESTful API")//页面标题
                .description("http://api.xwx.com/")
                .termsOfServiceUrl("https://xwx.com/")
                .contact("CD826")
                .version("1.0.0")
                .description("API 描述")//描述
                .build();
    }
}
```
- 新建api类
```
@RestController
@RequestMapping("/products")
@Api(value = "ProductEndpoint", description = "商品管理相关Api")
public class ProductEndpoint {
    @Autowired
    private ProductService productService;
    @Autowired
    private UserRepository userRepository;

    /**
     * 获取产品信息列表
     * @return
     */
    @RequestMapping(method = RequestMethod.GET)
    @ApiOperation(value = "获取商品分页数据", notes = "获取商品分页数据", httpMethod = "GET", tags = "商品管理相关Api")
    @ApiImplicitParams({
            @ApiImplicitParam(name = "page", value = "第几页，从0开始，默认为第0页", dataType = "int", paramType = "query"),
            @ApiImplicitParam(name = "size", value = "每一页记录数的大小，默认为20", dataType = "int", paramType = "query"),
            @ApiImplicitParam(name = "sort", value = "排序，格式为:property,property(,ASC|DESC)的方式组织，如sort=firstname&sort=lastname,desc", dataType = "String", paramType = "query")
    })
    public List<ProductDto> list(Pageable pageable) {
        Page<Product> page = this.productService.getPage(pageable);
        if (null != page) {
            return page.getContent().stream().map((product) -> {
                return new ProductDto(product);
            }).collect(Collectors.toList());
        }
        return Collections.EMPTY_LIST;
    }
```
- 启动项目,访问 http://localhost:8080/swagger-ui.html 显示如下图效果
![图片](/images/02.jpg)
- 访api接口 http://localhost:8080/products 显示如下图效果
![图片](/images/2-2.jpg)
### 三、SpringCloud
### 1、Feign熔断降级
- 只需要在服务消费者的application.properties配置文件中开启Hystrix功能
```
#指定服务名称
spring.application.name=hc-service
eureka.instance.prefer-ip-address=true
eureka.client.register-with-eureka=true
eureka.client.fetch-registry=true
#指定服务注册中心地址,多个注册中心以，隔开
eureka.client.service-url.defaultZone=http://localhost:8260/eureka
#开启Hystrix功能
feign.hystrix.enabled=true
```
- 为Feign客户端的定义接口编写一个具体的接口实现类，比如为HelloService接口实现一个服务降级类HelloServicFallback，其中每个重写方法的实现逻辑都可以用来定义相应的服务降级逻辑
```
/**
 * Hello服务降级实现
 */
@Component
public class HelloServiceFallback implements HelloService {
    public String hello(String name) {
        return "Hello, " + name + ", 熔断降级成功!";
    }
}
```
- 服务绑定接口HelloService中，通过@FeignClient注解的failback属性来指定对应的服务降级实现类。
```
/**
 * 远程Hello服务客户端
 */
@FeignClient(value = "HP-SERVICE", fallback = HelloServiceFallback.class)
public interface HelloService {
    @RequestMapping(value = "/hello/{name}", method = RequestMethod.GET)
    String hello(@PathVariable("name") String name);
}
```
- 测试验证，启动服务治理服务器，启动Hello服务消费者，不启动Hello服务提供者。访问 http://localhost:8080/hello/你好 显示如下图效果，则证明服务降级成功：
![图片](/images/3.jpg)
### 四、服务治理Eureka与负载均衡Ribbon
### 1、1搭建治理服务器-Eureka服务器
- maven添加Eureka服务器端依赖
```
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-eureka-server</artifactId>
    </dependency>
</dependencies>
```
- 在Springboot项目中的main入口，添加@EnableEurekaServer注解，来开启服务注册中心
```
@EnableEurekaServer
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        new SpringApplicationBuilder(Application.class).web(true).run(args);
    }
}
```
- 在默认情况下，服务注册中心也会把自己当做是一个服务，将自己注册进服务注册中心，所以我们可以通过配置来禁用他的客户端注册行为，在application.properties中添加如下配置：
```
#不要向注册中心注册自己
eureka.client.register-with-eureka=false
#禁止检索服务
eureka.client.fetch-registry=false
```
- 启动应用，并访问http://localhost:8026/即可看到Eureka信息面板，如下：
![图片](/images/4.jpg)<br>
在上图"Instances currently registered with Eureka"信息中，没有一个实例，说明目前还没有服务注册。
### 2、搭建服务提供者-注册服务
- maven添加Eureka客户端依赖（跟上面的服务端依赖不一样）
```
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-eureka</artifactId>
    </dependency>
</dependencies>
```
- 在主类上添加@EnableEurekaClient注解以实现Eureka中的DiscoveryClient实现
```
@EnableDiscoveryClient
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```
在application.properties中添加配置与服务相关的一些基本信息，如服务名、注册中心地址(即上一节中设置的注册中心地址http://localhost:8260/eureka)
```
#  设置服务名
spring.application.name=userservice
#  设置注册中心地址
eureka.client.service-url.defaultZone=http://localhost:8260/eureka
```
- 启动注册中心服务，可以看到如下的信息，说明此服务已经注册在了注册中心
![图片](/images/4-2.jpg)<br>
访问上面的服务治理注册中心http://localhost:8260/可以在Instances currently registered with Eureka中看到已经有一个服务注册了进来，并且名称为USERSERVICE的服务
![图片](/images/4-3.jpg)
### 3、搭建服务消费者-获取服务
- 跟上面的注册服务搭建步骤差不多，源码请参考demo/chapter04 -- eureka/product-service，启动成功之后，
访问服务治理注册中心http://localhost:8260/可以在Instances currently registered with Eureka中看到有服务注册进来，名称为PRODUCTSERVICE
![图片](/images/4-4.jpg)
### 4、客户端负载均衡Ribbon
- maven添加ribbon依赖
```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-ribbon</artifactId>
</dependency>
```
- 在启动类中向Spring容器中注入一个带有@LoadBalanced注解的RestTemplate Bean
```
@EnableDiscoveryClient
@EnableFeignClients
@SpringBootApplication
public class Application {
    @Bean(value = "restTemplate")
    @LoadBalanced
    RestTemplate restTemplate() {
        return new RestTemplate();
    }
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```
- 在调用那些需要做负载均衡的服务时，使用上面注入的RestTemplate Bean进行调用即可
```
UserDto userDto = this.restTemplate.getForEntity("http://USERSERVICE/users/{id}", UserDto.class, userId).getBody();
```
- 用三个端口分别启动服务提供者，访问服务消费者，刷新几次会看到，调用的端口不一样
### 5、Eureka高可用集群
- maven添加Eureka服务器端依赖
```
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-eureka-server</artifactId>
    </dependency>
</dependencies>
```
- 在Springboot项目中的main入口，添加@EnableEurekaServer注解，来开启服务注册中心
```
@EnableEurekaServer
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        new SpringApplicationBuilder(Application.class).web(true).run(args);
    }
}
```
- 在项目中创建3个配置文件application-node1.properties，application-node2.properties和application-node3.properties分别作为node1，node2和node3节点的配置文件。配置服务的端口和hostname，指定其他两个eureka server，地址中的hostname应与配置的hostname相对应。
如application-node1.properties
```
server.port=8761
eureka.instance.hostname=node1
eureka.client.service-url.defaultZone=http://node2:8762/eureka/,http://node3:8763/eureka
```
- 启动应用，并访问http://localhost:8761/即可看到Eureka信息面板，如下：
![图片](/images/4-7.jpg)<br>
在node1的页面上可以看到，General Info的available-replicas有node2和node3节点，说明3个节点的eureka server相互注册成功。<br>
如果available-replicas是空的，而unavailable-replicas有其他两个节点，说明配置有问题，集群搭建失败。
### 五、容错保护Hystrix
通过hystrix可以解决雪崩效应问题，它提供了资源隔离、降级机制、融断、缓存等功能。
- 资源隔离：包括线程池隔离和信号量隔离，限制调用分布式服务的资源使用，某一个调用的服务出现问题不会影响其他服务调用。
- 降级机制：超时降级、资源不足时(线程或信号量)降级，降级后可以配合降级接口返回托底数据。
- 融断：当失败率达到阀值自动触发降级(如因网络故障/超时造成的失败率高)，熔断器触发的快速失败会进行快速恢复。
- 缓存：返回结果缓存，后续请求可以直接走缓存。
- 请求合并：可以实现将一段时间内的请求（一般是对同一个接口的请求）合并，然后只对服务提供者发送一次请求。
- 如上面讲的Feign熔断降级
### 六、API服务网关Zuul
- Zuul提供了动态路由、监控、弹性负载和安全功能。Zuul底层利用各种filter可实现如下功能：
认证和安全 识别每个需要认证的资源，拒绝不符合要求的请求。<br>
性能监测 在服务边界追踪并统计数据，提供精确的生产视图。<br>
动态路由 根据需要将请求动态路由到后端集群。<br>
压力测试 逐渐增加对集群的流量以了解其性能。<br>
负载卸载 预先为每种类型的请求分配容量，当请求超过容量时自动丢弃。<br>
静态资源处理 直接在边界返回某些响应。<br>
### 1、搭建Zuul网关
- maven中引入eureka客户端
```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-zuul</artifactId>
</dependency>
```
- 在启动类中，增加@EnableZuulProxy注解
```
@EnableZuulProxy
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```
- 在application.properties中添加配置如服务名、注册中心地址等
```
#  设置注册中心地址
eureka.client.service-url.defaultZone=http://localhost:8260/eureka
```
- 按顺序先启动治理服务器、服务提供者、服务消费者、网关服务器<br>
访问服务治理注册中心http://localhost:8260/可以在Instances currently registered with Eureka中看到如下图
![图片](/images/4-5.jpg)
### 十、应用安全Security
- 常用安全认证有SSO，分布式会话Session，客户端令牌Token，客户端令牌与API网关结合
### 1、OAuth认证
- maven中引入
```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-security</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.security.oauth</groupId>
    <artifactId>spring-security-oauth2</artifactId>
</dependency>
```
- 在启动类中，增加@EnableAuthorizationServer注解
```
@SpringBootApplication
@RestController
@EnableResourceServer
@EnableAuthorizationServer
public class Application {
    @RequestMapping(value = { "/user" }, produces = "application/json")
    public Map<String, Object> user(OAuth2Authentication user) {
        Map<String, Object> userInfo = new HashMap<>();
        userInfo.put("user", user.getUserAuthentication().getPrincipal());
        userInfo.put("authorities", AuthorityUtils.authorityListToSet( user.getUserAuthentication().getAuthorities()));
        return userInfo;
    }
    public static void main(String[] args) {
        new SpringApplicationBuilder(Application.class).web(true).run(args);
    }
}
```
- 扩展OAuthWebSecurityConfigurer并实现configure
```
@Configuration
@Order(org.springframework.boot.autoconfigure.security.SecurityProperties.ACCESS_OVERRIDE_ORDER)
public class OAuthWebSecurityConfigurer extends WebSecurityConfigurerAdapter {
    @Override
    @Bean
    public AuthenticationManager authenticationManagerBean() throws Exception{
        return super.authenticationManagerBean();
    }
    @Override
    @Bean
    public UserDetailsService userDetailsServiceBean() throws Exception {
        return super.userDetailsServiceBean();
    }
    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.inMemoryAuthentication()
                .withUser("zhangsan")
                .password("password")
                .roles("USER")
                .and()
                .withUser("superAdmin")
                .password("superPwd")
                .roles("USER", "ADMIN");
    }
}
```
### 2、JWT认证
- maven中引入
```
<dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-jwt</artifactId>
</dependency>
```
- 定义令牌转换器，具体实现参考源码