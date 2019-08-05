---
title: SpringBoot 버전 마이그레이션 (1.3.8 -> 2.1.5)
layout: post
---

# maven module project 버전 변경
대상 : 버저닝을 하는 멀티 모듈 프로젝트 (운영 배포시 마다 버전 업)

1.~ 대 버전의 프로젝트의 버전을 2.~ 대로 일괄 변경 한다.

[참고 사이트](https://maven.apache.org/maven-release/maven-release-plugin/examples/update-versions.html)

```bash
mvn --batch-mode release:update-versions -DautoVersionSubmodules=true -DdevelopmentVersion=2.0.0-SNAPSHOT
```

# springboot 버전 변경 (1.3.8 -> 1.5.21)
[springboot migration guide](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-2.0-Migration-Guide) 에 따르면 springboot 1.5에서 2.0으로 마이그레이션 하는 방법이 소개 되어 있는데, 현재 사용하는 버전이 1.3.8이니 1.5로 변경 후 2.0으로 마이그레이션 작업을 진행 한다.

springboot인 경우 spring-boot-starter-parent pom 파일을 상속 받으면, springboot parent의 dependency mananger가 동작하며, 이로 인해 관련 라이브러리에 버전이 변경 된다.

spring boot parent pom을 상속 하지 않는 경우,  dependency manager에 spring-cloud-starter-parent pom과 platform-bom bom을 dependency manager에서 import 하여 사용하면 된다.

## spring-cloud-starter 버전 변경

spring-cloud-starter-parent을 import 하기위해 1.5.21에 맞는 버전을 찾아야 한다.
[참고 사이트](https://spring.io/projects/spring-cloud)를 보면 spring boot 버전에 호환되는 버전이 명시 되어 있다.

Release Train  |  Boot Version
---|---
`Greenwich` | 2.1.x |
`Finchley` | 2.0.x |
`Edgware` | 1.5.x |
`Dalston` | 1.5.x |

맞는 버전은 Edgware이므로 [maven public repo](https://mvnrepository.com/artifact/org.springframework.cloud/spring-cloud-starter-parent/Edgware.RELEASE)에서 확인을 하고 dependency의 버전을 변경 하자.

## io.spring.platform platform-bom 버전 변경

[Spring IO Platform 문서](https://spring.io/projects/platform)를 보니 2019.4.9일 까지 Support를 하며, 가이드 상으로는 spring-boot-starter-parent나 spring-boot-dependency를 사용할 것을 권고한다.

spring-boot-starter-parent를 사용 못할 경우 spring-boot-dependency를 dependency manager에서 import 하여 사용한다.

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-dependencies</artifactId>
            <version>${spring-boot.version}</version>
            <type>pom</type>
            <scope>import</scope>
         </dependency>
    </dependencies>
</dependencyManagement>
```

### 버전 변경에 따른 에러 제거 작업 

package 위치 변경 
* lombok.experimental.Builder -> lombok.Builder
* org.springframework.boot.context.embedded.ErrorPage -> org.springframework.boot.web.servlet.ErrorPage
* org.springframework.boot.context.embedded.FilterRegistrationBean -> org.springframework.boot.web.servlet.FilterRegistrationBean

annotation 변경
* @EnableDiscoveryClient -> @EnableDiscoveryClient
* @SpringApplicationConfiguration -> @SpringBootTest
* @WebIntegrationTest (없어진듯)

dependency 변경 
* spring-cloud-starter-eureka -> spring-cloud-starter-netflix-eureka-client

기존에 관리 되었으나 spring-boot-dependency나 spring-cloud-starter에서 관리 되지 않는 dependency에 대한 버전 재 정의

# springboot 버전 변경 (1.5.21 -> 2.1.6)
"1.5.21" 로 중간에 업그레이드 후 2.1.6으로 업그레이드를 진행 하였으나, 진행 해본 결과 굳이 중간에 1.5.21로 upgrade할 이유가 미흡함.

"2.1.6" 으로 업그레이드는 아래와 같이 2개의 dependency import 하면 됨

```xml
		<dependencies>
			<dependency>
				<groupId>org.springframework.cloud</groupId>
				<artifactId>spring-cloud-starter-parent</artifactId>
				<version>Greenwich.RELEASE</version>
				<type>pom</type>
				<scope>import</scope>
			</dependency>
			<dependency>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-dependencies</artifactId>
				<version>2.1.6.RELEASE</version>
				<type>pom</type>
				<scope>import</scope>
			</dependency>
		</dependencies>
```

##  EmbeddedServletContainerCustomizer -> WebServerFactoryCustomizer

reactive를 지원하기 위해 EmbeddedServletContainer가 WebServer로 변경되었고, EmbeddedServletContainerCustomizer는 WebServerFactoryCustomizer 변경 되었다. 

일단 tomcat servlet만 지원하기 위해 EmbeddedServletContainerCustomizer를 ```WebServerFactoryCustomizer<TomcatServletWebServerFactory>```로 고쳐서 수정한다.

## AbstractJsonpResponseBodyAdvice deprecate

jsonp를 지원하기 위해 아래와 같이 쓰던 기능이 없어짐

jsonp 대신 cors(cross domain resource sharing)을 통해 통신 하도록 권장함. 

문제는 IE8, IE9에서는 cors가 부분 지원되며, jquery에서는 ie에 대한 예외 처리를 지원하지 않음

[참고사이트 ](https://crontab.tistory.com/entry/IE8-IE9%EC%97%90%EC%84%9C-crossdomain-%EC%9A%94%EC%B2%AD-%EB%AC%B8%EC%A0%9C)

```java
@ControllerAdvice
public class JsonpAdvice extends AbstractJsonpResponseBodyAdvice {
	public JsonpAdvice() {
		super("callback");
	}
}
```

위와 같은 이유로 JsonpAdvice 클레스 제거 함.

## yaml 과 @ConfigurationProperties

@ConfigurationProperties의 prefix와 @Value의 키 값은 kebab-case여야 한다.
>>> Canonical names should be kebab-case ('-' separated), lowercase alpha-numeric characters and must start with a letter 


Yaml 파일을 읽는 부분에서는 kebab-case 및 camel-case를 둘다 지원 한다.

## mysql connector 버전 변경

### jdbc driver의 package 변경

Loading class "com.mysql.jdbc.Driver" This is deprecated. The new driver class is "com.mysql.cj.jdbc.Driver"

위와 같은 이유로 com.mysql.jdbc.Driver를 com.mysql.cj.jdbc.Driver로 변경 함 

### timezone 이슈

KST 타임 존이 없어졌음

jdbc url에 serverTimezone=Asia/Seoul를 추가하여 timezone 이슈를 제거 함

[참고사이트](https://offbyone.tistory.com/318)

## default datasource pool (tomcat pool -> HikariCP)

poolsize 설정은 maximumPoolSize 설정을 하면 minimumIdle의 기본 값 동일하게 맞쳐짐.
해당 옵션 말고 나머지는 default로 써도 큰 이슈는 없어 보임

[한글 참고 사이트](https://effectivesquid.tistory.com/entry/HikariCP-%EC%84%B8%ED%8C%85%EC%8B%9C-%EC%98%B5%EC%85%98-%EC%84%A4%EB%AA%85)

[공식 사이트](https://github.com/brettwooldridge/HikariCP#configuration-knobs-baby)


hikari pool를 명시적으로 사용하는 yaml 설정이 좀 더 명확해 보이므로 해당 방법을 채택함.


# 느낀점

springboot 1.3.8 -> 2.1.6으로 업그레이드 하였을 경우 장점이 뭐가 있을까?

1. springboot가 지원 되는 버전?
2. 좀더 빠른 datasource pool
3. reactive 지원 되지만 jdbc driver에서 미지원 ㅜㅜ
4. reference 찾기 쉬운점

[참고 사이트](https://brunch.co.kr/@springboot/112)

일단 올려야지 legacy가 안될듯...아.