---
title: maven archetype 만들기
layout: post
date: '2019-06-28 13:32:44'
categories: maven archetype
---

maven archetype을 이용하여 프로젝트를 생성하면 일관된 어플리케이션 구성을 할 수 있는 장점이 있습니다.

이런 maven archetype 생성 방법은 아래의 두가지 방법이 있습니다.
* archetype  프로젝트를 이용하여 작성하는 방법
* 기존 프로젝트를 archetype으로 생성하는 방법

[참고사이트](https://maven.apache.org/archetype/maven-archetype-plugin/index.html)

여기서 소개하고자 하는 방법은 기존 프로젝트로 부터 archetype을 생성하는 방법이며, 해당 방법을 선택한 이유는 기존 프로젝트를 이용한 "어플리케이션 테스트"가 용이 하기 때문 입니다.


## 버전 확인 

```bash
$ mvn --version
Apache Maven 3.6.0 (97c98ec64a1fdfee7767ce5ffb20918da4f719f3; 2018-10-25T03:41:47+09:00)
Maven home: /usr/local/Cellar/maven/3.6.0/libexec
Java version: 1.8.0_172, vendor: Oracle Corporation, runtime: /Library/Java/JavaVirtualMachines/jdk1.8.0_172.jdk/Contents/Home/jre
Default locale: ko_KR, platform encoding: UTF-8
OS name: "mac os x", version: "10.14.5", arch: "x86_64", family: "mac"
``` 
[*maven 버전*]
```bash
$ mvn -Dplugin=archetype help:describe

...

Name: Maven Archetype Plugin
Description: Maven Archetype is a set of tools to deal with archetypes, i.e.
  an abstract representation of a kind of project that can be instantiated into
  a concrete customized Maven project. An archetype knows which files will be
  part of the instantiated project and which properties to fill to properly
  customize the project.
Group Id: org.apache.maven.plugins
Artifact Id: maven-archetype-plugin
Version: 3.1.1
Goal Prefix: archetype

...

``` 
[*archetype plugin 버전*]

[참고 : archetype plugin release page](https://github.com/apache/maven-archetype/releases)

>  "-Dplugin=archetype help:describe"를 수행하니 신규 버전을 설치하는 듯  ㅡㅡ;;

## 기존 프로젝트 복제 및 불필요 파일 삭제 
기존 프로젝트를 특정 경로에 복사 후  eclipse project 관련 파일 삭제를 삭제 합니다. 

해당 작업을 하지 않을 경우 추후에 나오는 archetype-metadata.xml 설정이 지저분 하게 됩니다.

```bash
cp -r <project_path> for-archetype
cd for-archetype
	
#불필요한 파일 제거 
find . -type f -name ".project" -exec rm -rf {} +
find . -type d -name ".settings" -exec rm -rf {} +
find . -type d -name ".apt_generated" -exec rm -rf {} +
find . -type f -name ".classpath" -exec rm -rf {} +
find . -type f -name ".factorypath" -exec rm -rf {} +
find . -type d -name "target" -exec rm -rf {} +
 ```
		
## archetype 생성  		

복사한 경로에서 maven 명령어로 archetype 프로젝트를 생성 합니다.

```bash
$mvn clean archetype:create-from-project
```

위와 같이 수행 하면 archetype는 target 디렉토리 (path : ./target/generated-sources/archetype)에 생성 됩니다.

## archetype 설정 확인 및 수정
기존 프로젝트 기반으로 archetype을 설정 할 경우 아쉽게도 완벽하게 설정이 완료 되지 않습니다.

archtype의 설정 파일 archetype-metadata.xml (위치 : ./target/generated-sources/archetype/src/main/resources/META-INF/maven/archetype-metadata.xml) 통해 구성을 재 설정 합니다.

> [참고] maven archetype은 archetype-metadata.xml의 내용으로 velocity template를 활용하여 프로젝트를 생성합니다.

> > The maven-archetype-plugin uses Velocity to expand template files, and this documentation talks about 'Velocity Properties', which are values substituted into Velocity templates. ... All the files under one specified Java (or cognate) package are relocated to a project that the user chooses when generating a project.

xml를 확인해 보면 변수 "${rootArtifactId}(파일 내)" 와 ```__rootArtifactId__(경로일 때 )``` 가 보이는데 이건 archetype에서 프로젝트 생성시 사용자에 의해 입력되는 값이며, 어플리케이션의 이름 부분과 매칭됩니다.

변수 추가시  아래의 설정을 archetype-metadata.xml 에 추가 하면 됩니다.
```xml		
<requiredProperties>
    <!-- URL 에서만 사용 -->
    <requiredProperty key="BlarBlar">
    <validationRegex><![CDATA[[a-z]+]]></validationRegex>
    </requiredProperty>
</requiredProperties>
```
> archetype 빌드 시 test 수행 때 추가 변수가 없어서 에러가 발생 하면 "target/generated-sources/archetype/src/test/resources/projects/basic/archetype.properties"에 변수 명과 default 값 설정 하면 됩니다.

yml를 template에 적용이 필요하면 "fileSet filtered='true'" 내에 include에 추가 하면 됨 (directory 경로 확인)
```xml
<fileSet filtered="true" encoding="UTF-8">
    <directory>src/main/resources</directory>
    <includes>
        <include>**/*.yml</include>
    </includes>
</fileSet>
```


## 파일 수정 
pom.xml, application.yml, application.properties를 전반적으로 확인이 필요하며, 필요할 경우 특정 문자열을 변수로 대체 하여야 합니다.

```bash
find . -name "pom.xml" -exec vim {} + 
find . -name "*.yml" -exec vim {} +
find . -name "*.properties" -exec vim {} +
find . -name "*Mapper.xml" -exec vim {} + #mybatis mapper xml
```

[참고(케바케)] pom.xml에서 확인한 부분 
* parent 부분
* build > plugins > plugin(spring-boot-maven-plugin) > executions > execution > configuration > mainClass

[참고(케바케)] ``*.yml`` 에서 확인한 부분 
* url 
* key 설정 부분

[참고(케바케)] ``*.java`` 에서 확인한 부분 
* 변경된 yml 속성 중 @Value 된 부분


## archetype 생성

/target/generated-sources/archetype 경로에서 maven install 수행

 ```bash
 mvn clean install
 
 [INFO] Scanning for projects...
[INFO]
[INFO] ---------------< < 기반 프로젝트 package>:<기반 프로젝트 artifactId>-archetype >----------------
[INFO] Building <기반 프로젝트 artifactId>-archetype 0.0.1-SNAPSHOT
[INFO] --------------------------[ maven-archetype ]---------------------------
[INFO]
[INFO] --- maven-clean-plugin:2.5:clean (default-clean) @ <기반 프로젝트 artifactId>-archetype ---
[INFO] Deleting /Users/mspark/git/for-archetype/target/generated-sources/archetype/target
[INFO]
[INFO] --- maven-resources-plugin:3.1.0:resources (default-resources) @ <기반 프로젝트 artifactId>-archetype ---
[WARNING] Using platform encoding (UTF-8 actually) to copy filtered resources, i.e. build is platform dependent!
[INFO] Copying 1106 resources
[INFO]
[INFO] --- maven-resources-plugin:3.1.0:testResources (default-testResources) @ <기반 프로젝트 artifactId>-archetype ---
[WARNING] Using platform encoding (UTF-8 actually) to copy filtered resources, i.e. build is platform dependent!
[INFO] Copying 2 resources
[INFO]
[INFO] --- maven-archetype-plugin:3.1.1:jar (default-jar) @ <기반 프로젝트 artifactId>-archetype ---
[INFO] Building archetype jar: /Users/mspark/git/for-archetype/target/generated-sources/archetype/target/<기반 프로젝트 artifactId>-archetype-0.0.1-SNAPSHOT
[INFO]
[INFO] --- maven-archetype-plugin:3.1.1:integration-test (default-integration-test) @ <기반 프로젝트 artifactId>-archetype ---
[INFO] Processing Archetype IT project: basic
[INFO] ----------------------------------------------------------------------------
[INFO] Using following parameters for creating project from Archetype: <기반 프로젝트 artifactId>-archetype:0.0.1-SNAPSHOT
[INFO] ----------------------------------------------------------------------------
....
 ```
 
 maven install을 통해 local repository에 빌드된 결과물을 저장
 
##  archetype으로 프로젝트 생성 (테스트)
 
 특정 경로를 생성하여 아래의 mavne 명령어 수행 

```bash
mvn archetype:generate -DarchetypeGroupId=<기반 프로젝트 package> -DarchetypeArtifactId=<기반 프로젝트 artifactId>-archetype -DarchetypeVersion=0.0.1-SNAPSHOT
```

elipse를 통해 import existing maven projects 수행 한 후 test case 수행
실패 시 "archetype 설정 확인 및 수정" 부터 다시 확인

## archetype nexus에 배포 하기
release 로 배포 하기 위해서 pom.xml(.//target/generated-sources/archetype/pom.xml) 의 version 정보에 -SNAPSHOT을 제거 후 배포 한다.

```bash
mvn clean deploy
```