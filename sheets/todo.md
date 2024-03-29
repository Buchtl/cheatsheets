## Reminder 2024-01-15

* [see springdoc](https://docs.spring.io/spring-graphql/reference/transports.html)  and [git project](https://github.com/spring-projects/spring-graphql/tree/1.0.x/samples/webflux-websocket)



GraphQlWebSocketHandler can be exposed as a WebSocket endpoint by declaring a SimpleUrlHandlerMapping bean and using it to map the handler to a URL path. By default, the Boot Starter does not expose a GraphQL over WebSocket endpoint, but it’s easy to enable it by adding a property for the endpoint path. Please, see the Web Endpoints section for details, or check the GraphQlWebMvcAutoConfiguration or the GraphQlWebFluxAutoConfiguration for the actual Boot starter config.

The 1.0.x branch of this repository contains a WebFlux WebSocket sample application.

#
## Reminder / TODO 2024-01-14
```
<dependencies>
    <!-- Spring Boot Starter for WebFlux -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-webflux</artifactId>
    </dependency>

    <!-- Spring Boot Starter for WebSocket -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-websocket</artifactId>
    </dependency>

    <!-- Spring GraphQL Dependencies -->
    <dependency>
        <groupId>com.graphql-java-kickstart</groupId>
        <artifactId>graphql-spring-boot-starter</artifactId>
        <version>11.1.0</version> <!-- Use the latest version -->
    </dependency>
</dependencies>

```
```
import graphql.GraphQL;
import graphql.schema.GraphQLSchema;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class GraphQLConfig {

    @Bean
    public GraphQL graphQL(GraphQLSchema graphQLSchema) {
        return GraphQL.newGraphQL(graphQLSchema).build();
    }

    // Other GraphQL configurations go here
}

```
```
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.reactive.HandlerMapping;
import org.springframework.web.reactive.handler.SimpleUrlHandlerMapping;
import org.springframework.web.reactive.socket.WebSocketHandler;
import org.springframework.web.reactive.socket.server.support.WebSocketHandlerAdapter;

import java.util.HashMap;
import java.util.Map;

@Configuration
public class WebSocketConfig {

    @Bean
    public HandlerMapping webSocketMapping(GraphQLWebSocketHandler webSocketHandler) {
        Map<String, WebSocketHandler> map = new HashMap<>();
        map.put("/graphql", webSocketHandler);

        SimpleUrlHandlerMapping mapping = new SimpleUrlHandlerMapping();
        mapping.setOrder(1);
        mapping.setUrlMap(map);

        return mapping;
    }

    @Bean
    public WebSocketHandlerAdapter handlerAdapter() {
        return new WebSocketHandlerAdapter();
    }
}

```
```
import graphql.ExecutionInput;
import graphql.GraphQL;
import org.springframework.stereotype.Component;
import org.springframework.web.reactive.socket.WebSocketMessage;
import org.springframework.web.reactive.socket.WebSocketSession;
import reactor.core.publisher.Mono;

@Component
public class GraphQLWebSocketHandler implements org.springframework.web.reactive.socket.WebSocketHandler {

    private final GraphQL graphQL;

    public GraphQLWebSocketHandler(GraphQL graphQL) {
        this.graphQL = graphQL;
    }

    @Override
    public Mono<Void> handle(WebSocketSession session) {
        return session.receive()
                .map(WebSocketMessage::getPayloadAsText)
                .flatMap(query -> {
                    ExecutionInput executionInput = ExecutionInput.newExecutionInput()
                            .query(query)
                            .build();
                    return Mono.fromFuture(() -> graphQL.executeAsync(executionInput))
                            .map(result -> session.textMessage(result.toSpecification().toString()));
                })
                .flatMap(session::send)
                .then();
    }
}

```
