# 스프링 부트와 내장 톰캣
## WAR 배포 방식의 단점

웹 애플리케이션을 개발하고 배포하려면 다음과 같은 과정을 거쳐야 한다.
* 톰캣 같은 웹 애플리케이션 서버(WAS)를 별도로 설치해야 한다.
* 애플리케이션 코드를 WAR로 빌드해야 한다.
* 빌드한 WAR 파일을 WAS에 배포해야 한다.

단점
* 톰캣 같은 WAS를 별도로 설치해야 한다.
* 개발 환경 설정이 복잡하다.
    * 단순한 자바라면 별도의 설정을 고민하지 않고, `main()` 메서드만 실행하면 된다.
    * 웹 애플리케이션은 WAS 실행하고 또 WAR와 연동하기 위한 복잡한 설정이 들어간다.
* 배포 과정이 복잡하다. WAR를 만들고 이것을 또 WAS에 전달해서 배포해야 한다.
* 톰캣의 버전을 변경하러면 톰캣을 다시 설치해야 한다.

## 내장 톰캣 스프링 컨테이너
```java
public class EmbedTomcatSpringMain {

    public static void main(String[] args) throws LifecycleException {
        System.out.println("EmbedTomcatServletMain.main");

        // 톰캣 설정
        Tomcat tomcat = new Tomcat();
        Connector connector = new Connector();
        connector.setPort(8080);
        tomcat.setConnector(connector);

        // 스프링 컨테이너 설정
        AnnotationConfigWebApplicationContext applicationContext = new AnnotationConfigWebApplicationContext();
        applicationContext.register(HelloConfig.class);

        // 스프링 MVC 디스패처 서블릿 생성, 스프링 컨테이너 연결
        DispatcherServlet dispatcherServlet = new DispatcherServlet(applicationContext);

        // 디스패처 서블릿 등록
        Context context = tomcat.addContext("", "/");
        tomcat.addServlet("", "dispatcher", dispatcherServlet);
        context.addServletMappingDecoded("/", "dispatcher");

        tomcat.start();
    }
}
```

## 내장 톰캣 빌드와 배포
자바의 `main()` 메서드를 실행하기 위해서는 `jar` 형식으로 빌드해야 한다.
그리고 `jar` 안에는 `META-INF/MANIFEST.MF` 파일에 실행할 `main()` 메서드의 클래스를 지정해주어야 한다.

`META-INF/MANIFEST.MF`
```
Manifest-Version: 1.0
Main-Class: hello.embed.EmbedTomatSpringMain
```

Gradle의 도움을 받으면 이 과정을 쉽게 진행할 수 있다.
`build.gradle - buildJar`
```groovy
task buildJar(type: Jar) {
  manifest {
    attributes 'Main-Class': 'hello.embed.EmbedTomcatSpringMain'
  }
  with jar
}
```

### jar 파일은 jar 파일을 포함할 수 없다.
* WAR와 다르게 JAR 파일은 내부에 라이브러리 역할을 하는 JAR 파일을 포함할 수 없다. 포함한다고 해도 인식이 안된다. 이것이 JAR 파일 스펙의 한계이다. 그렇다고 WAR를 사용할 수 도 없다.  WAR는 웹 애플리케이션 서버(WAS) 위에서만 실행할 수 있다.
* 대안으로 라이브러리 jar 파일을 모두 구해서 MANIFEST 파일에 해당 경로를 적어주면 인식이 되지만 매우 번거롭고, JAR 파일안에 JAR 파일을 포함할 수 없기 때문에 라이브러리 역할을 하는 jar 파일도 항상 함께 지니고 다녀야 한다. 이 방법은 권장하지 않는다.

## FatJar
대안으로 `fat jar` 또는 `uber jar`라고 불리는 방법이다.
Jar안에 Jar를 포함할 수 없다. 하지만 클래스는 얼마든지 포함할 수 있다.
라이브러리에 사용되는 jar를 풀면 `class`들이 나온다. 이 `class`를 뽑아서 새로 만드는 jar에 포함하는 것이다.
이렇게 하면 수많은 라이브러리에서 나오는 `class` 때문에 뚱뚱한 jar가 탄생한다 그래서 fat jar라고 부르는 것이다.

`build.gradle - buildJar`
```groovy
task buildFatJar(type: Jar) {
  manifest {
    attributes 'Main-Class': 'hello.embed.EmbedTomcatSpringMain'
  }
  duplicatesStrategy = DuplicateStrategy.WARN
  from { configurations.runtimeClasspath.collect { it.isDirectory() ? it : zipTree(it) } }
  with jar
}
```

### Fat Jar의 장점
* Fat Jar 덕분에 하나의 jar 파일에 필요한 라이브러리들을 내장할 수있게 되었다.
* 내장 톰캣 라이브러리를 jar 내부에 내장할 수 있게 되었다.
* 덕분에 하나의 jar 파일로부터 배포부터, 웹 서버 설치 + 실행까지 모든 것을 단순화 할 수 있다.

### Fat Jar의 단점
* 어떤 라이브러리가 포함되었는지 확인하기 어렵다.
  * 모두 class로 풀려있으니 어떤 라이브러리가 사용되고 있는지 추적하기 어렵다.
* 파일명 중복을 해결할 수 없다.
  * 클래스나 리소스명이 같은 경우 하나를 포리해야 한다. 이것은 심각한 문제를 발생하게 한다. 서블릿 컨테이너 초기화에서 학습한 부분을 떠올려보자.
  * `META-INF/services/jakarta.servlet.ServletContainerInitializer` 파일이 여러 라이브러리에 있을 수 있다.
  * A 라이브러리와 B 라이브러리 둘다 해당 파일을 사용해서 서블릿 컨테이너 초기화를 시도한다. 둘다 해당 파일을 jar안에 포함한다.
  * Fat Jar를 만들면 파일명이 같으므로 A, B 라이브러리가 둘다 가지고 있는 파일중에 하나의 파일만 선택된다. 결과적으로 나머지 하나는 포함되지 않으므로 정상적으로 작동하지 않는다.

## 편리한 부트 클래스 만들기
지금까지 진행한 내장 톰캣 실행, 스프링 컨테이너 생성, 디스패처 서블릿 등록의 모든 과정을 편리하게 처리해주는 나만의 부트 클래스를 만들어보자. 부트는 이름 그대로 시작을 편하게 처리해주는 것을 뜻한다.

MySpringApplication
```java
import org.apache.catalina.Context;
import org.apache.catalina.LifecycleException;
import org.apache.catalina.connector.Connector;
import org.apache.catalina.startup.Tomcat;
import org.springframework.web.context.support.AnnotationConfigWebApplicationContext;
import org.springframework.web.servlet.DispatcherServlet;
import java.util.List;

public class MySpringApplication { 
    public static void run(Class configClass, String[] args) {
    
        System.out.println("MySpringBootApplication.run args=" + List.of(args));
        //톰캣 설정
        Tomcat tomcat = new Tomcat();
        Connector connector = new Connector();
        connector.setPort(8080);
        tomcat.setConnector(connector);

        //스프링 컨테이너 생성
        AnnotationConfigWebApplicationContext appContext = new 
        AnnotationConfigWebApplicationContext();
        appContext.register(configClass);

        //스프링 MVC 디스패처 서블릿 생성, 스프링 컨테이너 연결
        DispatcherServlet dispatcher = new DispatcherServlet(appContext);
        
        //디스패처 서블릿 등록
        Context context = tomcat.addContext("", "/");
        tomcat.addServlet("", "dispatcher", dispatcher);
        context.addServletMappingDecoded("/", "dispatcher");
        try {
            tomcat.start();
        } catch (LifecycleException e) {
            throw new RuntimeException(e);
        }
    }
 }
```
* `configClass`: 스프링 설정을 파라미터로 전달 받는다.
* `tomcat.start()`에서 발생한 예외는 잡아서 런타임 예외로 변경했다.

@MySpringBootApplication

```java
import org.springframework.context.annotation.ComponentScan;
import java.lang.annotation.*;

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@ComponentScan
public @interface MySpringBootApplication { }
```
* 컴포넌트 스캔 기능이 추가된 단순한 애노테이션이다.
* 시작할 때 이 애노테이션을 붙여서 사용하면 된다.

MySpringBootAppMain
```java
import hello.boot.MySpringApplication;
import hello.boot.MySpringBootApplication;

@MySpringBootApplication
public class MySpringBootMain {
    
  public static void main(String[] args) {
    System.out.println("MySpringBootMain.main");
    MySpringApplication.run(MySpringBootMain.class, args);
  }
}
```
* 패키지 위치가 중요하다. 최상위 패키지에 위치했다.
* 여기에 위치한 이유는 `@MySpringBootApplication`에 컴포넌트 스캔이 추가 되어있는데, 컴포넌트 스캔의 기본 동작은 해당 애노테이션이 붙은 클래스의 현재 피키지부터 그 하위 패키지를 컴포넌트 스캔의 대상으로 사용하기 때문이다.
