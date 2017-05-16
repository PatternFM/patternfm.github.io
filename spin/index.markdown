---
layout: spin
title:  "The Spin framework allows you to seamlessly start and stop your microservices infrastructure as part of a JUnit test run."
---

# Introduction


# Quick Start

#### 1. Add Spin to your project dependency list


**Maven**   

```xml
<dependency>
    <groupId>fm.pattern</groupId>
    <artifactId>spin</artifactId>
    <version>1.0.3</version>
</dependency>
```

**Gradle**

```
compile group: 'fm.pattern', name: 'spin', version: '1.0.3'
```

#### 2. Write a JUnit test extending AutomatedAcceptanceTest

```
import fm.pattern.spin.junit.AutomatedAcceptanceTest;

public class MyTestCase extends AutomatedAcceptanceTest {

    @Test
    public void someTest() {
    
    }

}
```
#### 3. Write start and stop scripts for your server

```
#!/bin/bash

nohup mvn spring-boot:run -Dspring.profiles.active=$SPRING_PROFILE >/dev/null 2>&1 &
```

```
#!/bin/bash

curl -s -X POST http://localhost:9601/manage/shutdown > /dev/null
```


#### 4. Place a file named spin.yml on the root of your classpath

```yaml
Tokamak Server: 
  start: ../your-server/start.sh
  stop: ../your-server/stop.sh
  ping: http://localhost:9600/v1/ping
  path: ":/usr/local/bin"
```  