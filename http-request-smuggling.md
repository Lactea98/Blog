# HTTP Request Smuggling

---

### Abstract

이 글을 쓴 계기는 [hackerone](https://www.hakcerone.com) 에서 paypal 취약점이 나왔는데, 이 취약점의 이름은 **http request smuggling** 이라고 한다.

바로 이 글이다. [링크](https://hackerone.com/reports/510152) 

이 취약점은 처음 들어봤고, 어떤 공격이 가능한지 시간이 날때마다 원글을 읽고 다른 사이트로 부터 정보를 얻은 후,
꽤나 흥미로운 취약점인 것 같아서 정리를 목적으로 작성하게 되었다. 이 취약점은 꽤나 오래 전 부터 발견된 취약점 이라고 한다. 이 취약점을 이용하여 공격을
성공적으로 수행하기 위해서는 다음과 같은 조건이 필요하다.
메인 server 앞에 proxy server가 있고, 이 두 server는 request의 header에 있는 content-length와 transfer-encoding 이 있을 경우 ,
서로 다른 parsing으로 문제가 발생하게 된다. 현재 real world 에서는 메인 server의 과부하, 보안 등의 목적으로 앞에 proxy server를 두는 경우가 있다.
이 취약점을 찾은 보안 전문가는 아마 이런 이유로 **http request smuggling** 공격을 시도 하지 않았나 하는 생각이 든다. (그냥 존경스럽다.)

어쨋든 이 이후의 글은 이 **http request smuggling** 공격에 대해 설명을 작성할 것인데, 실제로 나는 테스트를 해보지도 않았고, 단지 몇몇개의 글만 읽고
머리 속으로 정리한 거라 틀린 부분도 있을 것이고 내용이 부족할 수 도 있을 것이다. 이런 점은 양해를 바란다. 만약 이 글보다 더 좋은 글을 보고 싶을 경우
Reference를 참고하자.



### Concepts




HTTP/1.1 은 여러개의 HTTP 요청을 보낼 수 있게 지원한다. 이 프로토콜은 HTTP 요청이 일련(연속적)으로 배치되어 서버로 전송된다. 서버는 이 일련의
패킷을 받아 응답을 하기 위해 parsing을 하게되는데, 각각의 패킷이 어디가 끝 부분이고 다음 패킷의 시작부분이 어디인지 헤더를 분석한다. 

![image](https://user-images.githubusercontent.com/38517436/63819813-bc61c900-c981-11e9-9bd8-b03dc34bbb9e.png)
출처: portswigger.net

위 사진처럼 각각의 요청이 일련으로 배치되어 서버에 전송이 되고, 서버는 각각의 시작과 끝을 분석하여 패킷을 분리시키고 각각의 request 패킷에 response을
하게 된다. 만약 공격자가 아래 사진처럼 정상적인 패킷에 악의적인 패킷을 포함시켜 request를 보내면 어떻게 될까?
![image](https://user-images.githubusercontent.com/38517436/63820017-9c7ed500-c982-11e9-8c76-1a9e3a855143.png)

Front-end 에서 위 사진처럼 request 패킷을 받으면 각각의 패킷의 시작과 끝을 찾게 되는데, 분석을 하는 기준 때문에 공격자가 보낸 파란색 요청과 주황색 요청이
분리되어 Back-end 서버로 전송된다. Back-end 는 파란색 요청만 응답하고 주황색 요청은 서버측에 posioning 되어 대기하다가 초록색 요청이 들어보면 주황색
요청과 초록색 요청을 함쳐 요청을 처리하고 응답을 보내게 된다. 이렇게 HTTP request Smuggling 공격에 대해 간단히 설명하면 이런 것이다.

여기서 궁금한 점은 Front-end는 공격자가 보낸 요청을 왜 파란색과 주황색 요청으로 따로 분리했는지와, Back-end 에서 posioning 이 일어난 이유와 
피해자는 어떤 응답을 받게 되는지 이다. 차근차근 설명해 주겠다.

는 밥머그러 가야지


### Reference

1. [https://portswigger.net/blog/http-desync-attacks-request-smuggling-reborn](https://portswigger.net/blog/http-desync-attacks-request-smuggling-reborn)
2. [sungwanjo.tistory.com/entry/Chunked-Response란](sungwanjo.tistory.com/entry/Chunked-Response란)
3. [hahwul.com/2019/08/http-smuggling-attack-re-born.html](hahwul.com/2019/08/http-smuggling-attack-re-born.html)
4. [https://portswigger.net/web-security/request-smuggling](https://portswigger.net/web-security/request-smuggling)
