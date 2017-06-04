---
layout: spin
title:  "The Spin framework allows you to seamlessly start and stop your microservices infrastructure as part of a JUnit test run."
---

# Introduction


# Quick Start

#### 1. Add Spin to your project dependency list  

```xml
<dependency>
    <groupId>fm.pattern</groupId>
    <artifactId>spin</artifactId>
    <version>1.0.3</version>
</dependency>
```

#### 2. Write a JUnit test extending AutomatedAcceptanceTest

This is the only Java code you'll need to write.

```
import fm.pattern.spin.junit.AutomatedAcceptanceTest;

public class MyTestCase extends AutomatedAcceptanceTest {

    @Test
    public void someTest() {
    
    }

}
```

#### 3. Write start and stop scripts for your server

You can use any programming or scripting language that you like to start and stop your server, so long as Spin can execute it from the command line. 

```
#!/bin/sh
nohup mvn spring-boot:run >/dev/null 2>&1 &
```

```
#!/bin/sh
curl -s -X POST http://localhost:8081/manage/shutdown > /dev/null
```

#### 4. Place a file named spin.yml on the root of your classpath

Configure the relative location to your start and stop scripts. You must also specify a fully qualified URL to a service endpoint (a ping or healthcheck endpoint), which Spin will check to determine if your server is up and running.

```yaml
Your Server: 
  start: ../your-server/start.sh
  stop: ../your-server/stop.sh
  ping: http://localhost:9600/v1/ping
```  

#### 5. Run your test

Run your test like you normally would though your IDE or Maven. Spin will determine if your server is running, and boot it if necessary. The JUnit test suite will begin once your server is up and running. At the end of the test suite Spin will shutdown your server if it was started by Spin.