# HTTP Request Smuggling

---

### Abstract

이 글을 쓴 계기는 [hackerone](https://www.hakcerone.com) 에서 paypal 취약점이 나왔는데, 이 취약점의 이름은 **http request smuggling** 이라고 한다.

바로 이 글이다. [링크](https://hackerone.com/reports/510152) 

이 취약점은 처음 들어봤고, 어떤 공격이 가능한지 시간이 날때마다 원글을 읽고 다른 사이트로 부터 참고하여 공부를 했다.
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

### 1. Why split a request into two requests

  이 이유를 알기 위해서는 Content-length 와 Transfer-encodeing 헤더에 대해 알고 있어야 한다.

  - Content-length

    Content-length는 entity body의 크기를 bytes 숫자로 나타낸 것이다. 예를 들어 request나 response를 받을 때 헤더에 content-length 값이 5이면 5byes 만큼만 entity body에서 가져오게 된다. 만약 content-length 값이 10bytes 인데 entity body에서 5bytes 값만 있다면, 받는 입장에서는 "엇? 데이터가 덜 들어왔으니까 기다려야지" 라는 행동을 할 것이다. 기다라려도 요청이 추가적으로 들어오지 않으면 timeout이 된다.

  - Transfer-encoding
    
    여러가지의 Tansfer-encoding 방식이 있는데 여기서 다룰 encoding은 Chunked 이다.
  
    ```
    Transfer-Encoding: chunked
    Transfer-Encoding: compress
    Transfer-Encoding: deflate
    Transfer-Encoding: gzip
    Transfer-Encoding: identity
    ```

    HTTP/1.1 에서 데이터 전송 메커니즘 중 하나로, 덩어리(Chunk)의 나열로 데이터를 전송한다.
    이를 사용하는 이유는 content-length를 모를때 Transfer-encoding을 사용하여 중간중간에 계속 데이터를 전송 할 수 있게 된다. 더이상 데이터를 줄 것이 없다면 마지막에 0 \r\n 을 보내게 된다.

    Chunked Transfer-encoding 형태는 다음과 같다.

    ```
    [크기 16진수] \r\n
    [data]\r\n
    ```

    형태는 위와 같고 실제 패킷에는 다음과 같이 찍히게 된다.

    ```
    4\r\n
    Wiki\r\n
    5\r\n
    pedia\r\n
    e\r\n
     in\r\n\r\nchunks.\r\n
    0\r\n
    \r\n
    ```

    위 내용을 decode 하면 아래와 같다.

    ```
    Wikipedia in 
    chunks.
    ```

  - Simple example

    위에 설명했던 Content-length와 Transfer-encoding을 이용하여 간단한 예를 보여주겠다.

    ![image](https://user-images.githubusercontent.com/38517436/63820017-9c7ed500-c982-11e9-8c76-1a9e3a855143.png)

    우선 위에서 봤던 사진을 다시 가져와서 victim 에게 어떤 영향을 미치는지 설명하겠다.

    ![image](https://user-images.githubusercontent.com/38517436/63986113-894c4080-cb0d-11e9-9eb9-a27211aecba3.png)

    여기서 가정을 하겠다. Front-end는 첫번째 Content-length 를 기준으로 패킷을 분석하고, Back-end는 두번째 Content-length를 기준으로 패킷을 분석한다고 가정을 하자. 공격자는 파란색과 주황색 패킷을 보냈다. 파란색 패킷에는 위에 사진처럼 2개의 Content-length 헤더를 가지고 있다.

    - 처리 과정
      - 공격자가 보낸 패킷을 분석하기 위해 첫번째 Content-length를 기준으로 분석한다. 첫번째 Content-length = 6 이므로 entity body는 \r\n 이후에 있는 값 부터 bytes를 세어 12345G 까지 읽고 Back-end로 보낸다. 
      - 패킷을 받은 Back-end는 두번째 Content-length를 기준으로 분석한다. 두번째 Content-length = 5 이므로 12345 까지 데이터를 읽어 그에 맞는 응답을 하게 된다(!).
      - 이렇게 되면 Back-end는 G라는 데이터가 남게 되고 이 값을 처리하기 위해 기다린다.
      - 이때 피해자가 요청을 보내게 된다.(초록색 패킷)
      - Back-end까지 온 요청을 처리하는 과정에서 기존에 남아있던 G 데이터가 피해자의 요청 앞에 붙어 POST 요청이 아니라 GPOST 요청이라는 이상한 요청이 들어오게 된다. 결국엔 피해자는 정상적인 응답을 받지 못하게 된다.
      
    이러한 서버 설정은 CL.CL 이라고 부른다. 

    실제로 2개의 Content-length 기술은 거의 동작하지 않는데, 이유는 많은 시스템은 여러개의 Content-length 헤더 요청을 거절하기 때문이다. 대신에, Chunked encoding을 사용하여 시스템을 공격할 수 있다. RFC 2616 에 따르면,

    > If a message is received with both a Transfer-Encoding header field and a Content-Length header field, the latter MUST be ignored.

    위 문장을 해석하면 "만약 Transfer-encoding 헤더 필드와 Content-length 헤더 필드와 함께 메시지를 받게 된다면, 후자는 무시해야 한다." 라고 해석이 된다. 이 말은 암시적으로 Transfer-Encoding : chunked 와 Content-length 모두 사용하여 서버 설정상 요청을 처리 할 수 도 있다는 뜻이다. 이러한 요청을 거절하는 서버는 거의 없다고 한다. 

    이번 예제는 Content-length 와 Transfer-encoding을 이용한 간단한 예제를 보여주겠다.
    
    이번 서버 설정은 CL.TE 방식이다.
    ![image](https://user-images.githubusercontent.com/38517436/63986950-28266c00-cb11-11e9-9255-c15b28ea5af3.png)
    
    우선 Front-end는 Content-length = 6 이므로 \r\n 이후에 있는 값 0\r\n\r\nG 을 데이터로 인식해서 처리한 후 Back-end로 보내게 된다. 
    Back-end는 Transfer-encoding : chunked 를 기준으로 데이터를 처리하게 되는데, 위에서 Transfer-encoding : chunked에 대한 설명을 되새겨 보면 (기억 안나면 위로 ㄱㄱ) 데이터의 끝은 0\r\n으로 나타낸다고 설명했다. 따라서 위 패킷에는 0\r\n\r\nG 데이터에서 0\r\n 를 보고 "여기까지가 데이터가 끝이네" 라고 생각하며 여기까지만 응답을 하게 된다. 남은 \r\nG는 아까 첫번째 예제와 똑같이 피해자가 요청이 들어오면 피해자 패킷 앞에 붙어 응답을 하게된다. 
    
    이 예는 Front-end가 Chunked encoding을 지원하지 않는 시스템에 해당되는 익스플로잇 이다. 이는 콘텐츠 전송 네트워크 Akamai를 사용하는 많은 웹 사이트에서 볼 수 있는 동작이라고 한다.
    
    반대로 Front-end가 chunked encoding을 지원하고, Back-end가 Content-length를 지원한다면, 즉 TE.CL 일 경우 다음과 같이 요청을 보내야 한다.
    
    ![image](https://user-images.githubusercontent.com/38517436/63987344-c2d37a80-cb12-11e9-8671-b764ea14cacf.png)
    
    Front-end는 Transfer-encoding을 기준으로 패킷을 분석한다. 위에서 설명 했듯이 6\r\n prefix\r\n 0\r\n 까지가 body의 끝이고 이것을 Back-end로 보낸다. Back-end는 Content-length = 3 이므로 6\r\n 까지 데이터로 인식하여 응답을 하게 된다. 이 이후로는 남은 데이터는 피해자의 요청 앞에 붙어 응답을 보내게 된다.
    
    여기까지 글을 읽으면서 이해가 된 분들은 나처럼 소름이 돋았을 것이다.(두근두근...)
    
    필자가 참고한 사이트에서 Burp Suite에 HTTP Request Smuggler 라는 extension 이 있다고 한다. 궁금한 분들은 [링크](https://github.com/PortSwigger/http-request-smuggler) 클릭!
    
    밥 머그러 감.
    







### Reference

1. [https://portswigger.net/blog/http-desync-attacks-request-smuggling-reborn](https://portswigger.net/blog/http-desync-attacks-request-smuggling-reborn)
2. [sungwanjo.tistory.com/entry/Chunked-Response란](sungwanjo.tistory.com/entry/Chunked-Response란)
3. [hahwul.com/2019/08/http-smuggling-attack-re-born.html](hahwul.com/2019/08/http-smuggling-attack-re-born.html)
4. [https://portswigger.net/web-security/request-smuggling](https://portswigger.net/web-security/request-smuggling)
5. [https://en.wikipedia.org/wiki/Chunked_transfer_encoding](https://en.wikipedia.org/wiki/Chunked_transfer_encoding)
