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
	    * [1、搭建服务治理注册中心](#1搭建服务治理注册中心)		
	* [五、容错保护Hystrix](#五容错保护Hystrix)
	* [六、API服务网关Zuul](#六API服务网关Zuul)
	* [七、统一配置中心Config](#七统一配置中心Config)
	* [八、分布式服务跟踪Sleuth](#八分布式服务跟踪Sleuth)
	* [九、消息驱动Stream](#九消息驱动Stream)
	* [十、应用安全Security](#十应用安全Security)
	* [十一、容器Docker与Jenkins](#十一容器Docker与Jenkins)
	
### 一、架构设计
### 1、拆分原则
- 
### 2、自治原则
- 
### 3、交互原则
- 
### 4、架构迁移
- 
### 5、微服务优缺点
- 
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
### 1、搭建服务治理注册中心
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
![图片](/images/4.jpg)