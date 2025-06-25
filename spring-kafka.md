### Project Structure

A well-organized project structure is crucial for maintainability.

```
.
├── build.gradle
├── gradle
│   └── wrapper
│       ├── gradle-wrapper.jar
│       └── gradle-wrapper.properties
├── gradlew
├── gradlew.bat
├── src
│   ├── main
│   │   ├── avro
│   │   │   └── SimpleMessage.avsc
│   │   ├── java
│   │   │   └── com
│   │   │       └── example
│   │   │           └── kafkaproducer
│   │   │               ├── config
│   │   │               │   └── KafkaTopicConfig.java
│   │   │               ├── model
│   │   │               │   └── SimpleMessage.java
│   │   │               ├── producer
│   │   │               │   └── MessageProducer.java
│   │   │               └── KafkaProducerApplication.java
│   │   └── resources
│   │       └── application.yaml
│   └── test
│       └── java
│           └── com
│               └── example
│                   └── kafkaproducer
│                       └── producer
│                           └── MessageProducerTest.java
```

### 1. Gradle Build File (`build.gradle`)

This file defines project dependencies and the Avro plugin for code generation.

```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.5.0-SNAPSHOT'
    id 'io.spring.dependency-management' version '1.1.6'
    id 'com.github.davidmc24.gradle.plugin.avro' version '1.9.1'
}

group = 'com.example'
version = '0.0.1-SNAPSHOT'

java {
    sourceCompatibility = '17'
}

repositories {
    mavenCentral()
    maven { url 'https://repo.spring.io/snapshot' }
    maven { url 'https://packages.confluent.io/maven/' }
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter'
    implementation 'org.springframework.kafka:spring-kafka'
    implementation 'org.apache.avro:avro:1.11.3'
    implementation 'io.confluent:kafka-avro-serializer:7.6.0'

    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'org.springframework.kafka:spring-kafka-test'
}

generateAvroJava {
    source "src/main/avro"
    outputDir = file("src/main/java")
    stringType = "String"
    useDecimalType = true
    enableDecimalLogicalType = true
}

tasks.withType(JavaCompile) {
    dependsOn 'generateAvroJava'
}

tasks.named('test') {
    useJUnitPlatform()
}
```

### 2. Avro Schema (`SimpleMessage.avsc`)

This schema defines the structure of the message to be sent.

```json
{
  "namespace": "com.example.kafkaproducer.model",
  "type": "record",
  "name": "SimpleMessage",
  "fields": [
    {
      "name": "timestamp",
      "type": {
        "type": "long",
        "logicalType": "timestamp-millis"
      }
    },
    {
      "name": "content",
      "type": "string"
    }
  ]
}
```

After defining the schema, run `./gradlew build` to generate the `SimpleMessage.java` class from the Avro schema.

### 3. Spring Boot Application (`KafkaProducerApplication.java`)

This is the main entry point for the Spring Boot application.

```java
package com.example.kafkaproducer;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class KafkaProducerApplication {

    public static void main(String[] args) {
        SpringApplication.run(KafkaProducerApplication.class, args);
    }
}
```

### 4. Kafka Topic Configuration (`KafkaTopicConfig.java`)

This configuration class programmatically defines the Kafka topics.

```java
package com.example.kafkaproducer.config;

import org.apache.kafka.clients.admin.NewTopic;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.config.TopicBuilder;

@Configuration
public class KafkaTopicConfig {

    @Bean
    public NewTopic topicA() {
        return TopicBuilder.name("topicA")
                .partitions(1)
                .replicas(1)
                .build();
    }

    @Bean
    public NewTopic topicB() {
        return TopicBuilder.name("topicB")
                .partitions(1)
                .replicas(1)
                .build();
    }
}
```

### 5. Application Configuration (`application.yaml`)

This file contains the configuration for the Kafka producer, including SSL and schema registry details.

```yaml
spring:
  kafka:
    producer:
      bootstrap-servers: localhost:9092
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: io.confluent.kafka.serializers.KafkaAvroSerializer
      properties:
        schema.registry.url: http://localhost:8081
    properties:
      security.protocol: SSL
      ssl:
        truststore:
          location: classpath:kafka.truststore.jks
          password: password
        keystore:
          location: classpath:kafka.keystore.jks
          password: password
          key-password: password
```

### 6. Kafka Producer Service (`MessageProducer.java`)

This service uses `KafkaTemplate` to send messages to the Kafka topics.

```java
package com.example.kafkaproducer.producer;

import com.example.kafkaproducer.model.SimpleMessage;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;

import java.time.LocalDateTime;
import java.time.ZoneOffset;

@Service
public class MessageProducer {

    private final KafkaTemplate<String, SimpleMessage> kafkaTemplate;

    public MessageProducer(KafkaTemplate<String, SimpleMessage> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    public void sendMessage(String topic, String content) {
        long timestamp = LocalDateTime.now().toInstant(ZoneOffset.UTC).toEpochMilli();
        SimpleMessage message = new SimpleMessage(timestamp, content);
        kafkaTemplate.send(topic, message);
    }
}
```

### 7. Integration Test (`MessageProducerTest.java`)

This test utilizes an embedded Kafka broker to verify the producer's functionality without needing an external Kafka instance.

```java
package com.example.kafkaproducer.producer;

import com.example.kafkaproducer.model.SimpleMessage;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.kafka.core.DefaultKafkaConsumerFactory;
import org.springframework.kafka.listener.ContainerProperties;
import org.springframework.kafka.listener.KafkaMessageListenerContainer;
import org.springframework.kafka.listener.MessageListener;
import org.springframework.kafka.test.context.EmbeddedKafka;
import org.springframework.kafka.test.utils.ContainerTestUtils;
import org.springframework.kafka.test.utils.KafkaTestUtils;

import java.util.Map;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;
import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
@EmbeddedKafka(partitions = 1, brokerProperties = { "listeners=PLAINTEXT://localhost:9092", "port=9092" })
class MessageProducerTest {

    @Autowired
    private MessageProducer messageProducer;

    @Test
    void testSendToTopicA() throws InterruptedException {
        // GIVEN
        String topic = "topicA";
        String messageContent = "Hello Topic A";
        BlockingQueue<ConsumerRecord<String, SimpleMessage>> records = new LinkedBlockingQueue<>();
        KafkaMessageListenerContainer<String, SimpleMessage> container = createContainer(topic, records);
        container.start();
        ContainerTestUtils.waitForAssignment(container, 1);

        // WHEN
        messageProducer.sendMessage(topic, messageContent);

        // THEN
        ConsumerRecord<String, SimpleMessage> received = records.poll(10, TimeUnit.SECONDS);
        assertThat(received).isNotNull();
        assertThat(received.topic()).isEqualTo(topic);
        assertThat(received.value().getContent()).isEqualTo(messageContent);

        container.stop();
    }

    @Test
    void testSendToTopicB() throws InterruptedException {
        // GIVEN
        String topic = "topicB";
        String messageContent = "Hello Topic B";
        BlockingQueue<ConsumerRecord<String, SimpleMessage>> records = new LinkedBlockingQueue<>();
        KafkaMessageListenerContainer<String, SimpleMessage> container = createContainer(topic, records);
        container.start();
        ContainerTestUtils.waitForAssignment(container, 1);

        // WHEN
        messageProducer.sendMessage(topic, messageContent);

        // THEN
        ConsumerRecord<String, SimpleMessage> received = records.poll(10, TimeUnit.SECONDS);
        assertThat(received).isNotNull();
        assertThat(received.topic()).isEqualTo(topic);
        assertThat(received.value().getContent()).isEqualTo(messageContent);

        container.stop();
    }

    private KafkaMessageListenerContainer<String, SimpleMessage> createContainer(
            String topic, BlockingQueue<ConsumerRecord<String, SimpleMessage>> records) {
        Map<String, Object> consumerProps = KafkaTestUtils.consumerProps("testGroup", "true", "localhost:9092");
        DefaultKafkaConsumerFactory<String, SimpleMessage> consumerFactory = new DefaultKafkaConsumerFactory<>(consumerProps);
        ContainerProperties containerProperties = new ContainerProperties(topic);
        KafkaMessageListenerContainer<String, SimpleMessage> container = new KafkaMessageListenerContainer<>(consumerFactory, containerProperties);
        container.setupMessageListener((MessageListener<String, SimpleMessage>) records::add);
        return container;
    }
}
```

### Running the Application and Tests

To run the application and its tests, you will need to have Docker installed and running to start a Kafka and Schema Registry instance. You can use a `docker-compose.yml` file for this purpose. Once the Kafka environment is up, you can run the tests and the application using Gradle.
