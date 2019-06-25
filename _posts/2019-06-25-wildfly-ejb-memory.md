---
layout: post
title:  "wildfly ejb memory 이슈"
date:   2019-06-26 00:46:18 +0900
categories: jekyll update
---
# 이슈 사항
ejb 서버에서  old gen의 memory가 가득차서 full gc가 발생하는 이슈 


![메모리분석](/assets/post/2019-06-25-wildfly-ejb-memory/IMG_3070.JPG){: width="900"}

*heapdump를 mat에서 확인한 화면*


## 원인 분석
소스의 위치는 org.jboss.remoting3.ConnectionImpl 이고 해당 객체의 멤버 변수인 authMap의 요소가 2,097,152개가 할당되어 상황임.

[github 내 해당 class file link ](https://github.com/jboss-remoting/jboss-remoting/blob/master/src/main/java/org/jboss/remoting3/ConnectionImpl.java)

해당 class에서 authMap.put를 해주는 메소드를 찾아보면 **receiveAuthRequest**와 **receiveAuthResponse** 임

어떤 메소드인지 파악은 어려운 상태에서 임.

다행이도 메소드의 내에 log.tracef를 통해 로그를 남기고 있으니 로그를 출력하여 요청이 되는 메소드를 찾기로 함


{% highlight java %}
import static org.jboss.remoting3._private.Messages.log;

public void receiveAuthRequest(final int id, final String mechName, final byte[] initialResponse) {
        log.tracef("Received authentication request for ID %08x, mech %s", id, mechName);
				...
	
void receiveAuthResponse(final int id, final byte[] response) {
        log.tracef("Received authentication response for ID %08x", id);
				...
{% endhighlight %}
