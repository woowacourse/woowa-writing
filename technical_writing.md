# IAM을 활용하여 Spring에서 AWS KEY 노출 시키지 않기

> 해당 글은 S3를 한번이라도 프로젝트에 적용한 경험이 있는 개발자 또는 S3에 대한 기본적인 지식이 있는 개발자를 위한 글입니다.

## Spring에서 AWS KEY정보 이렇게 저장하고 있는가?

혹시 S3를 스프링 프로젝트에 적용할 때, 아래와 같은 코드를 사용하고 있지는 않은가?

```yaml
spring:
  cloud:
    aws:
      credentials:
        accessKey: <YOUR_ACCESS_KEY>
        secretKey: <YOUR_SECRET_KEY>
      s3:
        bucket: <BUCKET_NAME>
      region:
        static: <REGION>
      stack:
        auto: false
```

대부분의 S3를 처음 사용해보는 개발자는 이렇게 yml 파일에 S3의 key정보를 넣고 사용하게 된다. 하지만 이러한 방법은 문제점이 있다.

## **위 S3사용법의 보안 문제점**

accessKey와 scretKey는 AWS 계정에 대한 자격증명을 위한 키이다. 이러한 키가 github public repository나 인터넷에 올라가게 되면 나의 AWS 서비스는 침범당할 수 있다.

그러면 해당 설정 yml파일을 .gitignore에 추가해서 노출 안 되게 하면 되지 않나? 라는 생각을 할 수도 있다. 하지만 그런 생각은 개인 프로젝트까지 맞는 말이다.

팀원과 모든 내용을 공유한다고 해도 직접 작업한 사람이 아니면 쉽게 잊게 된다. 해당 설정이 담긴 yml 파일의 이름만 바꿔도 .gitignore에서 제외될뿐더러, 어떠한 방법으로도 노출되려면 노출될 수 있다.

그렇기 때문에 실제 회사를 가게 되면, 애초에 개발자에게 key 정보를 전혀 주지 않을 가능성이 높다.

key 정보를 모두가 모르는 상태에서 spring에서 S3 인프라를 구축해두는 방법을 알아보자

## S3, Spring 다이어그램

오늘 우리가 만들 구조를 다이어그램으로 알아보자
![image1](https://github.com/user-attachments/assets/14509ec8-2bff-4a55-8b28-b0fe9530aee7)

1. S3 버킷에 사진을 요청해서 받아올 수 있는 **역할을 생성**
2. EC2 인스턴스에 해당 **역할을 부여**
3. Spring Application 실행 시, 해당 Application이 실행된 **EC2로부터 역할을 찾는다**
4. **S3에 접근 가능한 역할로 인증받고** S3에 접근한다.

위에서 설명한 단계에서 반복적으로 나오는 단어가 있다. 그것은 역할이다. 미리 EC2 인스턴스에 특정 행동을 할 수 있는 역할을 부여하고, 해당 역할로 Application 실행할 때, S3에 접근하고 있다.

그러한 역할을 부여할 수 있는 AWS 서비스가 바로 **IAM**이다.

# AWS IAM이란?

**IAM(Identity and Access Management)**은 AWS에서 사용자가 **누가 무엇에 접근할 수 있는지**롤 관리하는 서비스이다.

 쉽게 말해, **“누구에게 어떤 권한을 줄 것인가?＂** 를 결정하고 관리하는 시스템이다.

위에서 설명한 그림에서는 그 ‘누구’가 EC2가 된 사례다.

IAM에서는 ‘누구’를 사용자, 그룹 역할 총 세 가지로 분류한다. 각 ‘누구’에 다른 정책을 부여할 수 있다.

### IAM 주요 개념

1. **사용자(User)**:
    - AWS에 로그인해서 서비스를 사용할 수 있는 사람. 사용자마다 개별 계정이 있고, 각자의 권한이 다를 수 있다.
2. **그룹(Group)**:
    - 여러 사용자를 하나로 묶어, 한 번에 같은 권한을 부여하는 방법. 예를 들어, ＂개발자 그룹＂, ＂관리자 그룹＂처럼 구성할 수 있다.
3. **역할(Role)**:
    - 사람이 아닌, **AWS 서비스나 애플리케이션에 권한을 부여하는 방법**. 예를 들어, EC2 인스턴스가 S3 버킷에 접근하려면 IAM 역할을 할당해 줄 수 있다. 역할을 이용하면, 비밀번호나 키 없이도 안전하게 AWS 리소스에 접근할 수 있다.
4. **정책(Policy)**:
    - **누가 무엇을 할 수 있는지**를 구체적으로 정의한 규칙이다. 예를 들어, ＂S3 버킷에서 파일을 읽을 수 있지만 쓸 수는 없다＂는 규칙을 정책으로 정의할 수 있다. 정책을 사용자, 그룹, 또는 역할에 부여하여 권한을 설정할 수 있다.

이제는 위에서 그렇게 **역할** 이라는 단어를 자주 꺼낸 이유를 알 수 있을 것이다. 바로 EC2(AWS 서비스)에 S3에 접근할 수 있는 권한을 줄 것이기 때문에, 역할을 생성, 부여라는 단어를 사용한  것이다. 

그러면 이제 실제로 적용하는 방법을 알아보자

# Spring, S3, IAM 적용하기

### 준비물

EC2, S3, Spring 프로젝트

이미 EC2에 배포된 Spring프로젝트이면 좋겠다. 배포하는 것까지는 이 포스트에서 다루지 않겠다.

S3도 미리 만들어 놓기를 바란다. 그리고 버킷이름을 어딘가에 복사해두자.

## S3에 IAM 역할 부여하기

해당 단계는 4가지 단계로 나누어서 설명하겠다.

1. **IAM 정책 생성**: S3에 접근할 수 있다. 라는 **정책**을 먼저 생성해야 된다.
2. **IAM 역할 생성:** EC2가 위에서 생성한 **정책**을 할 수 있다는 **역할**을 생성해야 된다.
3. **EC2 역할 부여:** EC2에서 위에서 생성한 **역할**을 할 수 있다고 부여해주어야 한다.
4. **Spring에서 S3 인증하기:** EC2환경의 Spring 프로젝트에서 S3에 인증을 해본다.

### 1. IAM 정책 생성
![image2](https://github.com/user-attachments/assets/3539647b-9f7f-4aa9-8bdf-494a7ae9f5ef)


IAM → 정책 → **정책생성**을 선택한다

그리고 **서비스 선택 → S3** 선택

그러면 S3에 관한 여러 가지 엑세스 수준이 나오는데, 필자는 사진을 저장하고 불러오는 기능만 허용할 것이기 때문에, PutObject, GetObject만 선택한다. 필요한 것을 더 추가해서 넣어도 된다.

![image3](https://github.com/user-attachments/assets/c9b10535-fece-4e51-9b68-7283c6a55bf5)


그리고 리소스를 선택해야 하는데, S3의 어느 버킷에 접근 권한을 줄 것 인가를 선택하는 것이다.

모든 버킷에 접근해도 되면 **모두**를 선택하고, 특정 버킷에 허용하고 싶으면 모두를 선택하지 않고 S3에 생성해둔 버킷 이름을 넣는다. 만약 버킷 안에서도 특정 오브젝트그룹에만 접근하게 하고 싶으면 obejct name도 작성한다.

예를들어, dodo-bucket-test버킷의 public이라는 패키지의 모든 오브젝트에 접근하게 해주고 싶으면, Resource object name에 **“public”** 을 하면 된다. 필자는 **모든 object name**을 허용했다.

![image4](https://github.com/user-attachments/assets/d89d9d5c-8a1d-406b-986f-58cdfe5b48a5)


다음은, 적절한 정책이름을 주고 **정책생성**을 클릭하면 된다.

### 2. IAM 역할 생성

![image5](https://github.com/user-attachments/assets/79ebe9d8-63f9-4e6f-a254-0691afdc3fd9)


IAM→역할→역할생성을 한다.

신뢰할 수 있는 엔티티 유형은 **AWS 서비스**로한다.

그리고 **서비스는 EC2를 선택**한다. 

**다음**을 눌린다.

![image6](https://github.com/user-attachments/assets/96e0821c-550c-43b6-9b75-8903e48c4604)


아까 생성한 **정책이름을 검색해서 선택**하고 **다음**을 선택한다

![image7](https://github.com/user-attachments/assets/0cc74bdc-d709-4ab7-95e7-1587a72abbc5)


적절한 역할 이름을 부여하고 **역할 생성**을 눌리면 된다. 다음을 눌리기 전에 몇 가지 항목이 더 있는데, 요약이니까 보고 넘어가면 된다.

### **3. EC2 역할 부여**

![image8](https://github.com/user-attachments/assets/dbf10560-4597-4b7a-b3f6-b47695db2ef3)


적용할 EC2에 들어가서 **작업→보안→IAM** **역할 수정**을 누른다

![image9](https://github.com/user-attachments/assets/b0a13180-ccbd-40f9-b028-3d5ec3a45422)


그리고 **생성한 IAM 역할을 부여**하고 **IAM 역할 업데이트**를 누른다.

### 4. **Spring에서 S3 인증하기**

이제 해당 EC2에서는 키 없어도 S3에 Put과 Get이 가능하게 되었다. 지금부터 스프링에서 S3에 접근하는 방법을 보여주겠다.

이번 단계는 IAM 부여 전, 부여 후 나누어서 보여주겠다.

필자의 S3 sdk 버전은 아래와 같다

```yaml
implementation ＂software.amazon.awssdk:s3:2.26.21＂
```

### IAM 적용 전 코드

```java
public class S3StorageManage {

    // application.yml에서 값을 불러옴
    @Value("${aws.access-key}")
    private String accessKey;

    @Value("${aws.secret-key}")
    private String secretKey;

    private final S3Client s3Client;

    public S3StorageManager() {
        this.s3Client = S3Client.builder()
                .credentialsProvider(
                        StaticCredentialsProvider.create(
                                AwsBasicCredentials.create(accessKey, secretKey)
                        ))
                .region(Region.AP_NORTHEAST_2)
                .build();
    }
}
```

세부적인 구현 방법은 다를 수 있으니 초기화하는 부분만 참고하길 바란다.

코드에서 볼 수 있듯이 @Value 어노테이션을 활용해서 yml에 있는 accessKey와 secretKey값을 참조하고 있다.

그리고 해당 key로 Aws인증을 거치고 있다.

이러한 코드의 문제는 서두에서 이야기했으니 넘어가겠다.

### IAM 적용 후 코드

```java

public class S3StorageManage {

    private final S3Client s3Client;

    public S3StorageManager() {
        this.s3Client = S3Client.builder()
                .credentialsProvider(
                        InstanceProfileCredentialsProvider.create())
                .region(Region.AP_NORTHEAST_2)
                .build();
    }
}
```

필드에 아무런 정보도 없게 됐다. `InstanceProfileCredentialsProvider.()` 메서드로 인증을 거치고 있는 것으로 보인다. 이렇게 개발자는 코드에 민감정보인 Key를 올리지 않아 보안 적으로 노출될 위험은 없어졌다.

그러면 어떻게 인증을 하고 있는가?

이 부분은 동작 원리 내용이라 궁금하면 읽고 아니면 넘어가길 바란다.

### IAM은 어떻게 EC2의 어플리케이션에서 인증을 하고 있는 것인가?

 해당 create()내부를 보면 이러한 코드를 찾을 수 있다.

```java
private String[] getSecurityCredentials(String imdsHostname, String metadataToken) {
        ResourcesEndpointProvider securityCredentialsEndpoint =
            new StaticResourcesEndpointProvider(URI.create(imdsHostname + SECURITY_CREDENTIALS_RESOURCE),
                                                getTokenHeaders(metadataToken));
...생략
```

여기서 주목할 부분은 `URI.(imdsHostname + SECURITY_CREDENTIALS_RESOURCE` 은 이 부분이다.

이 URI는 변수를 찾아서 바꿔 보면 `http://169.254.169.254/latest/meta-data/IAM/security-credentials/` 가 나오게 된다.

`169.254.169.254`는 EC2 인스턴스가 자신과 관련된 메타데이터 및 IAM 역할의 자격 증명을 조회하는 데 사용하는 특별한 내부 IP 주소이다.

 `/latest/meta-data/iam/security-credentials` URI로 해당 EC2에 부여된 iam을 역할을 찾게 된다.

더 자세한 내용은 [인스턴스 메타데이터를 사용하여 EC2 인스턴스를 관리](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-metadata.html) 를 참고하길 바란다.

이렇게 해당 EC2에 적절한 S3 접근권한이 있다면 S3Client 객체를 생성해서 사용할 수 있게 된다.

이후 동작은 기존에 하던 코드대로 하면 된다. 이 글에서는 Key 정보를 코드에 노출되지 않도록 하는 것에 집중하여 이후 이야기는 하지 않겠다.

# 마무리

IAM이 무엇인지, EC2에 IAM 역할을 부여하기, Spring에서 IAM 인증을 거치는 방법까지 알아보았다.

이렇게 하면, 이제 로컬 환경에서는 Spring IAM 인증 과정에서 실패하게 될 것이다.

이러한 문제는 필자는 Local 환경과 EC2 환경 각각 다른 빈을 선택하도록 해결했다. 다른 해결 방법도 여러 가지 있으니, 여러 가지 알아보기를 바란다.

### 참조

[IAM 역할을 사용하여 Amazon EC2 인스턴스에서 실행되는 애플리케이션에 권한 부여](https://docs.aws.amazon.com/ko_kr/IAM/latest/UserGuide/id_roles_use_switch-role-ec2.html)

[인스턴스 메타데이터를 사용하여 EC2 인스턴스를 관리](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-metadata.html)

[Amazon EC2에 대한 IAM 역할](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/iam-roles-for-amazon-ec2.html)
