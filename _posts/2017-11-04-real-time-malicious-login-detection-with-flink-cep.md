---
layout: post
title:  "Real-time Malicious Login Detector with Apache Flink CEP"
date:   2017-11-04 00:06:36 +0800
categories: flink, java
permalink: real-time-malicious-login-detector-with-flink-cep
---

I was playing with Apache Flink several weeks ago. For exercise, I made a simple rule to detect malicious login pattern with Complex Event Processing (CEP). The rule is very simple, 3x login activities for the same username and IP in a minute. Then for the data source, I use Kafka.

<b>pom.xml</b>
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.wilianto.blog.login</groupId>
    <artifactId>cep-malicious-login</artifactId>
    <version>1.0-SNAPSHOT</version>

    <packaging>jar</packaging>

    <dependencies>
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-streaming-java_2.10</artifactId>
            <version>1.3.2</version>
        </dependency>
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-connector-kafka-0.10_2.10</artifactId>
            <version>1.3.2</version>
        </dependency>
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-cep_2.10</artifactId>
            <version>1.3.2</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <artifactId>maven-assembly-plugin</artifactId>
                <version>2.3</version>
                <configuration>
                    <descriptorRefs>
                        <descriptorRef>jar-with-dependencies</descriptorRef>
                    </descriptorRefs>
                    <archive>
                        <manifest>
                            <mainClass>com.wilianto.blog.login.App</mainClass>
                        </manifest>
                    </archive>
                </configuration>
                <executions>
                    <execution>
                        <id>make-assembly</id>
                        <phase>package</phase>
                        <goals>
                            <goal>single</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

<b>MaliciousLoginEvent.java</b>

```java
public class MaliciousLoginEvent {
    private String username;
    private String ip;
    private String time;

    //setter
    //getter
} 
```

The data will be saved in JSON format in the Kafka topic. So I created a deserializer for JSON to MaliciousLoginEvent POJO.

<b>MaliciousLoginDeserializationSchema.java</b>

```java
public class MaliciousLoginDeserializationSchema 
    implements DeserializationSchema<MaliciousLoginEvent> {
    public MaliciousLoginEvent deserialize(byte[] bytes) 
        throws IOException {
        MaliciousLoginEvent maliciousLoginEvent = null;
        try {
            JSONObject jsonData = new JSONObject(new String(bytes));
            maliciousLoginEvent = new MaliciousLoginEvent();
            maliciousLoginEvent.setUsername(jsonData.getString("username"));
            maliciousLoginEvent.setIp(jsonData.getString("ip"));
            maliciousLoginEvent.setTime(jsonData.getString("time"));
        } catch (JSONException e) {
            System.out.println("Unable to deserialize Malicious Login Kafka Message");
        } finally {
            return maliciousLoginEvent;
        }
    }

    public boolean isEndOfStream(MaliciousLoginEvent maliciousLoginEvent) {
        return false;
    }

    public TypeInformation<MaliciousLoginEvent> getProducedType() {
        return TypeExtractor.getForClass(MaliciousLoginEvent.class);
    }
}
```

The main code does:
1. Add streaming data source from Kafka topic (FailLogin)
2. Create a malicious login pattern (3x failed with the same username & IP in a minute)
3. Match the pattern with coming data
4. Add sink to print out the malicious pattern

<b>App.java</b>

```java
public class App {
    public static void main(String[] args) throws Exception {
        //set kafka properties
        Properties properties = new Properties();
        properties.setProperty("bootstrap.servers", "localhost:9092");

        //set stream execution environtment
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        //stream kafka
        FlinkKafkaConsumer010<MaliciousLoginEvent> flinkKafkaConsumer = new FlinkKafkaConsumer010<MaliciousLoginEvent>("FailLogin", new MaliciousLoginDeserializationSchema(), properties);
        flinkKafkaConsumer.setStartFromLatest();
        DataStream<MaliciousLoginEvent> kafkaInputStream = env.addSource(flinkKafkaConsumer).keyBy("username", "ip");

        //create pattern
        Pattern<MaliciousLoginEvent, ?> pattern = Pattern.<MaliciousLoginEvent>begin("firstAttempt")
                .next("secondAttempt")
                .next("thirdAttempt")
                .within(Time.minutes(1));
        PatternStream<MaliciousLoginEvent> patternStream = CEP.pattern(kafkaInputStream, pattern);

        //add simple sink alarm
        DataStream<String> result = patternStream.select(new PatternSelectFunction<MaliciousLoginEvent, String>() {
            public String select(Map<String, List<MaliciousLoginEvent>> suspects) throws Exception {
                MaliciousLoginEvent firstAttempt = suspects.get("firstAttempt").get(0);
                MaliciousLoginEvent secondAttempt = suspects.get("secondAttempt").get(0);
                MaliciousLoginEvent thirdAttempt = suspects.get("thirdAttempt").get(0);
                String message = String.format("Suspected Login Activity! \n" +
                        "First event: %s \n" +
                        "Second event: %s \n" +
                        "Third event: %s \n", firstAttempt, secondAttempt, thirdAttempt);

                System.out.println(message);

                return message;
            }
        });

        //execute
        env.execute("Mallicious Login Detector");
    }
}
```

Then for testing purpose, I made a simple ruby code in Sinatra to simulate login request. When login failed, an event will be sent to a Kafka topic in JSON format.

<b>login.rb</b>

```ruby
require "sinatra"
require "kafka"
require "json"

user = { username: "wilianto", password: "12345678" }

post "/login" do
  username = params["username"]
  password = params["password"]
  ip       = params["ip"] # should be request.ip, just for testing purpose

  if user[:username] == username && user[:password] == password
    "Login success"
  else
    event = {
      username: username,
      ip: ip,
      time: Time.now
    }

    # push to kafka topic FailLogin
    kafka = Kafka.new(
      seed_brokers: ["localhost:9092"]
    )
    kafka.deliver_message(JSON.dump(event), topic: "FailLogin")

    "Login failed"
  end
end
```

Compile the CEP Flink apps and run the JAR file on Flink. Run the ruby login code and simulate some requests.

<img src="/assets/images/malicious-login-cep-flink-example.webp" alt="CEP Running on Flink" title="CEP Running on Flink">>

Example Request:

```bash
curl -X POST \
  http://localhost:4567/login \
  -H 'cache-control: no-cache' \
  -H 'content-type: multipart/form-data; boundary=----WebKitFormBoundary7MA4YWxkTrZu0gW' \
  -H 'postman-token: cb6cf5f1-2361-ad2d-4d9b-addd7b71b474' \
  -F username=wilianto \
  -F password=wrongpassword \
  -F ip=127.0.0.1
```

Example Output:

```bash
Suspected Login Activity!
First event: MaliciousLogin[username: wilianto, ip: 127.0.0.1, time: 2017-11-04 18:55:06 +0700]
Second event: MaliciousLogin[username: wilianto, ip: 127.0.0.1, time: 2017-11-04 18:55:12 +0700]
Third event: MaliciousLogin[username: wilianto, ip: 127.0.0.1, time: 2017-11-04 18:55:13 +0700]
```

Thanks for reading. Feel free to share what’s on your mind in the comment form below.
