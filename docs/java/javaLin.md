


https://javalin.io/


```xml
<dependencies>
        <dependency>
            <groupId>io.javalin</groupId>
            <artifactId>javalin</artifactId>
            <version>6.7.0</version>
        </dependency>
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-simple</artifactId>
            <version>2.0.16</version>
        </dependency>
        <dependency>
            <groupId>cn.hutool</groupId>
            <artifactId>hutool-all</artifactId>
            <version>5.8.35</version>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
            <version>2.19.1</version>
        </dependency>
    </dependencies>
```




```java
package com.geshu;

import cn.hutool.json.JSONObject;
import cn.hutool.json.JSONUtil;
import io.javalin.Javalin;
import io.javalin.websocket.WsConnectContext;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.time.Duration;
import java.util.List;
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicReference;

/**
 * @Description: ${DESCRIPTION}
 * @Author: buyuanhao
 * @CreateDate: 2025/8/13 17:50
 */

public class Main {
    Logger logger = LoggerFactory.getLogger(Main.class);
    public static void main(String[] args) {
        
        Javalin.create().get("/hello", context -> {
            context.result("hello world");
            context.status(200);
        }).ws("/ws", context -> {

            context.onConnect(session -> {
                session.send("hello world");
                
            });
            context.onClose(session -> {
                map.remove(session.sessionId());
            });
            context.onMessage(session -> {
                session.send("hello world");
            });

        }).post("/add", context -> {
            User user = context.bodyAsClass(User.class);
            context.result(user.getName());
            context.status(200);
        }).start(8787);
    }
}
```