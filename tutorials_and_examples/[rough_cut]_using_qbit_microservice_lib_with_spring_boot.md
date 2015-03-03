# [Rough Cut] Using QBit microservice lib with Spring Boot

You can use Spring Boot and QBit together. You could configure QBit to work with/in Spring MVC, or you can just use QBit as the servlet that handles requests.

Remember this about QBit. It runs standalone with Jetty or Vertx. It can run in any Servlet container.

Here is a basic example combining QBit in the Spring Boot world.

#### Project structure
```bash
$ tree
.
├── build.gradle
├── gradle
│   └── wrapper
│       ├── gradle-wrapper.jar
│       └── gradle-wrapper.properties
├── gradlew
├── gradlew.bat
├── qbit-spring-boot-together.iml
├── settings.gradle
└── src
    └── main
        ├── java
        │   └── io
        │       └── advantageous
        │           └── qbit
        │               └── examples
        │                   └── spring
        │                       ├── Application.java
        │                       ├── DispatcherServlet.java
        │                       └── HelloService.java
        └── resources
            └── default.properties

11 directories, 11 files

```

We present the above which is a simple project structure.
We used gradle but you could easily use maven.


First we define the Application and Configuration bean as follows:

```java


/**
 * @author Geoff Chandler
 * @author  (Rick Hightower)
 */
@Configuration
@EnableAutoConfiguration
@PropertySource(value = {"classpath:default.properties",
        "file:${properties.location}"},
        ignoreResourceNotFound = true)
public class Application extends SpringBootServletInitializer {

    @Autowired
    private Environment environment;

    public static void main(String[] args) throws Exception {
        SpringApplication.run(Application.class, args);
    }

    @Bean
    public String dataDir() {
        return environment.getProperty("data.location");
    }

    @Bean
    public HelloService helloService() {
        return new HelloService();
    }

    @Bean
    public DispatcherServlet dispatcherServlet() {
        return new DispatcherServlet();
    }

    @Override
    protected SpringApplicationBuilder configure(SpringApplicationBuilder application) {
        return application.sources(Application.class);
    }
}

```

There are many ways to do this. This example picked a simple approach.

Next we subclass a QBit Servlet as follows:

```java

package io.advantageous.qbit.examples.spring;

import io.advantageous.qbit.http.HttpTransport;
import io.advantageous.qbit.server.ServiceServer;
import io.advantageous.qbit.servlet.QBitHttpServlet;
import org.springframework.beans.factory.annotation.Autowired;

import javax.servlet.ServletConfig;

import static io.advantageous.qbit.server.ServiceServerBuilder.serviceServerBuilder;

/**
 * @author Rick Hightower
 */
public class DispatcherServlet extends QBitHttpServlet {

    public static final String SERVICES_API_PROXY_URI_PREAMBLE = "/services/myapp/";

    @Autowired
    private HelloService helloService;
    //Hit this at http://localhost:8080/services/myapp/helloservice/hello


    private ServiceServer serviceServer;

    public DispatcherServlet() {

    }

    @Override
    protected void stop() {
        serviceServer.stop();
    }

    @Override
    protected void wireHttpServer(final HttpTransport httpTransport,
                                  final ServletConfig servletConfig) {


        serviceServer = serviceServerBuilder().setHttpTransport(httpTransport)
                .setUri(SERVICES_API_PROXY_URI_PREAMBLE)
                .build().initServices(helloService).startServer();

    }

}

```

Now one could imagine that we could do all sorts of thing with Spring like discover servers and what not.
We could also write an adapter so QBit can run inside of Spring MVC. This would not be hard.
The above just uses the QBit support for adapting QBit to run inside of a servlet container.

Lastly is our Hello World example which we injected into our Servlet.

```java
import io.advantageous.qbit.annotation.RequestMapping;

@RequestMapping("helloservice")
public class HelloService {


    @RequestMapping("hello")
    public String helloWorld() {
        return "Hello from QBit";
    }
}

```

If you are skilled with Spring, you could imagine creating lifecycle listeners and do a lot of the service discovery on the fly. (I did this before with Crank and Geoff did it before as well.)


Here is the build file.

```groovy

apply plugin: 'idea'
apply plugin: 'java'
apply plugin: 'application'


sourceCompatibility = 1.8
version = '1.0'

repositories {
    mavenLocal()
    mavenCentral()
}


dependencies {
    compile group: 'io.advantageous.qbit', name: 'qbit-servlet', version: '0.6.1-SNAPSHOT'
    compile group: 'javax.inject', name: 'javax.inject', version: '1'
    compile('org.springframework.boot:spring-boot-starter-web:1.2.1.RELEASE') {
        exclude module: 'spring-boot-starter-tomcat'
    }
    compile 'org.eclipse.jetty:jetty-webapp:9.+'
    compile 'org.eclipse.jetty:jetty-jsp:9.+'

    testCompile "junit:junit:4.11"
    testCompile "org.slf4j:slf4j-simple:[1.7,1.8)"
}
```

You can find the complete example in git.

https://github.com/advantageous/qbit/tree/master/qbit-examples/tutorial/qbit-spring-boot-together
