---
title: maven archetype 만들기
layout: post
date: '2019-06-28 13:32:44'
categories: maven archetype
---

maven archetype을 이용하면 프로젝트를 생성하면 일관화된 공통 모듈 및 어플리케이션 설정을 할 수 있는 장점이 있습니다.

먼저 archetype은  archetype  프로젝트를 이용하여 작성하는 방법과 기존 프로젝트를 archetype으로 생성하는 두 가지 방법이 있습니다. 
[참고사이트](https://maven.apache.org/archetype/maven-archetype-plugin/index.html)

여기서 소개하고자 하는 방법은 기존 프로젝트로 부터 archetype을 생성하는 방법입니다.

일단 archetype 설정에 앞서 maven 및 archetype plugin의 version을 확인 해 보겠습니다.


```bash
$ mvn --version
Apache Maven 3.6.0 (97c98ec64a1fdfee7767ce5ffb20918da4f719f3; 2018-10-25T03:41:47+09:00)
Maven home: /usr/local/Cellar/maven/3.6.0/libexec
Java version: 1.8.0_172, vendor: Oracle Corporation, runtime: /Library/Java/JavaVirtualMachines/jdk1.8.0_172.jdk/Contents/Home/jre
Default locale: ko_KR, platform encoding: UTF-8
OS name: "mac os x", version: "10.14.5", arch: "x86_64", family: "mac"
```
*maven 버전*

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
*archetype plugin 버전*

[참고 : archetype plugin release page](https://github.com/apache/maven-archetype/releases)

> 근데 "-Dplugin=archetype help:describe"를 수행하니 신규 버전을 설치하는 듯 합니다. ㅡㅡ;;

일단 mvn 명령어 수행에 앞서 아래와 같이 환경을 구성 하겠습니다.

1. 기존 프로젝트를 특정 경로에 복사 
2. eclipse project 관련 파일 삭제
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
3. archetype 생성  
   ```bash
	 $mvn clean archetype:create-from-project
	 ```
	 * 위와 같이 수행 하면 archetype는 target path 하위에 생성됨
	 * archtype의 설정는  './for-archetype/target/generated-sources/archetype/src/main/resources/META-INF/maven/archetype-metadata.xml' 에 설정 되어 있음
		 *  archetype-metadata.xml는 archetype의 뼈대가 되는 파일로 대상 파일과 변수를 통하여 프로젝트 명이 어떻게 확장될지를 설정할 수 있음
		 *  만일 archetype을 통해 프로젝트 생성 시 입력 받고자 하는 값이 있다면 아래와 같이 설정을 추가 하면 아래와 같이 설정하면 됨
    ```xml		
		<requiredProperties>
        <!-- URL 에서만 사용 -->
        <requiredProperty key="serviceDomainName">
            <validationRegex><![CDATA[[a-z]+]]></validationRegex>
        </requiredProperty>
        <requiredProperty key="serviceName">
            <defaultValue>${artifactId}</defaultValue>
        </requiredProperty>
    </requiredProperties>
		
    ```



: