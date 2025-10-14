# 📚 목차

1. 🚨 [HTTP/1.1을 쓰고 있어요](#http11을-쓰고-있어요)
2. 🤔 [왜 HTTP/2로 바꿔야 하나요?](#왜-http2로-바꿔야-하나요)
   - HTTP/1.1
   - HTTP/2
3. ✅ [진짜 그런지 확인하기](#진짜-그런지-확인하기)
   - chrome-net-export로 로그 파일 생성하기
   - 로그 파일 분석하기
4. ‼️ [진짜 소켓이 6개인지 쉽게 확인하기](#진짜-소켓이-6개인지-쉽게-확인하기)
5. 📣 [어떻게 HTTP/2로 보낼 수 있도록 요청하나요?](#어떻게-http2로-보낼-수-있도록-요청하나요)
6. 💡 [번외) 왜 prod 서버에서는 HTTP/2가?](#번외-왜-prod-서버에서는-http2가)
   - prod와 dev의 차이
   - ALB란?
   - ALB 규칙을 보았더니

  
​

# 🚨 HTTP/1.1을 쓰고 있어요

​
포게더 성능을 개선하기 위해서 먼저 LightHouse를 측정해보았습니다.  
​
그런데, HTTP/2를 통해 게재되지 않고 있다는 경고와,
무려 22910ms의 엄청난 Load Delay가 발생하고 있었습니다.

​
포게더에서는 이미지 리소스에 대한 S3경로를 api 응답으로 받고,
클라이언트가 직접 S3에 접근하여 이미지를 들고오는 구조였습니다.

구조도를 그리자면 아래와 같습니다.

<img width="773" height="441" alt="기존 이미지 리소스 요청 방식" src="https://github.com/user-attachments/assets/8662c9d1-cdc7-4ea3-b0bd-3064a8ef25e4" />
  
​

가추를 해보았을 때, http/1.1의 특징상, 앞 쪽에서 병목현상이 생겼을 때,
순수 이미지 다운로드가 늦어진다고 생각했습니다.  
​  
그렇기에, **http2로 개선하면 어느정도 개선이 될 것**이라고 생각했습니다.
​
<p align="center">
  <img src="https://github.com/user-attachments/assets/04d1c770-2d95-4de9-8154-a46801ec3512" width="49%"/>
  <img src="https://github.com/user-attachments/assets/28be1276-fabd-4c2f-8ee8-5559e0483d44" width="49%"/>
</p>

​  
실제로, 네트워크 탭을 열고 프로토콜을 확인해보니 아래와 같이, http/1.1로 요청이 보내지고 있었습니다.  
그리고, waterfall을 확인해 보니, **http/1.1의 특징에 따라 이미지들을 순차적으로 가져오고** 있었습니다.
​
<p align="center">
  <img alt="각각의 요청이 http/1.1로 되고 있는 모습" src="https://github.com/user-attachments/assets/75cfcc19-b54c-4618-b2f7-1de1227846d2" width="49%"/>
  <img alt="이미지들의 요청 순서" src="https://github.com/user-attachments/assets/f442fc93-2cb9-478c-934b-54a36b5db821" width="49%"/>
</p>

<img width="773" height="222" alt="전체 리소스 로드" src="https://github.com/user-attachments/assets/9f4542fd-8040-4220-bb76-3172da14b388" />
​
​

# 🤔 왜 http/2로 바꿔야 하나요?

​
이를 설명하기 위해서는 http/1.1과 http/2의 특징에 대해서 분석해볼 필요가 있습니다.  
​

> HTTP/1.1

http/1.1은 기본적으로 한 Connection 당 하나의 요청을 처리할 수 있도록 설계되어있습니다.

하지만, HTTP/1.0 이후에 keep-alive를 통해 Connection을 재사용할 수 있게 되었습니다.
​  
하지만, HTTP/1.1은 동시 전송을 지원하지 않기 때문에,
요청과 응답이 순차적입니다.  
​  
따라서, 많은 리소스를 요청한 경우에는
개수에 비례하여 Latency가 길어집니다.

따라서, HOL(Head Of Line) Blocking 즉, `특정 응답 지연 문제`가 발생합니다.  
​  
첫 번째 요청에 대한 응답이 지연되는 경우,
그 뒤의 요청들도 영향을 받아서 지연이 되는 것입니다.

<img height="400" alt="image" src="https://github.com/user-attachments/assets/5b3879d9-9663-45b5-b21f-b31d35be039d" />

​

또 다른 문제 역시 존재합니다.  
대부분의 **모던 브라우저는 도메인당 6개의 동시 HTTP/1.1 연결을 허용** (Chrome, Firefox, Safari 등) 합니다.  
​  
따라서, 포게더에서 사용자가 100장을 이미지를 요청했다면,  
총 6개의 TCP 연결을 100장이 순차적으로 돌려가며 받아야 하는 것이죠.  
​  
**커넥션 제한과 HOL Blocking이 동시에 이루어졌다면,  
그만큼 브라우저에서는 리소스에 대한 응답이 늦게 받을 것이고,  
이는 좋지 않은 UX로 연결됩니다.  **
​

> HTTP/2

위의 문제를 HTTP/2은 Multiplexed Stream 방식으로 해결합니다.  
한 커넥션으로 동시에 여러 개의 메세지를 주고 받을 수 있고,  
응답은 순서에 상관없이 stream으로 주고 받을 수 있는 것입니다.  
​  
즉, 첫 번째 요청에 대한 응답이 지연되더라도  
뒤의 요청들이 영향을 받지 않게 됩니다.  
​  
아래와 같이, 첫번째 요청의 응답 지연과 상관 없이,  
두번째, 세번째 요청에 대한 응답이 온 것을 확인할 수 있습니다.

​  
​  
​<img height="400" alt="image" src="https://github.com/user-attachments/assets/9175994a-da38-46b0-8fc9-addc42c4001d" />

​
​
​

# ✅ 진짜 그런지 확인하기

​
이게 이론상 이야기인지, 실제로 포게더에도 적용되고 있는지 확인이 필요했습니다.  
심증은 있는데 직접 확인을 하여 문제 진단이 필요했습니다.  
​
정말 위와 같은 문제가 발생하는지 확인해보기 위해서는  
직접 로그를 찍어봐야 합니다.  
​
크롬 개발자 도구에서 Network 패널에서 Header를 확인해보니,  
http 버전만 확인이 가능하고 구체적인 connection id까지는 보여주지 않더라고요.  
​
그래서 구체적인 connection id 를 찍어보고, 확인하기 위해서는 다른 툴을 사용해야합니다.  
​
WireShark와 같은 패킷 분석 툴을 사용할 수도 있지만,  
설치를 해야하는 약간의 귀찮음 때문에 저는 chrome-net-export를 사용하도록 하겠습니다.

> chrome-net-export로 로그 파일 생성하기

먼저 chrome://net-export/ 로 접속합니다.  
그리고 원하는 옵션을 적절히 선택해준 후, 로깅을 시작하고 멈추면 됩니다.  

<img width="386" height="185" alt="chrome-net-export 로깅 화면" src="https://github.com/user-attachments/assets/762c8078-6372-4d9f-ad47-4a5ac778503b" />

​<img width="386" height="219" alt="chrome-net-export 로깅 화면" src="https://github.com/user-attachments/assets/1a525cac-d5c9-46ac-90ec-59a8dff7cd2a" />



​  
그리고, 저장된 json을 열어보면 아래와 같이 나옵니다.  

<img width="773" height="676" alt="로깅된 json 파일  " src="https://github.com/user-attachments/assets/d8b84a7c-229f-47ba-96b6-14fbfbff6bc1" />


​  
하지만 이걸 눈으로 하나하나 분석을 하기엔 방대한 양에,  
눈이 아파질 것 같으므로  
크롬에서 제공해주는 다른 분석툴을 통해 시각화 해보도록 하겠습니다.

> 로그 파일 분석하기

먼저 netlog-veiwer(https://netlog-viewer.appspot.com/#import)로 접속합니다.

파일을 선택하면 아래와 같이 표로 정리해줍니다.  
​
<img width="773" height="163" alt="image" src="https://github.com/user-attachments/assets/0d997963-bda2-4671-9689-8dee02134eda" />

​  
이제, 왼쪽 패널의 Events에 들어가서 패킷 요청 하나씩 직접 뜯어볼 수 있습니다.  
이미지 요청 중 하나인 1954020의 요청을 뜯어보도록 하겠습니다.  
​
<img width="773" height="643" alt="image" src="https://github.com/user-attachments/assets/06e92183-378d-4a6e-b822-8a2996c85939" />

​  
하지만, 이 역시 어떤 정보를 봐야하는지 헷갈립니다.  
정확하게 어떤 요청들이 TCP를 공유하고 있는지 파악하기가 어렵습니다.  
​  
하지만, 아래 사진과 같이 로그 중에 "source_dependency"를 보면,  
1954020 요청이 결국 1954035에 의존하고 있다는 것을 확인할 수 있어요.  
​
<img width="773" height="74" alt="image" src="https://github.com/user-attachments/assets/dce859bb-10c2-4e66-9d15-8a455b487868" />
<img width="773" height="74" alt="image" src="https://github.com/user-attachments/assets/9ae8b3e1-b980-4c13-b56e-58fc5f888728" />


​
그리고, 다시 1954035에 가면, **"source_dependency"가 195417 SOCKET으로 연결되어 있는 것을 확인**할 수 있습니다.  
이 SOCKET이 궁극적으로 찾던 것입니다.

<img width="773" height="627" alt="image" src="https://github.com/user-attachments/assets/1837077c-a965-4a9a-ae21-831497434c6b" />

  
​
잠깐 다른 source_dependency도 확인을 하자면,  
​

- 1954034 (HTTP_STREAM_JOB_CONTROLLER)

  이건 HTTP_STREAM_JOB_CONTROLLER를 가르키는데, 컨트롤러가 스트림 잡을 관리한다고 보면 됩니다. 즉, "35 스트림잡은 34가 관리해요 ~" 라는 뜻입니다.

- 1954146 (SSL_CONNECT_JOB)

  TCP 소켓을 만들고, https니까 SSL hand-shake 과정을 거쳐야 해요.  
   거기서 이제 SSL_CONNECT_JOB을 의존해서 TLS을 확보하게 됩니다.

- 1954147 (SOCKET)

  이제 진짜 만들어진 TCP 소켓 객체에 묶였다는 의미입니다.

  여기서부터 진짜 요청이 1954147이라는 소켓을 통해 흘러간다는 것을 알 수 있습니다.

- 1954020 (URL_REQUEST)

  결국 얘가 왜 만들어졌냐 195420 << 얘 때문에 만들어졌다를 의미합니다.

  다시 돌아가서, SOCKET 요청을 찾았으니,
  1954147 소켓을 찾으면 아래와 같은 다양한 로그들을 확인할 수 있습니다.
  ​
<img width="773" height="424" alt="image" src="https://github.com/user-attachments/assets/f308d4f3-53e4-42d7-a194-22215ef71f99" />

​
그리고, "SOCEKT_IN_USE"​를 통해 언제 이 소켓이 사용되었는지 추적할 수 있는데요,

<img width="773" height="298" alt="image" src="https://github.com/user-attachments/assets/92dfab56-0ae8-4cd0-bb3d-63bf042f5812" />
  
이런 식으로 소켓이 계속 재사용되고 있는 중이라는 것을 확인 할 수 있습니다.  
​
이제 하나의 소켓을 확인 한 것이고,  
앞으로 남은 5개를 찾아야 합니다.  
​  
하지만, 아래의 방법으로 조금 더 쉽게 소켓 요청을 확인할 수 있습니다.  
​  
​
​  
​

# ‼️ 진짜 소켓이 6개인지 확인 쉽게하기

​
netlog-viewer 좌측 패널에 보면, Sockets가 있습니다.  
여기서 destination을 보고 우리가 요청하는 S3 주소에 대해 소켓이 몇개 열렸는지 확인할 수 있습니다.  
 ​
<img width="773" height="20" alt="image" src="https://github.com/user-attachments/assets/bfdfa9e0-0a9e-4010-a576-1244e655f595" />

​
정말로 idle 상태로 들어간 소켓이 6개네요.  
그리고 6을 클릭해주면 관련된 TCP 소켓들을 표로 확인할 수 있습니다.  
​
<img width="773" height="407" alt="image" src="https://github.com/user-attachments/assets/d582ff3f-193e-4874-9783-6e3ed27083da" />

​
한 소켓에서 맨처음 분석했던 1954935 요청이 있는 것으로 보아 재검증이 되었네요.  
​  
길었지만,  
**결론은 TCP 연결 6개로 100장의 이미지들이 공유되는 것을 확인할 수 있었어요.**  
​

​<img width="773" height="351" alt="image" src="https://github.com/user-attachments/assets/e9c4ba96-2da3-4f51-bac9-ffbdc4a9f9e8" />

  
그래서 이런식으로 처음 6개 이미지는 동시에 요청이 되는 반면,  
응답이 오는 시점에 따라 뒷 이미지들의 요청이 밀리게 되는 것이었습니다.

​
그래서 한 응답이 딜레이가 되면 결국 뒷 요청도 계속해서 밀리게 되겠죠.  
​
​
​
​
​

​

# 📣 어떻게 http/2로 보낼 수 있도록 요청하나요?

​  
일단 기존에는 s3 주소로 직접 접근해서 이미지들을 직접 가져오고 있었어요.  
그래서 terminal을 통해 "최대한 http2로 날려줘"라고 해도, 응답이 http/1.1로 오는 것을 알 수 있어요.

<img width="773" height="307" alt="image" src="https://github.com/user-attachments/assets/3e11c2f6-720c-4daf-b0d6-0c5a7b912640" />

​  
그리고 정확한 공식 문서를 찾을 수는 없으나,  
아래 글과 위 명령어를 통해 S3 엔드포인트에 직접 접근할 때 클라이언트나 네트워크 설정에 따라 HTTP/1.1로 폴백될 수 있다고 파악할 수 있어요.

​
https://repost.aws/questions/QU22F0GAPNRfmTPdW283_i7A/does-amazon-s3-support-http-2
https://stackoverflow.com/questions/45471590/serve-s3-resources-via-http-2
https://github.com/aws/aws-sdk-java-v2/issues/4615
​

그런데, 자세히 로그를 분석하다가 알게된 것은  
CDN을 거쳐서 오는 소스코드들은 http/2로 오고 있다는 것이었어요.  
 ​
<img width="773" height="283" alt="image" src="https://github.com/user-attachments/assets/7e9c9c86-a1ed-450e-9339-4ec52144f0c2" />

​  
위 링크들을 들어가셨다면 파악하셨을 수도 있지만,  
CDN은 자체적으로 http2로 래핑해서 보내주고 있었어요.  
심지어 http/3도 지원한다는 공식 문서가 있어요.

​
https://aws.amazon.com/ko/blogs/korea/new-http-3-support-for-amazon-cloudfront/

그래서 직접 클라이언트가 S3에서 이미지를 들고오는 것이 아니라,  
클라이언트 -> CDN -> S3로 가면 되겠구나는 것을 깨달았습니다.  
​  
그리고 CDN을 사용하면, 캐싱 정책도 적용할 수 있다는 장점이 있어요.  
브라우저나 CDN에서 리소스를 캐싱해두면, 매번 origin까지 요청이 가지 않아도 됩니다.  
​  
그래서 이미지를 주로 다루는 포게더에서는 http2는 덤으로 따라오고  
이미지를 캐시해서 origin까지 요청이 가지 않도록 리소스를 세이브할 수 있었습니다.  
​
<img width="773" height="315" alt="image" src="https://github.com/user-attachments/assets/59d64c37-092a-4f7a-a21e-5cae5781e20a" />

​  
캐시 정책에서는 TTL을 1년으로 두고,  
응답 헤더 정책에서도 max-age를 1년으로 설정하면서  
올바르게 캐시 전략도 적용하였어요.  
​  
그리고, 배포한 이미지전용 CDN 주소를 통해 http2 요청을 보내보면  
아래와 같이 HTTP/2 200이 뜨는 것을 확인할 수 있습니다.

​  <img width="773" height="364" alt="image" src="https://github.com/user-attachments/assets/f44fec08-d8f9-49fb-a122-61f56dd8062e" />

그리고 local로 돌려보았더니 아래와 같이 요청이 나란히 가는 것을 확인할 수 있었습니다 ㅎㅎ  
​

​  <img width="773" height="397" alt="image" src="https://github.com/user-attachments/assets/554c6d17-663c-4a08-bd06-bdb2a561e0c6" />

​
​
​

# 하지만 여전히 LCP의 Load Delay는 ...

​
<img width="386" height="303" alt="local" src="https://github.com/user-attachments/assets/f2e5de93-b4ba-457e-9a2c-a930d8f77cde" />
<img width="386" height="323" alt="prod" src="https://github.com/user-attachments/assets/70db1adc-9674-4050-b881-46199eef1bec" />


로컬 기준으로 이전과 비교하여 Load Delay가 15~16초 정도 줄어든 것을 확인할 수 있습니다.

​
하지만,  
여전히 Load Delay가 다른 phase와 비교해서 오래 걸리는 것으로 보아, 근본적인 해결 방법은 아닙니다.  
​  
Load Delay 자체가
`리소스 요청(Request)의 시점과 실제 브라우저 리소스를 네트워크로 가져오기 시작(Start Loading)의 시간 지연`이 크다는 뜻입니다.  
​  
그래서 이미지 데이터 자체가 네트워크에서 전송되는 속도가 느리다.
의 뜻이 아니라,  
"이미지 요청할 차례가 올 때까지 브라우저 기다리는 시간이 길다"는 의미입니다.  
​  
webpagetest를 돌려본 결과,  
​
<img width="773" height="414" alt="image" src="https://github.com/user-attachments/assets/878ecae4-1f25-4090-9c89-6cd4212d109e" />

​
메인 소스코드 js 자체를 불러오는 데 시간이 오래 걸려서,  
Load Delay가 크다는 것을 알 수 있습니다.  
​  
추가적으로 Code Split, Tree-shaking, 초기 JS 번들 최적화 등을 통해  
이미지 요청 시작 전까지 지연 시간을 줄이는 것이 우선임을 알 수 있습니다.

# 번외) 왜 prod 서버에서는 http/2가?

위 문제를 해결하다가,  
하나 흥미로운 것을 발견했습니다.  

<img width="773" height="58" alt="dev api 호출" src="https://github.com/user-attachments/assets/c277d393-4f82-4cf0-b0d4-765d7da5d68b" />
<img width="773" height="59" alt="prod api 호출" src="https://github.com/user-attachments/assets/58a484df-810f-4aa3-8ce0-b41e6c1cc826" />

​  
동일한 API를 호출했는데,  
개발 서버(dev) api를 호출하면 http/1.1을 사용하고,  
운영 서버(prod) api를 호출하면 http/2를 사용하는 것이었습니다.  
​
  
​  
클라이언트 단에서는 prod와 dev는 빌드 산출물 결과만 다르고,  
동일한 방식으로 배포를 하였기 때문에 동일한 환경이라고 보아도 무방합니다.  
​  
그런데 "왜 프로토콜이 다를까?"하는 의문이 생겼습니다.  
​

> prod와 dev의 인프라 차이

이 의문점을 해결하기 위해서  
prod와 dev의 인프라차이가 존재하는지 부터 확인해봤습니다.  
​두 환경의 차이는 ALB(Application-Load-Balancer) 였습니다. 

<img width="773" height="393" alt="포게더 인프라 일부 요약" src="https://github.com/user-attachments/assets/9338aceb-dc3e-498a-945e-c631db47a226" />

​


​  
dev에서는  
브라우저에서 요청을 하면, 바로 EC2의 퍼블릭 인스턴스로 접근하는 구조였습니다.  
​
prod에서는  
브라우저에서 요청을 하면, ALB를 거쳐서 대상그룹으로 요청을 보내는 구조였습니다.  
포게더의 경우에는 EC2 인스턴스였던 것이죠.  
​  
이런 인프라 구조는 aws 콘솔을 통해서 확인할 수도 있고,  
터미널에 dig 명령어를 통해 amaxonaws.com이 포함된 CNAME을 확인해서 추론할 수도 있습니다.  
​

> ALB란?

ALB는 Application Load Balancer의 약자입니다.  
직역하면 어플리케이션 부하 나눔 정도가 되겠네요.  
​  
ALB는 AWS에서 제공하는 로드 밸런서 중 하나인데요,  
이름에서 알 수 있듯이 OSI 7계층 중 어플리케이션 계층에서 동작하는 로드 밸런서를 말합니다.  
​  
사용자 요청을 받아서(application load) 여러 서버로 나눠서 분배(balnace)하는 역할을 합니다.  
​  
AWS에서는 아래와 같이 설명합니다.  
​
<img width="555" height="243" alt="image" src="https://github.com/user-attachments/assets/e5cc2d8d-6b50-420a-8a36-5e461e4bbb0e" />

​  
ALB는 Load Balancer, Listener, Target Group으로 구성되어 있습니다.  
​  
ALB는 요청의 Host, Path, Header, Method를 보고 Target Group을 선택하여 보냅니다.  
​  
예를 들어,

```if
host = api.pongju AND Path starts with /pongju
(호스트 이름이 api.pongju이고, path가 /pongju로 시작하면)
then
EC2 Target Group
(EC2로 요청 보내기)
​
if
Path starts with /upload
then
Lamda Target Group
(Lamda로 요청 보내기)
```

​  
이렇게 규칙을 Listener에게 등록 시킬 수 있습니다.  
​  
이제 이 규칙을 보고 ALB가 적절히 트래픽을 분산(라우팅)시키는 것입니다.  
​  
서버1의 헬스체크를 확인하고 만약 죽었다면, 서버2로 보내는 역할 등을 하면서  
정상적으로 서비스가 이루어질 수 있도로 하는 것입니다.  
​

> ALB의 역할

1.  트래픽을 분산

    들어오는 요청들을 적절히 여러 서버로 분산시키면서 한 서버가 모든 부하를 감당하지 않도록 합니다.

2.  헬스 체크

    서버가 정상, 비정상인지 확인하여 비정상인 경우 정상 서버로 요청을 보냅니다.

3.  SSL 종료

    HTTPS를 사용하는 경우, SSL 인증과정을 ALB에서 처리하고, 내부 서버에서는 HTTP로 전달할 수 있습니다.

4.  라우팅

    Host, Path, Method, Header 등 기반으로 각기 다른 대상 그룹으로 요청을 보냅니다.

5.  세션 유지
    같은 사용자에 대한 요청을 항상 같은 대상 그룹으로 보냅니다.

> ALB의 속성을 보았더니

<img width="773" height="135" alt="image" src="https://github.com/user-attachments/assets/17fab1c1-2f6a-4c02-a964-29a57b78cfe8" />

​  
이렇게 ALB 속성에  
HTTP/2 활성화 옵션이 켜져 있었습니다.  
​  
따라서,  
dev 환경에서는 EC2에 직접 접근하므로 http/1.1로,  
prod 환경에서는 ALB를 통해 접근하여 http/2로 응답하는 구조였습니다.  
​  
이러한 차이로 인해  
prod와 dev간이 요청에 대한 프로토콜이 다르게 적용되었던 것입니다.
