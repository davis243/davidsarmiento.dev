---
title: Introduction to BlockHound
date: 2025-05-03
featured: false

tags:
  - spring-boot
  - webflux
  - tools
authors:
  - David Sarmiento Patron


---


# Introduction


In this article, you learn how detect if your reactive non-blocking Spring app has blocking calls.
To achieve this I will use [BlockHound](https://github.com/reactor/BlockHound) 

BlockHound is a Java agent that works intercepting the blocking calls from JVM classes
BlockHound 

Requirements:
- [x] Support JDK13+
- [x] Use Project Reactor or RxJava2


### 1. Generate the base code:
- Go to https://start.spring.io/
- Project : Maven
- Dependendencies: Webflux
- 
![](start-spring.png#center)
### 2. Add dependencies:
- pom.xml
```xml
<dependency>  
    <groupId>io.projectreactor.tools</groupId>  
    <artifactId>blockhound</artifactId>  
    <version>1.0.11.RELEASE</version>  
</dependency>
```

- pom.xml build section
The argument -XX:+AllowRedefinitionToAddDeleteMethods is added to maven-surefire-plugin configuration because it is responsible for test execution. 
Also,   -XX:+AllowRedefinitionToAddDeleteMethods is included in the spring-boot-maven-plugin configuration because it is necessary when packaged application is executed 
```xml
 <build>
	<plugins>
...
		<plugin>
			<groupId>org.apache.maven.plugins</groupId>
			<artifactId>maven-surefire-plugin</artifactId>
			<version>3.1.2</version>
			<configuration>
				<argLine>-XX:+AllowRedefinitionToAddDeleteMethods</argLine>
			</configuration>
		</plugin>
		<plugin>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-maven-plugin</artifactId>
			<configuration>
				<jvmArguments>-XX:+AllowRedefinitionToAddDeleteMethods</jvmArguments>
			</configuration>
		</plugin>
	</plugins>
</build>
```

### 3. Configuration
BlockhoundDemoApplication.java
```java
@SpringBootApplication  
public class BlockhoundDemoApplication {  
  
    public static void main(String[] args) {  
       BlockHound.install();  //add to install BlockHound
       SpringApplication.run(BlockhoundDemoApplication.class, args);  
    }  
  
}
```

### 4. Testing BlockHound

For testing BlockHound is working properly, I used a simple blocking and non-blocking code:
```java
@RestController  
public class BlockingController {  
  
    @GetMapping("/blocking")  
    public Mono<String> blockingEndpoint() {  
        return Mono.just("Starting blocking code...")  
                .doOnNext(s -> {  
                    try {  
                        Thread.sleep(2000);  
                    } catch (InterruptedException e) {  
                        Thread.currentThread().interrupt();  
                    }  
                })  
                .map(s -> s + "Finishing blocking code...");  
    }  
  
    @GetMapping("/non-blocking")  
    public Mono<String> nonBlockingEndpoint() {  
        return Mono.just("Non-blocking code");  
    }  
}
```
### 5. JUnit Jupiter and BlockHound
Add this method as static for work with only one BlockHound instance .
```java
@BeforeAll  
static void setupBlockHound() {  
    BlockHound.install();  
}
```

### 6. Results
After call http://localhost:8080/blocking
we can see this:
```sh
reactor.blockhound.BlockingOperationError: Blocking call! java.lang.Thread.sleepNanos0
	at java.base/java.lang.Thread.sleepNanos0(Thread.java) ~[na:na]
	Suppressed: reactor.core.publisher.FluxOnAssembly$OnAssemblyException: 
Error has been observed at the following site(s):
	*__checkpoint ⇢ Handler dev.davidsarmiento.blockhounddemo.controller.BlockingController#blockingEndpoint() [DispatcherHandler]
	*__checkpoint ⇢ HTTP GET "/blocking" [ExceptionHandlingWebHandler]
```

# Conclusions
- BlockHound dectect propertly the non-blocking code in our Spring WebFlux App.
- BlockHound can be used in the main application as well as on unit tests.

# Source code
[Github source](https://github.com/davis243/blockhound-demo)