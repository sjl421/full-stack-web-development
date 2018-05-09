# 使用 ElasticSearch 提升搜索性能

## 配置

### 安装 ElasticSearch

```bash
docker pull docker.elastic.co/elasticsearch/elasticsearch:6.2.4
```

### 添加依赖

在 `api/build.gradle` 中添加 `spring-boot-starter-data-elasticsearch` 依赖

```groovy
// 省略
dependencies {
    //省略
    implementation("org.springframework.boot:spring-boot-starter-data-redis")
    implementation("org.springframework.boot:spring-boot-starter-data-elasticsearch") // <--- 这里
    testImplementation("org.springframework.security:spring-security-test")
}
```

### 添加配置类

```java
package dev.local.gtm.api.config;

import com.fasterxml.jackson.databind.DeserializationFeature;
import com.fasterxml.jackson.databind.ObjectMapper;
import lombok.RequiredArgsConstructor;
import org.elasticsearch.client.transport.TransportClient;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.boot.autoconfigure.data.elasticsearch.ElasticsearchProperties;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.elasticsearch.core.ElasticsearchTemplate;
import org.springframework.data.elasticsearch.core.EntityMapper;
import org.springframework.http.converter.json.Jackson2ObjectMapperBuilder;

import java.io.IOException;

@RequiredArgsConstructor
@Configuration
@EnableConfigurationProperties(ElasticsearchProperties.class)
@ConditionalOnProperty("spring.data.elasticsearch.cluster-nodes")
public class ElasticsearchConfig {

    private final TransportClient transportClient;

    @Bean
    public ElasticsearchTemplate elasticsearchTemplate(Jackson2ObjectMapperBuilder jackson2ObjectMapperBuilder) {
        return new ElasticsearchTemplate(transportClient, new CustomEntityMapper(jackson2ObjectMapperBuilder.createXmlMapper(false).build()));
    }

    public class CustomEntityMapper implements EntityMapper {

        private ObjectMapper objectMapper;

        public CustomEntityMapper(ObjectMapper objectMapper) {
            this.objectMapper = objectMapper;
            objectMapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
            objectMapper.configure(DeserializationFeature.ACCEPT_SINGLE_VALUE_AS_ARRAY, true);
        }

        @Override
        public String mapToString(Object object) throws IOException {
            return objectMapper.writeValueAsString(object);
        }

        @Override
        public <T> T mapToObject(String source, Class<T> clazz) throws IOException {
            return objectMapper.readValue(source, clazz);
        }
    }
}
```
