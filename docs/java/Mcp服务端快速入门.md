## 一、依赖导入

``` xml
    //SSE基于 Spring MVC 的（服务器发送事件）服务器传输和可选STDIO传输
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-starter-mcp-server-webmvc</artifactId>
        <version>1.0.0</version>
    </dependency>

    //SSE基于 Spring WebFlux 和可选传输的（服务器发送事件）服务器传输STDIO。
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-starter-mcp-server-webflux</artifactId>
        <version>1.0.0</version>
    </dependency>

```
## 二、配置

示例:
```yml
spring:
  ai:
    mcp:
      server:
        enabled: true
        name: webmvc-mcp-server
        version: 1.0.0
        #同步
        type: SYNC
        instructions: "This server provides weather information tools and resources"
        capabilities:
          tool: true
          resource: false
          prompt: false
          completion: false
```

完整：

![MCP Server配置](/img/mcp服务器配置.png)

## 三、使用

1. 定义工具

    **@Tool**: 定义工具信息,包括工具的名称、描述等

    **@ToolParam**：定义工具请求参数，方便自动生成schema定义。注意，不会对请求时是否携带此参数进行校验，需要自定义校验


    ```java
    @Service
    public class McpToolService {

        @Tool(description = "获取系统时间")
        public AjaxResult<Long> geTime(){
        return AjaxResult.success(DateUtil.current());
        }

        @Tool(description = "获取实时天气")
        public AjaxResult<String> geWeather(@ToolParam(description = "城市名称") String city){
        return AjaxResult.success("晴转多云");
        }

    }
    ```
2. 注册

    ```java
    @Component
    public class McpTool {

        @Bean
        public ToolCallbackProvider weatherTools(McpToolService mcpToolService) {
            return MethodToolCallbackProvider.builder().toolObjects(mcpToolService).build();
        }
    }
    ```
    
3. 客户端测试

    ```java
    public class TestMcp {
        public static void main(String[] args) {
            HttpClientSseClientTransport.Builder builder = HttpClientSseClientTransport
                    .builder("http://192.168.31.154:9601").sseEndpoint("/sse");
            McpClientTransport transport = builder.build();
            //连接mcp服务端
            McpSyncClient mcpSyncClient = McpClient.sync(transport).requestTimeout(Duration.ofSeconds(20)).build();
            //初始化客户端
            mcpSyncClient.initialize();
            //查询工具列表
            McpSchema.ListToolsResult listToolsResult = mcpSyncClient.listTools();
            listToolsResult.tools().forEach(tool -> {
                System.out.println(tool.name());
                if(tool.name().equals("geWeather")){
                    Map<String, Object> arguments = new HashMap<>();
                    arguments.put("city", "上海");
                    McpSchema.CallToolRequest callToolRequest = new McpSchema.CallToolRequest(tool.name(), arguments);
                    //工具调用
                    McpSchema.CallToolResult callToolResult = mcpSyncClient.callTool(callToolRequest);
                    System.out.println(callToolResult.content());
                }

            });
        }
    }

    ```