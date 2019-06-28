---
title: wildfly ejb 관련 memory 이슈
layout: post
date: '2019-06-26 00:46:18 +0900'
categories: wildfly ejb issue
---

# 이슈 사항
ejb server에서  old gen의 memory가 가득차서 full gc가 발생하는 이슈 

![메모리분석](/assets/post/2019-06-25-wildfly-ejb-memory/IMG_3070.JPG){: width="900" class="border"}
*[mat 화면]*

# 원인 분석 및 테스트 사항
mat로 분석 하였을때 메모리 상에 가장 큰 오브젝트는 org.jboss.remoting3.ConnectionImpl 였으며,   멤버 변수  authMap에 할당된 요소가 2,097,152개나 됨.  
이렇게 많은 요소가 어떤 메소드로 인하여 추가 되었는지 추적하기 위해  [[ConnectionImpl 소스](https://github.com/jboss-remoting/jboss-remoting/blob/master/src/main/java/org/jboss/remoting3/ConnectionImpl.java)] 에서authMap.put을 호출하는 메소드를  찾고, 이 메소드가 **receiveAuthRequest**와 **receiveAuthResponse** 임을 확인.  
그리고 다행히도 두 메소드는 log.tracef를 이용하여 로그를 남기고 있으므로 wildfly에 Trace레벨의 log 설정 함.
> 로그 카테고리 : org.jboss.remoting.remote, 로그 레벨 : TRACE

{% highlight java %}
import static org.jboss.remoting3._private.Messages.log;

public void receiveAuthRequest(final int id, final String mechName, final byte[] initialResponse) {
        log.tracef("Received authentication request for ID %08x, mech %s", id, mechName);
				...
	
void receiveAuthResponse(final int id, final byte[] response) {
        log.tracef("Received authentication response for ID %08x", id);
				...
{% endhighlight %}

설정 이후 운영 환경에서 아래와 같은 로그가 출력됨 (log를 grep 한 결과)

![서버로그](/assets/post/2019-06-25-wildfly-ejb-memory/log.png){: width="900" class="border"}
*[server.log]*

###  로그를 통해 ConnectionImpl.receiveAuthRequest에 지속적인 인증 요청이 들어옴을 확인

그럼 "왜 인증 요청이 지속적으로 들어올까?"라는 의문이 생겼고, 나름 합리적이라는 가정을 세우고 이슈 처리를 위한 테스트를 진행 함.

## 가정1
처음에는  '인증을  ejb 서버에서 저장하는데 왜 인증 요청이 많지? 매번 다른 인증 요청을 하는게 혹시 라우팅 시 기존에 접속 했던 서버가 아닌 다른 서버에 인증을 요청 하나' 라고 생각 함.

**삽집1**  일단 개발 환경에서 ejb client와 ejb server 2:3으로 구성 하고 web 서버 라우팅을 1:1, 2:1, 2:3으로  테스트 하였으나 "Reviced authentication ~~ id" 로그가 남지 않음을 확인.

**삽질2** '재기동시 인증 이슈가 있을까?'라고 생각을 들어 ab를 통해 지속적으로 ejb 호출 한 후 개발서버에 위치한 ejb client 쪽 및 ejb server 쪽을 stop, start, stop, start ... 하였지만 간헐적으로 "Reviced authentication ~~ id" 로그가 발생 하였을 뿐 재기동이 직접적인 원인으로 보이지 않음.

## 가정2
몇일 동안 **왜? 언제? 원인이 뭘까**라는 고민하고 원인에 대해 검색을 하다가, 다시 자세히 로그를 보니  "mech JBOSS-LOCAL-USER" 문자열이 눈에 띄어서 이에 대한 조사를 하였고, 검색을 통해 다음과 같은 내용을 확인함.
> **JBOSS-LOCAL-USER**는 동일 머신에서 조용하게 인증을 하며  id/password 없이 파일 기반으로 인증하는 메카니즘

[참고 사이트](https://docs.jboss.org/author/display/EJBCLIENT/Overview+of+Client+properties)
```
JBOSS_LOCAL_USER;  means the silent authentication mechanism used when the client and server are on the same machine, is disabled.
```

'**삽질1, 삽질2** 에 대한 테스트는 동일 머신이라서 "Reviced authentication ~~ id" 로그가 안남았구나'를 깨달았고, ejb client와 ejb server가 분리된 환경에서 log를 하니 2분 마다 "Reviced authentication ~~ id" 로그 출력됨을 확인함.

### 즉 같은 머신에 ejb client와 server가 위치하고 있으면, ejb server 의 파일 시스템에 저장된 내용을 ejb client가 위치한 server에서 읽을 수 없으므로 인증은 실패되고 client는 지속적으로 ejb client는 server에게 인증을 요청 하는것이라는 결론을 내림.


**삽질3** JBOSS-LOCAL-USER에 대한 메카니즘을 제거하기 위해,  ejb client 설정을 아래와 같이 고쳐서 테스트 진행.

1. ejb 통신시 사용되는  outbound connection의 SASL_DISALLOWED_MECHANISMS 에 JBOSS-LOCAL-USER 설정 (실패)  [[참고 사이트1](https://access.redhat.com/documentation/en-us/red_hat_jboss_enterprise_application_platform/7.0/html/configuration_guide/reference_material#remoting_attributes)], [[참고 사이트2](https://docs.jboss.org/xnio/3.0/api/org/xnio/Options.html)]
2. ejb 통신시 사용되는  outbound connection의 	SASL_MECHANISMS를 DIGEST-MD5만 설정 (실패)
3. ejb client에서 사용하는 security Realm은 따로 있으나 혹시 전역적으로 사용되는 무엇가 있지 않을까 해서 ApplicationRealm에 local 제거 (실패) [[참고 사이트](https://access.redhat.com/documentation/en-us/jboss_enterprise_application_platform/6/html/administration_and_configuration_guide/remove_silent_authentication_from_the_default_security_realm)]
4. standalone에 보이는 JBOSS-LOCAL-USER를 주석으로 막아볼까 (실패)

위와 같은 설정 실패 한 후  JBOSS-LOCAL-USER disable 방법을 검색하던 중 JBOSS-LOCAL-USER가 동작하는 방식에 대한 설명하는 글을 발견함.

[참고 사이트](https://developer.jboss.org/wiki/AS710Beta1-SecurityEnabledByDefault)
```
For this scenario we have added a new SASL mechanism that is used internally by our client APIs, when this mechanism is selected the following sequence is followed to authenticate the user: -

1. Client sends message to server saying "I want to use the JBoss Local User SASL mechanism"
2. Server generates one time token, writes it to a unique file and sends message to client with full path of the file.
3. Client reads the token from the file and sends it to the server.
4. Server verifies token received and deletes the file.
```

즉 'client는 JBOSS-LOCAL-USER 인증이 가능한지만 물어보고,  server가 가능하다 라고 판단되면 파일을 생성하여 응답한다' 라고 하면 server쪽에서 JBOSS-LOCAL-USER 메카니즘은 안된다고 설정 하면 되는 구나' 라고 생각하여, 서버쪽에 사용 인증을 제거 하니 로그가 더 이상 남지 않음을 확인함. 
[[참고 사이트](https://access.redhat.com/documentation/en-us/jboss_enterprise_application_platform/6/html/administration_and_configuration_guide/remove_silent_authentication_from_the_default_security_realm)]


끝.
# 느낀점
 1. wildfly 설정은 어렵다. 
 2. 설정이 어려운 ejb는 되도록 쓰지 말자. 
 3. 로그는 자세히 봐야 한다.
 3. 검색 문서는 자세히 봐야 한다.
 4. 2주 고생하고 설정 한줄 변경이라...ㅜㅜ
 5. 난 글을 참 못 쓰는 구나.
