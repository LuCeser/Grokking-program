# spring-test构建Java测试体系(1)

对于使用Spring MVC构建的Java Web应用，Controller是属于最前面一层的，本文将介绍如何针对Controller编写单元测试。

# 1. Maven依赖信息
```xml
    <parent>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-parent</artifactId>
            <version>2.1.1.RELEASE</version>
            <relativePath/> <!-- lookup parent from repository -->
    </parent>
    
    <dependencies>
    
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-web</artifactId>
            </dependency>
    
            <!-- test -->
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-test</artifactId>
                <scope>test</scope>
            </dependency>
            <!-- test end -->
    
    </dependencies>
```
# 2. 定义一个REST接口

定义一个实体类`Demo`
```java
    public class Demo {
        private String name;
    		// setter, getter...
    }
```
新建一个Controller类，定义一个查询方法。
URL为`GET demos`，调用成功后将返回一个json数组，http返回码为200 OK。
```java
    @Controller
    @RequestMapping("demos")
    public class DemoController {
    
        @GetMapping
        public ResponseEntity<List> searchDemo() {
            return new ResponseEntity<>(new ArrayList<Demo>(), HttpStatus.OK);
        }
    }
```
# 3. 编写测试用例

对于单元测试来说只需要关注Controller层，而不需要加载整个Spring上下文。

```java
    // 告诉junit使用MockitoJUnitRunner来运行测试用例
    // 这样就可以使用@Mock和@InjectMocks注解
    @RunWith(MockitoJUnitRunner.class)
    public class MockDemoControllerTest {
    
        private MockMvc mockMvc;
    
        @InjectMocks
        private DemoController demoController; // 创建demoController
    
        @Before
        public void setUp() throws Exception {
    				// 构造mockMvc，指定需要测试的Controller对象
            mockMvc = MockMvcBuilders.standaloneSetup(demoController).build();
        }
    
        @Test
        public void should_get_demos() throws Exception {
    				// 调用此接口并断言返回 200 OK
            mockMvc.perform(get("/demos")).andDo(print())
                    .andExpect(status().isOk());
        }
    
    }
```

# 4. 总结

在本篇小文中介绍了如何针对Spring Boot编写的REST接口进行测试，用到了`spring-tes`提供的`MockMvc`实现对HTTP请求的模拟。除此之外，测试中还利用`MockMvc`提供的验证工具对结果进行断言。

本文只能算是一个开头，示例项目中并没有调用任何业务逻辑，我将在下一篇中讲述如何mock依赖关系。

[SpringBoot基础之MockMvc单元测试](https://zhuanlan.zhihu.com/p/61342833)
