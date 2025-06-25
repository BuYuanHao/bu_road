# Java测试框架技术选型指南

## 目录
1. [测试框架分类](#测试框架分类)
2. [单元测试框架](#单元测试框架)
3. [Mock测试框架](#Mock测试框架)
4. [集成测试框架](#集成测试框架)
5. [断言库](#断言库)
6. [数据库测试框架](#数据库测试框架)
7. [Web测试框架](#Web测试框架)
8. [性能测试框架](#性能测试框架)
9. [测试工具生态](#测试工具生态)
10. [框架选型建议](#框架选型建议)

## 测试框架分类

### 按测试层次分类
```
┌─────────────────┐
│   E2E测试框架    │  Selenium, Cypress, Playwright
├─────────────────┤
│   集成测试框架   │  Spring Test, TestContainers, WireMock
├─────────────────┤
│   单元测试框架   │  JUnit, TestNG, Spock
└─────────────────┘
```

### 按功能分类
- **测试运行器**：JUnit, TestNG
- **断言库**：AssertJ, Hamcrest, Truth
- **Mock框架**：Mockito, PowerMock, EasyMock
- **测试数据**：Faker, JFixture, ObjectMother
- **性能测试**：JMH, Gatling, JMeter

## 单元测试框架

### JUnit 5 (Jupiter)

#### 特性对比
| 特性 | JUnit 4 | JUnit 5 | 说明 |
|------|---------|---------|------|
| 架构 | 单一JAR | 模块化 | JUnit 5分为Platform、Jupiter、Vintage |
| Java版本 | Java 5+ | Java 8+ | 支持Lambda表达式 |
| 注解 | @Test | @Test, @DisplayName | 更丰富的注解 |
| 断言 | 基础断言 | 增强断言 | assertAll, assertThrows等 |
| 参数化 | @Parameterized | @ParameterizedTest | 更灵活的参数化 |

#### 核心模块
```xml
<!-- JUnit 5 核心依赖 -->
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter</artifactId>
    <version>5.9.2</version>
    <scope>test</scope>
</dependency>
```

#### 高级特性
```java
// 动态测试
@TestFactory
Stream<DynamicTest> dynamicTests() {
    return Stream.of("apple", "banana", "orange")
        .map(fruit -> DynamicTest.dynamicTest(
            "Test " + fruit,
            () -> assertTrue(fruit.length() > 0)
        ));
}

// 条件测试
@Test
@EnabledOnOs(OS.LINUX)
void onlyOnLinux() {
    // 只在Linux上运行
}

@Test
@EnabledIfSystemProperty(named = "os.arch", matches = ".*64.*")
void onlyOn64BitArchitectures() {
    // 只在64位架构上运行
}

// 嵌套测试
@Nested
@DisplayName("When user is authenticated")
class WhenAuthenticated {
    @Test
    @DisplayName("Should allow access")
    void shouldAllowAccess() {
        // 测试代码
    }
}
```

### TestNG

#### 核心特性
```xml
<dependency>
    <groupId>org.testng</groupId>
    <artifactId>testng</artifactId>
    <version>7.7.1</version>
    <scope>test</scope>
</dependency>
```

#### 独特优势
```java
public class TestNGAdvancedFeatures {
    
    // 测试依赖
    @Test
    public void serverStartedOk() { }
    
    @Test(dependsOnMethods = {"serverStartedOk"})
    public void method1() { }
    
    // 测试分组
    @Test(groups = {"unit", "fast"})
    public void fastUnitTest() { }
    
    @Test(groups = {"integration", "slow"})
    public void slowIntegrationTest() { }
    
    // 并行执行配置
    @Test(threadPoolSize = 3, invocationCount = 10)
    public void parallelTest() { }
    
    // 数据驱动测试
    @DataProvider(name = "test-data")
    public Object[][] dataProviderMethod() {
        return new Object[][]{
            {"data1", 1},
            {"data2", 2}
        };
    }
    
    @Test(dataProvider = "test-data")
    public void parameterizedTest(String data, int number) { }
}
```

### Spock Framework

#### Groovy驱动的BDD测试
```xml
<dependency>
    <groupId>org.spockframework</groupId>
    <artifactId>spock-core</artifactId>
    <version>2.3-groovy-4.0</version>
    <scope>test</scope>
</dependency>
```

#### 特色语法
```groovy
class CalculatorSpec extends Specification {
    
    def "addition of two numbers"() {
        given: "a calculator"
        def calculator = new Calculator()
        
        when: "adding two numbers"
        def result = calculator.add(a, b)
        
        then: "the result should be correct"
        result == expected
        
        where: "test data"
        a | b | expected
        1 | 2 | 3
        3 | 4 | 7
        5 | 6 | 11
    }
    
    def "should handle exceptions"() {
        given:
        def calculator = new Calculator()
        
        when:
        calculator.divide(10, 0)
        
        then:
        thrown(ArithmeticException)
    }
}
```

## Mock测试框架

### Mockito

#### 核心特性
```xml
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-core</artifactId>
    <version>5.1.1</version>
    <scope>test</scope>
</dependency>
```

#### 基础用法
```java
@ExtendWith(MockitoExtension.class)
class ServiceTest {
    
    @Mock
    private UserRepository repository;
    
    @InjectMocks
    private UserService service;
    
    @Test
    void testUserCreation() {
        // Given
        User user = new User("test@example.com");
        when(repository.save(any(User.class)))
            .thenReturn(user);
        
        // When
        User result = service.createUser(user);
        
        // Then
        verify(repository).save(user);
        assertEquals("test@example.com", result.getEmail());
    }
}
```

#### 高级功能
```java
@Test
void advancedMockingFeatures() {
    List<String> mockedList = mock(List.class);
    
    // 行为验证
    doThrow(new RuntimeException()).when(mockedList).clear();
    
    // 参数捕获
    ArgumentCaptor<String> captor = ArgumentCaptor.forClass(String.class);
    mockedList.add("someValue");
    verify(mockedList).add(captor.capture());
    
    // 自定义Answer
    when(mockedList.get(anyInt())).thenAnswer(invocation -> {
        Integer index = invocation.getArgument(0);
        return "element-" + index;
    });
    
    // Mock静态方法 (Mockito 3.4+)
    try (MockedStatic<StaticUtils> mockedStatic = mockStatic(StaticUtils.class)) {
        mockedStatic.when(() -> StaticUtils.staticMethod("test"))
                   .thenReturn("mocked");
    }
}
```

### PowerMock (已不推荐)

#### 遗留项目支持
```xml
<dependency>
    <groupId>org.powermock</groupId>
    <artifactId>powermock-module-junit4</artifactId>
    <version>2.0.9</version>
    <scope>test</scope>
</dependency>
```

### EasyMock

#### 录制-回放模式
```java
@Test
public void testWithEasyMock() {
    // 创建Mock
    UserService mockService = EasyMock.createMock(UserService.class);
    
    // 录制期望行为
    expect(mockService.getUser(1L)).andReturn(new User("test"));
    replay(mockService);
    
    // 执行测试
    User user = mockService.getUser(1L);
    
    // 验证
    verify(mockService);
    assertEquals("test", user.getName());
}
```

## 集成测试框架

### Spring Test

#### 核心注解
```java
// Web层测试
@WebMvcTest(UserController.class)
class UserControllerTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @MockBean
    private UserService userService;
    
    @Test
    void testGetUser() throws Exception {
        when(userService.getUser(1L))
            .thenReturn(new User("test"));
        
        mockMvc.perform(get("/users/1"))
               .andExpect(status().isOk())
               .andExpect(jsonPath("$.name", is("test")));
    }
}

// 数据层测试
@DataJpaTest
class UserRepositoryTest {
    
    @Autowired
    private TestEntityManager entityManager;
    
    @Autowired
    private UserRepository repository;
    
    @Test
    void testFindByEmail() {
        User user = new User("test@example.com");
        entityManager.persistAndFlush(user);
        
        Optional<User> found = repository.findByEmail("test@example.com");
        
        assertTrue(found.isPresent());
    }
}

// 完整集成测试
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class FullIntegrationTest {
    
    @Autowired
    private TestRestTemplate restTemplate;
    
    @Test
    void testCompleteWorkflow() {
        ResponseEntity<String> response = restTemplate
            .postForEntity("/api/users", new User("test"), String.class);
        
        assertEquals(HttpStatus.CREATED, response.getStatusCode());
    }
}
```

### TestContainers

#### 容器化测试环境
```xml
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>junit-jupiter</artifactId>
    <version>1.17.6</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>postgresql</artifactId>
    <version>1.17.6</version>
    <scope>test</scope>
</dependency>
```

#### 数据库集成测试
```java
@Testcontainers
class DatabaseIntegrationTest {
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:13")
            .withDatabaseName("testdb")
            .withUsername("test")
            .withPassword("test");
    
    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
    
    @Test
    void testDatabaseOperation() {
        // 使用真实的PostgreSQL容器
        assertTrue(postgres.isRunning());
    }
}
```

#### 多容器编排
```java
@Testcontainers
class MicroserviceIntegrationTest {
    
    @Container
    static Network network = Network.newNetwork();
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:13")
            .withNetwork(network)
            .withNetworkAliases("postgres");
    
    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:6-alpine")
            .withNetwork(network)
            .withNetworkAliases("redis")
            .withExposedPorts(6379);
    
    @Container
    static GenericContainer<?> app = new GenericContainer<>("myapp:latest")
            .withNetwork(network)
            .withEnv("DB_HOST", "postgres")
            .withEnv("REDIS_HOST", "redis")
            .dependsOn(postgres, redis);
}
```

### WireMock

#### HTTP服务Mock
```xml
<dependency>
    <groupId>com.github.tomakehurst</groupId>
    <artifactId>wiremock-jre8</artifactId>
    <version>2.35.0</version>
    <scope>test</scope>
</dependency>
```

```java
@Test
void testExternalServiceCall() {
    // 启动WireMock服务器
    WireMockServer wireMockServer = new WireMockServer(8089);
    wireMockServer.start();
    
    // 配置Mock响应
    wireMockServer.stubFor(get(urlEqualTo("/api/users/1"))
        .willReturn(aResponse()
            .withStatus(200)
            .withHeader("Content-Type", "application/json")
            .withBody("{\"id\":1,\"name\":\"Test User\"}")));
    
    // 执行测试
    String response = restTemplate.getForObject(
        "http://localhost:8089/api/users/1", String.class);
    
    assertThat(response).contains("Test User");
    
    // 验证请求
    wireMockServer.verify(getRequestedFor(urlEqualTo("/api/users/1")));
    
    wireMockServer.stop();
}
```

## 断言库

### AssertJ

#### 流畅断言API
```xml
<dependency>
    <groupId>org.assertj</groupId>
    <artifactId>assertj-core</artifactId>
    <version>3.24.2</version>
    <scope>test</scope>
</dependency>
```

```java
@Test
void testWithAssertJ() {
    List<String> friends = Arrays.asList("Alice", "Bob", "Charlie");
    
    // 集合断言
    assertThat(friends)
        .hasSize(3)
        .contains("Alice", "Bob")
        .doesNotContain("David")
        .startsWith("Alice");
    
    // 字符串断言
    assertThat("Hello World")
        .isNotEmpty()
        .startsWith("Hello")
        .endsWith("World")
        .contains("o W");
    
    // 对象断言
    Person person = new Person("John", 30);
    assertThat(person)
        .hasFieldOrPropertyWithValue("name", "John")
        .hasFieldOrPropertyWithValue("age", 30);
    
    // 异常断言
    assertThatThrownBy(() -> {
        throw new IllegalArgumentException("Invalid argument");
    })
    .isInstanceOf(IllegalArgumentException.class)
    .hasMessage("Invalid argument");
}
```

### Hamcrest

#### 匹配器模式
```xml
<dependency>
    <groupId>org.hamcrest</groupId>
    <artifactId>hamcrest</artifactId>
    <version>2.2</version>
    <scope>test</scope>
</dependency>
```

```java
@Test
void testWithHamcrest() {
    List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5);
    
    assertThat(numbers, hasSize(5));
    assertThat(numbers, hasItems(1, 3, 5));
    assertThat(numbers, everyItem(greaterThan(0)));
    
    // 自定义匹配器
    assertThat("myStringOfNote", either(containsString("my"))
                                 .or(containsString("Note")));
}
```

### Google Truth

#### Google风格断言
```xml
<dependency>
    <groupId>com.google.truth</groupId>
    <artifactId>truth</artifactId>
    <version>1.1.3</version>
    <scope>test</scope>
</dependency>
```

```java
@Test
void testWithTruth() {
    List<String> colors = Arrays.asList("red", "green", "blue");
    
    assertThat(colors).hasSize(3);
    assertThat(colors).contains("red");
    assertThat(colors).containsExactly("red", "green", "blue").inOrder();
    
    Map<String, Integer> map = Map.of("a", 1, "b", 2);
    assertThat(map).containsEntry("a", 1);
    assertThat(map).containsExactly("a", 1, "b", 2);
}
```

## 数据库测试框架

### H2 内存数据库

#### 快速测试配置
```properties
# application-test.properties
spring.datasource.url=jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE
spring.datasource.driverClassName=org.h2.Driver
spring.jpa.hibernate.ddl-auto=create-drop
```

### DbUnit

#### 数据库状态管理
```xml
<dependency>
    <groupId>org.dbunit</groupId>
    <artifactId>dbunit</artifactId>
    <version>2.7.3</version>
    <scope>test</scope>
</dependency>
```

```java
@Test
void testWithDbUnit() throws Exception {
    // 准备测试数据
    IDataSet dataSet = new FlatXmlDataSetBuilder()
        .build(new FileInputStream("test-data.xml"));
    
    DatabaseTester databaseTester = new JdbcDatabaseTester(
        "org.h2.Driver", "jdbc:h2:mem:test", "sa", "");
    
    databaseTester.setDataSet(dataSet);
    databaseTester.onSetup();
    
    // 执行测试
    // ...
    
    databaseTester.onTearDown();
}
```

## Web测试框架

### Selenium WebDriver

#### 浏览器自动化测试
```xml
<dependency>
    <groupId>org.seleniumhq.selenium</groupId>
    <artifactId>selenium-java</artifactId>
    <version>4.8.1</version>
    <scope>test</scope>
</dependency>
```

```java
@Test
void testWebInterface() {
    WebDriver driver = new ChromeDriver();
    
    try {
        driver.get("http://localhost:8080/login");
        
        WebElement username = driver.findElement(By.name("username"));
        WebElement password = driver.findElement(By.name("password"));
        WebElement loginButton = driver.findElement(By.id("login-btn"));
        
        username.sendKeys("testuser");
        password.sendKeys("password");
        loginButton.click();
        
        WebDriverWait wait = new WebDriverWait(driver, Duration.ofSeconds(10));
        WebElement dashboard = wait.until(
            ExpectedConditions.presenceOfElementLocated(By.id("dashboard"))
        );
        
        assertTrue(dashboard.isDisplayed());
    } finally {
        driver.quit();
    }
}
```

### REST Assured

#### REST API测试
```xml
<dependency>
    <groupId>io.rest-assured</groupId>
    <artifactId>rest-assured</artifactId>
    <version>5.3.0</version>
    <scope>test</scope>
</dependency>
```

```java
@Test
void testRestAPI() {
    given()
        .contentType(ContentType.JSON)
        .body("{\"name\":\"John\",\"email\":\"john@example.com\"}")
    .when()
        .post("/api/users")
    .then()
        .statusCode(201)
        .body("name", equalTo("John"))
        .body("email", equalTo("john@example.com"))
        .body("id", notNullValue());
}

@Test
void testGetUser() {
    given()
        .pathParam("id", 1)
    .when()
        .get("/api/users/{id}")
    .then()
        .statusCode(200)
        .body("name", not(emptyString()))
        .time(lessThan(2000L)); // 响应时间少于2秒
}
```

## 性能测试框架

### JMH (Java Microbenchmark Harness)

#### 微基准测试
```xml
<dependency>
    <groupId>org.openjdk.jmh</groupId>
    <artifactId>jmh-core</artifactId>
    <version>1.36</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.openjdk.jmh</groupId>
    <artifactId>jmh-generator-annprocess</artifactId>
    <version>1.36</version>
    <scope>test</scope>
</dependency>
```

```java
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@State(Scope.Benchmark)
@Fork(value = 2, jvmArgs = {"-Xms2G", "-Xmx2G"})
public class StringConcatenationBenchmark {
    
    @Param({"100", "1000", "10000"})
    private int size;
    
    private String[] strings;
    
    @Setup
    public void setup() {
        strings = new String[size];
        for (int i = 0; i < size; i++) {
            strings[i] = "string" + i;
        }
    }
    
    @Benchmark
    public String stringConcat() {
        String result = "";
        for (String s : strings) {
            result += s;
        }
        return result;
    }
    
    @Benchmark
    public String stringBuilder() {
        StringBuilder sb = new StringBuilder();
        for (String s : strings) {
            sb.append(s);
        }
        return sb.toString();
    }
}
```

### Gatling

#### 负载测试
```xml
<dependency>
    <groupId>io.gatling.highcharts</groupId>
    <artifactId>gatling-charts-highcharts</artifactId>
    <version>3.9.0</version>
    <scope>test</scope>
</dependency>
```

```scala
import io.gatling.core.Predef._
import io.gatling.http.Predef._

class UserSimulation extends Simulation {
  
  val httpProtocol = http
    .baseUrl("http://localhost:8080")
    .acceptHeader("application/json")
  
  val scn = scenario("User API Test")
    .exec(http("Get Users")
      .get("/api/users")
      .check(status.is(200)))
    .pause(1)
    .exec(http("Create User")
      .post("/api/users")
      .body(StringBody("""{"name":"Test User","email":"test@example.com"}"""))
      .check(status.is(201)))
  
  setUp(
    scn.inject(
      rampUsers(100) during (30 seconds),
      constantUsersPerSec(20) during (1 minutes)
    )
  ).protocols(httpProtocol)
}
```

## 测试工具生态

### 构建工具集成

#### Maven Surefire插件
```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <version>3.0.0-M9</version>
    <configuration>
        <includes>
            <include>**/*Test.java</include>
            <include>**/*Tests.java</include>
        </includes>
        <excludes>
            <exclude>**/*IntegrationTest.java</exclude>
        </excludes>
        <parallel>methods</parallel>
        <threadCount>4</threadCount>
    </configuration>
</plugin>
```

#### Gradle Test任务
```groovy
test {
    useJUnitPlatform()
    
    testLogging {
        events "passed", "skipped", "failed"
        exceptionFormat "full"
    }
    
    systemProperty 'junit.jupiter.execution.parallel.enabled', 'true'
    systemProperty 'junit.jupiter.execution.parallel.mode.default', 'concurrent'
    
    reports {
        html.enabled = true
        junitXml.enabled = true
    }
}
```

### 代码覆盖率工具

#### JaCoCo配置
```xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.8</version>
    <executions>
        <execution>
            <goals>
                <goal>prepare-agent</goal>
            </goals>
        </execution>
        <execution>
            <id>report</id>
            <phase>test</phase>
            <goals>
                <goal>report</goal>
            </goals>
        </execution>
        <execution>
            <id>check</id>
            <goals>
                <goal>check</goal>
            </goals>
            <configuration>
                <rules>
                    <rule>
                        <element>BUNDLE</element>
                        <limits>
                            <limit>
                                <counter>LINE</counter>
                                <value>COVEREDRATIO</value>
                                <minimum>0.80</minimum>
                            </limit>
                        </limits>
                    </rule>
                </rules>
            </configuration>
        </execution>
    </executions>
</plugin>
```

### 测试报告工具

#### Allure Framework
```xml
<dependency>
    <groupId>io.qameta.allure</groupId>
    <artifactId>allure-junit5</artifactId>
    <version>2.20.1</version>
    <scope>test</scope>
</dependency>
```

```java
@Epic("User Management")
@Feature("User Registration")
public class UserRegistrationTest {
    
    @Test
    @Story("Successful user registration")
    @Description("Test successful user registration with valid data")
    @Severity(SeverityLevel.CRITICAL)
    void testSuccessfulRegistration() {
        step("Prepare user data", () -> {
            // 准备数据
        });
        
        step("Register user", () -> {
            // 注册用户
        });
        
        step("Verify registration", () -> {
            // 验证结果
        });
    }
    
    @Step("Execute step: {stepName}")
    private void step(String stepName, Runnable action) {
        action.run();
    }
}
```

## 框架选型建议

### 选型决策矩阵

| 项目特征 | 推荐框架组合 | 原因 |
|----------|-------------|------|
| 新项目，Spring Boot | JUnit 5 + Mockito + Spring Test + AssertJ | 生态完整，社区活跃 |
| 遗留项目，JUnit 4 | JUnit 4 + Mockito + Hamcrest | 兼容性好，迁移成本低 |
| 微服务架构 | JUnit 5 + TestContainers + WireMock | 容器化测试，服务隔离 |
| 性能敏感应用 | JUnit 5 + JMH + Gatling | 微基准+负载测试 |
| 大型团队项目 | TestNG + Mockito + Allure | 并行执行，丰富报告 |
| BDD需求 | Spock + Cucumber | 自然语言描述，业务可读 |

### 技术栈推荐

#### 标准企业级组合
```xml
<!-- 核心测试框架 -->
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter</artifactId>
</dependency>

<!-- Mock框架 -->
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-core</artifactId>
</dependency>

<!-- 断言库 -->
<dependency>
    <groupId>org.assertj</groupId>
    <artifactId>assertj-core</artifactId>
</dependency>

<!-- Spring测试支持 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
</dependency>

<!-- 容器化测试 -->
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>junit-jupiter</artifactId>
</dependency>

<!-- API测试 -->
<dependency>
    <groupId>io.rest-assured</groupId>
    <artifactId>rest-assured</artifactId>
</dependency>
```

#### 性能优化组合
```xml
<!-- 微基准测试 -->
<dependency>
    <groupId>org.openjdk.jmh</groupId>
    <artifactId>jmh-core</artifactId>
</dependency>

<!-- 并发测试 -->
<dependency>
    <groupId>org.awaitility</groupId>
    <artifactId>awaitility</artifactId>
</dependency>

<!-- 内存分析 -->
<dependency>
    <groupId>org.openjdk.jol</groupId>
    <artifactId>jol-core</artifactId>
</dependency>
```

### 迁移策略

#### JUnit 4 → JUnit 5
```java
// JUnit 4
@Before → @BeforeEach
@After → @AfterEach
@BeforeClass → @BeforeAll
@AfterClass → @AfterAll
@Test(expected = Exception.class) → assertThrows()
@RunWith → @ExtendWith

// 迁移工具
// junit-vintage-engine 提供向下兼容
```

#### TestNG → JUnit 5
```java
// 功能映射
@BeforeMethod → @BeforeEach
@AfterMethod → @AfterEach
@BeforeClass → @BeforeAll
@AfterClass → @AfterAll
@Test(groups) → @Tag
@DataProvider → @ParameterizedTest
```

### 最佳实践总结

1. **框架选择原则**
   - 团队熟悉度优先
   - 生态系统完整性
   - 长期维护活跃度
   - 性能和功能需求匹配

2. **组合使用建议**
   - 核心框架统一（JUnit 5或TestNG）
   - Mock框架标准化（Mockito）
   - 断言库现代化（AssertJ）
   - 集成测试容器化（TestContainers）

3. **持续改进**
   - 定期评估框架版本
   - 关注新兴测试技术
   - 优化测试执行性能
   - 完善测试报告体系

记住：**选择合适的测试框架组合，是构建高质量软件的重要基础！**