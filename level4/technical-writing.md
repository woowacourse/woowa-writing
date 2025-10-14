# 💬 들어가며

안녕하세요 헤일러입니다.

저는 서비스를 기획하고, 그 기획을 실제 서비스로 구현하는 데 관심이 많습니다.

백엔드 개발자의 관점에서, 기획 프로세스를 구체화하는 것보다는 프로토타입 → 서비스 런칭으로 이어지는 빠른 사이클을 만들어보는 연습을 하고 있는데요.

![](https://velog.velcdn.com/images/heiler/post/8a5e8a87-41d7-4edc-b1bc-6c342068aa99/image.png)

요즘은 Lovable과 같은 노코드 AI를 이용하면 프로토타입을 쉽게 만들어 볼 수 있습니다. 기획한 내용으로 여러 개의 프로토타입을 만들어 보다 보면 실제 서비스로 만들고 싶은 마음에 드는 결과물이 나올 때가 종종 있어요.

> 최근 Lovable + Cursor로 만들어본 프로토타입 앱들
> 1.  데이트 코스 공유 & 지도 네비게이터(웹 앱) - [[GitHub](https://github.com/threepebbles/courseitda-date-route)] [[배포 링크](https://threepebbles.github.io/courseitda-date-route/)]
> 2. 동선 플래너(PC) -  [[GitHub](https://github.com/threepebbles/route-wander-visualizer)] [[배포 링크](https://route-wander-visualizer.lovable.app/)]
> 3. 코스잇다(웹 앱) - [[GitHub](https://github.com/threepebbles/day-trip-pro)] [[배포 링크](https://day-trip-pro.lovable.app/)]

프로토타입 단계에서 이거다! 라는 판단이 들었다면, 이제 운영 가능한 서비스로 옮기기 위해 자체 인프라를 준비해야 합니다.

이 때 IGW, Route table, VPC, Subnet, EIP, EC2, RDS, S3 등 이미 익숙한 인프라 구조를 재현하는 일이 굉장히 귀찮고 번거로웠습니다.

이후에 개발(dev), 운영(prod), 스테이징(staging) 환경처럼 구조는 비슷하고, 하드웨어 스펙만 조금씩 다른 환경을 여러 개 만든다면 더 그럴 것 같았습니다.

이 문제를 해결하기 위해, 이미 설계된 인프라를 일관되게 재현하고 쉽게 관리할 수 있는 방법을 찾아보다 Terraform을 처음 접하게 됐습니다.

올해 추석 연휴에 시작한 코스잇다라는 프로젝트에 Terraform을 처음 적용해보게 되었는데요.

프로젝트에서 사용했던 코드를 예제로 들어 Terraform 입문자가 알아두면 좋을만한 핵심 개념을 정리해 보았습니다. 그리고 사용 중에 겪었던 어려움도 몇 가지 공유하고자 합니다.

Terraform을 처음 접하는 분들께 도움이 되길 바랍니다. 😄

---

# ✅ Terraform 핵심 개념

## Terraform이 무엇인가요?
> HashiCorp Terraform is an **infrastructure as code tool** that lets you define both cloud and on-prem resources in **human-readable configuration files** that you can version, reuse, and share.
>

간단히 말해, Terraform은 **사람이 읽기 쉬운 설정 파일**로 인프라를 정의하고 관리하는 **IaC 도구**입니다.

IaC는 단어 뜻 그대로 인프라를 코드로 관리하는 방법론입니다.

코드를 실행하면 작성한 코드대로 인프라가 짠하고 만들어지는 거죠.

대표적인 IaC 도구인 Terraform의 특징 3가지와 핵심 키워드 7가지를 소개하겠습니다.

## Terraform의 특징

### 1) 사람이 읽기 쉬운 언어(**human-readable)**

Terraform은 HCL(HashiCorp Configuration Language) 언어를 사용합니다.

HCL에서는 block, argument 구조로 코드를 작성하는데요.

![](https://velog.velcdn.com/images/heiler/post/c49c7bf9-d6a7-4d98-adb8-c9b94b0a088d/image.png)

HCL을 학습하면서 느낀 점은 HCL에 대한 러닝 커브보다는 사용하는 클라우드(예: AWS)에 대한 이해도가 더 큰 러닝 커브라 느꼈습니다.

AWS를 이미 잘 알고 있는 분이라면, AWS 관련 HCL 코드를 작성하는 것은 크게 어렵지 않을 거라 생각합니다.

### 2) 선언적(Declarative)

원하는 최종 인프라 상태만 코드로 선언하면, 그 상태를 달성하기 위해 필요한 작업은 도구(Terraform)가 알아서 해주는 특징을 `선언적`이라고 합니다.

AWS EC2를 생성하는 간단한 코드를 예로 들어보겠습니다.

```hcl
resource "aws_vpc" "vpc" {
  cidr_block       = "10.0.0.0/16"
  instance_tenancy = "default"
}

resource "aws_subnet" "subnet" {
  vpc_id            = aws_vpc.vpc.id    # vpc 참조
  cidr_block        = "10.0.0.0/24"
  availability_zone = "ap-northeast-2a"
}

resource "aws_instance" "app" {
  ami = "ami-0607797cadde98e9b"
  instance_type = "t4g.small"
  subnet_id     = aws_subnet.subnet.id   # subnet 참조
}
```

코드를 보면 app(EC2)에서 subnet을 참조하고, subnet에서 vpc를 참조하고 있습니다.

실제 리소스를 생성할 때, vpc를 먼저 생성하고, 그다음 subnet, 그다음 EC2 순서로 리소스를 생성해야 합니다.

Terraform은 리소스 간 참조 관계를 기반으로 참조 그래프를 만들고, 그 순서에 따라 순서대로 리소스를 생성해줍니다.

### 3) 멱등성(Idempotency)

같은 Terraform 코드로 여러 번 실행한다고 해서 인프라를 계속 생성하지 않습니다. 현재 인프라 상태를 추적해서, 코드 변경에 의해 발생한 차이만 실행에 반영하기 때문에 멱등성이 보장됩니다.

## Terraform 핵심 키워드

### 1) provider

![](https://velog.velcdn.com/images/heiler/post/50ee0b15-000c-4819-b5c7-898859d6a488/image.png)

provider는 Terraform과 외부 인프라 서비스(Target API) 사이를 연결해주는 플러그인 역할을 합니다.

코드로 AWS 인프라를 정의해두면, 내부적으로 Terraform Core가 AWS API를 호출해서 AWS 인프라를 생성합니다.

아래의 코드는 AWS provider를 사용하는 예제입니다.

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.12"
    }
  }

  required_version = ">= 1.2"
}

provider "aws" {
  region = "ap-northeast-2"
}
```

### 2) resource

resource 블록은 VPC, Subnet, EC2, RDS와 같은 하나의 **리소스를 정의**하는 블록입니다.

```hcl
resource "aws_instance" "app_instance" {
  ami           = data.aws_ami.app_ami.id
  instance_type = var.instance_type

  ...
  tags = {
    Project     = var.project_name
    Environment = var.environment
    Name        = "${var.project_name}-app"
  }
}
```

익숙하지 않은 data, var와 같은 키워드들이 등장해 혼란스러울 수 있으실 텐데요. 자세한 내용은 아래에서 살펴보겠습니다. 😄

### 3) variable

variable 블록은 같은 모듈(폴더) 내 다른 Terraform 코드에서 사용할 수 있는 변수를 정의합니다.

가령 같은 모듈에 variables.tf와 main.tf가 존재한다고 하겠습니다.

```
# 폴더 구조
.
├── main.tf
└── variables.tf
```

```hcl
# variables.tf
variable "project_name" {
  type        = string
  description = "Project name"
  default     = "courseitda"
}

variable "environment" {
  type        = string
  description = "Environment name (e.g., dev, prod)"
  default     = "dev"
}
```

```hcl
# main.tf
resource "aws_eip" "app_eip" {
  domain = "vpc"

  tags = {
    Project     = var.project_name
    Environment = var.environment
    Name        = "${var.project_name}-${var.environment}-app-eip" # courseitda-dev-app-eip
  }
}
```

위 코드와 같이 main.tf에서 variables.tf에 정의한 변수를 사용할 수 있습니다.

별다른 조치 없이 variable에 default 값을 입력하지 않으면, Terraform 코드를 실행하는 시점에 콘솔로 사용자 입력을 받게 됩니다. 

default 값을 variable 선언부에 두지 않고, 실행 시점에 외부에서 값을 주입해서 사용할 수 있는 방법이 여러가지가 있는데요.

그 중에 자주 사용되는 방법 중 하나가 module을 사용하는 방법입니다.

### 4) module

Terraform에서는 하나의 폴더 단위를 모듈이라고 부릅니다.

이 모듈 덕분에 구조는 비슷한데 스펙만 다른 환경별(dev, prod) 인프라를 쉽게 관리할 수 있습니다.

저는 EC2 생성 로직을 아래와 같이 modules 폴더 하위에 모듈화 해놓고 사용하고 있습니다.

```
# 폴더 구조
.
├── environments
│   ├── dev
│   │   ├── backend.tf
│   │   ├── main.tf
│   │   ├── provider.tf
│   │   └── variables.tf
│   └── prod
│       ├── backend.tf
│       ├── main.tf
│       ├── provider.tf
│       └── variables.tf
└── modules
    └── application
        ├── main.tf
        ├── outputs.tf
        └── variables.tf
```

환경 간 공통적인 설정은 모듈로 표준화하고, 차이는 변수로 주입해 재사용성을 얻을 수 있습니다.

모듈화 해놓은 코드는 module 블록으로 불러와 재사용할 수 있고, variable을 함께 사용하면 환경별로 필요한 값만 다르게 설정할 수 있습니다.

예를 들어 dev 환경과 prod 환경에서 EC2의 인스턴스 타입, 볼륨 타입, 볼륨 크기에만 차이를 두고 싶다면, 아래 코드와 같이 그 차이를 두고 싶은 값을 모듈에서 variables.tf로 분리해두고, 각 환경의 main.tf에서 다른 값을 주입하여 사용할 수 있습니다.

```hcl
# modules/application/variables.tf
variable "instance_type" {
  type        = string
  description = "EC2 instance type (e.g., t4g.micro)"
}

variable "volume_type" {
  type        = string
  description = "EBS volume type (e.g., gp2, gp3)"
}

variable "volume_size" {
  type        = number
  description = "EBS volume size in GiB"
}
```

```hcl
# environments/dev/main.tf
module "application" {
  source = "../../modules/application"

  instance_type = "t4g.small"
  volume_type   = "gp2"
  volume_size   = 20
}
```

```hcl
# environments/prod/main.tf
module "application" {
  source = "../../modules/application"

  instance_type = "t4g.medium"
  volume_type   = "gp3"
  volume_size   = 30
}
```

추후 staging 환경이 필요해진다면, 아래와 같이 application 모듈을 재사용할 수 있습니다.

```hcl
# environments/staging/main.tf
module "application" {
  source = "../../modules/application"

  instance_type = "t4g.medium"
  volume_type   = "gp3"
  volume_size   = 30
}
```

### 5) output

output 블록을 이용하면 생성된 리소스의 정보를 다른 Terraform 모듈로 넘겨줄 수 있습니다.

예제로 살펴보겠습니다.

현재 EC2 생성 로직은 modules/application 경로에, RDS 생성 로직은 modules/database 경로에 모듈화 해놓은 상태입니다.

```
# 폴더 구조
.
├── environments
│   └── dev
│       ├── backend.tf
│       ├── main.tf
│       ├── provider.tf
│       └── variables.tf
└── modules
    ├── application
    │   ├── main.tf
    │   ├── outputs.tf
    │   └── variables.tf
    └── database
        ├── main.tf
        ├── outputs.tf
        └── variables.tf
    
```

`app_sg_id`는 application 모듈에서 생성된 SG(Security Group)의 id이고, 이 id는 SG 리소스가 생성된 뒤에 발급되는 id입니다.

```hcl
# modules/application/outputs.tf
output "app_sg_id" {
  description = "Security Group ID for the application instances"
  value       = aws_security_group.app_sg.id
}
```

앞에서 Terraform은 선언적이기 때문에 참조 관계를 자동으로 파악해서 순서대로 생성해준다고 했는데요. 여기서 그 장점이 드러납니다.

```hcl
# environments/dev/main.tf
module "application" {
  source = "../../modules/application"

  ...
}

module "database" {
  source = "../../modules/database"

  db_name                    = "courseitda_dev_db"
  db_instance_class          = "db.t3.micro"
  db_allocated_storage       = 20
  db_backup_retention_period = 7

  ...
  ingress_security_group_ids = [module.application.app_sg_id] # output 사용
}
```

Terraform은 SG 생성 → app_sg_id 추출 → RDS 생성 순서를 보장합니다.

다만, 주의해야 할 점이 있는데 모듈 간 순환 참조 문제가 발생할 수 있음을 유의해야 합니다.

예를 들어 지금 database 모듈이 application의 `app_sg_id`(application의 output)를 참조하고 있는데요, 동시에 application 모듈에서 database의 output을 참조하는 로직이 있다면, 순환 참조가 발생하게 됩니다. 이 경우 Terraform은 순환 참조 문제를 해결할 수 없기 때문에 에러가 발생합니다.

### 6) data

data 블록은 AWS AMI(Amazon Machine Image)와 같이 Terraform 코드로 생성하지 않은, 외부 리소스 정보를 읽기 전용(read only)으로 조회할 때 사용합니다.

```hcl
data "aws_ami" "app_ami" {
  most_recent = true
  owners      = ["099720109477"] # Canonical

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd-gp3/ubuntu-noble-24.04-arm64-server-*"]
  }
}

resource "aws_instance" "app_instance" {
  ami           = data.aws_ami.app_ami.id
  instance_type = var.instance_type
  ...
}
```

### 7) backend

Terraform에서 말하는 backend는 백엔드/프론트엔드의 백엔드와는 전혀 다른 의미입니다. 

backend는 Terraform **상태 파일(이하 상태 파일)을 저장하고 관리하는 위치**를 의미합니다.

**상태 파일**을 쉽게 설명하면, 현재 리소스들의 상태를 JSON 형식으로 기록한 파일로, Terraform의 멱등성을 보장하기 위해 사용되는 아주 아주 중요한 파일입니다.

상태 파일을 로컬에 저장하면 그 상태 파일을 가진 개발자만 리소스를 관리할 수 있기 때문에, 팀원 모두가 공유할 수 있게 원격 저장소(예: S3, Terraform Cloud)에 저장하도록 해야 합니다.

```hcl
terraform {
  backend "s3" {                                            # 상태 파일을 S3에 저장
    bucket       = "courseitda-backend-dev-terraform-state" # 상태 파일을 저장할 S3 버킷 이름
    key          = "backend.tfstate"                        # 파일 경로 (예: courseitda-backend-dev-terraform-state/backend.tfstate)
    region       = "ap-northeast-2"                         # S3 버킷이 위치한 리전
    use_lockfile = true                                     # 동시에 여러 사용자가 terraform apply 실행 시 충돌 방지
    encrypt      = true                                     # 서버 측 암호화 적용
  }
}
```

---

# ✅ Terraform 사용 중 겪었던 문제

Terraform을 프로젝트에 도입하면서 온전하게 해결하지 못한 문제가 두 가지 있었습니다. 🥲
부트스트랩 상태 파일 관리 문제와, Terraform에서 지원하지 않는 외부 서비스를 Terraform과 연계해서 사용해야 하는 문제였는데요.. 차례대로 설명드리겠습니다.

## 1) 부트스트랩 상태 파일 관리 문제

부트스트랩 상태 파일은 제가 임의로 붙인 이름이고, 명확히는 "백엔드 인프라 상태 파일을 저장하는 backend(S3 버킷)를 생성하는 Terraform 코드에 대한 상태 파일"을 말합니다.

설명이 너무 길기 때문에 "부트스트랩 상태 파일"이라 줄여 부르겠습니다.

예를 들면 백엔드 인프라의 상태 파일을 `courseitda-backend-dev-terraform-state` 라는 이름의 S3 버킷에 저장한다고 해보겠습니다.

이 `courseitda-backend-dev-terraform-state` S3 버킷을 만드는 Terraform 코드가 따로 존재하고, 그 코드에 대한 상태 파일은 수동으로 관리해야 합니다.

`courseitda-backend-dev-terraform-state` S3 버킷을 만드는 Terraform 코드는 bootstrap 경로에 두었습니다.

```
# 폴더 구조
.
├── bootstrap
│   ├── main.tf
│   ├── outputs.tf
│   ├── provider.tf
│   ├── terraform.tfstate
│   └── variables.tf
├── environments
└── modules
```

bootstrap/terraform.tfstate이 부트스트랩 상태 파일인데요. 이 상태 파일을 잃어버리게 되면 더이상 `courseitda-backend-dev-terraform-state` S3 버킷을 Terraform으로 관리할 수 없게 됩니다.

상태 파일을 로컬에서 관리하다 실수로 잃어버렸다면, 복원할 수 있는 살짝의 요령이 있습니다.

```bash
$ terraform import 'aws_s3_bucket.bucket["dev"]' courseitda-backend-dev-terraform-state
terraform import 'aws_s3_bucket_versioning.versioning["dev"]' courseitda-backend-dev-terraform-state
terraform import 'aws_s3_bucket_public_access_block.public_access["dev"]' courseitda-backend-dev-terraform-state
terraform import 'aws_s3_bucket_object_lock_configuration.object_lock_config["dev"]' courseitda-backend-dev-terraform-state

...
Import successful!                                                                                                                                     
                                                                                                                                                       
The resources that were imported are shown above. These resources are now in                                                                           
your Terraform state and will henceforth be managed by Terraform.
```

네... 제가 잃어버렸었는데요.

Terraform 코드의 리소스 이름을 하나하나 분석해가며 `terraform import` 명령어를 통해 tfstate 파일을 복원할 수 있었습니다.

Terraform Cloud를 사용하면, 이런 부트스트랩 상태 파일 관리 문제가 발생하다고 해서 Terraform Cloud로 옮기려고 생각 중입니다.

## 2) CloudFront와 커스텀 도메인 연결

다음은 CloudFront와 Gabia에서 구매한 도메인을 연결하면서 겪었던 문제입니다.

도메인 구매/유지 비용을 조금이라도 줄이고자, AWS Route 53이 아닌 Gabia를 이용했는데요. 하필 Terraform이 지원 중인 5,000개가 넘는 provider 중에 Gabia는 없었습니다. 😭

그래서 Terraform으로 완전한 자동화는 하지 못했고, Terraform 코드 실행과 수동 작업 🔨을 번갈아가며 진행해야 했습니다.

CloudFront와 Gabia 도메인을 연결하기 위해 진행했던 과정을 설명드리겠습니다.

먼저 Gabia에서 도메인 `example.com` 을 구매했다고 가정하겠습니다. (example.com은 실제 사용한 도메인이 아닌 예시입니다.)

목표는 `www.example.com`을 CloudFront 배포 도메인과 연결하는 작업입니다.

CloudFront에 Gabia 도메인을 연결하려면 ACM 인증서가 필요합니다. 그리고 `www.example.com`에 대한 ACM 인증서를 생성하기 위해, 도메인 소유권 검증 절차가 필요한데요.

처음 `terraform apply`를 실행하면 이 도메인 소유권 검증 절차가 완료되지 않아 실행 중간에 실패합니다.

```bash
$ terraform apply -auto-approve
...
module.static_website.data.aws_acm_certificate.issued_certificate: Still reading... [01m00s elapsed]
module.static_website.data.aws_acm_certificate.issued_certificate: Still reading... [01m10s elapsed]
module.static_website.data.aws_acm_certificate.issued_certificate: Still reading... [01m20s elapsed]
╷
│ Error: reading ACM Certificates: empty result
│ 
│   with module.static_website.data.aws_acm_certificate.issued_certificate,
│   on ../../modules/static-website/main.tf line 74, in data "aws_acm_certificate" "issued_certificate":
│   74: data "aws_acm_certificate" "issued_certificate" {
```

output 블록으로 출력한 값을 통해 ACM 인증서의 도메인 소유권 검증 상태가 `PENDING_VALIDATION` 임을 확인할 수 있습니다.

```bash
$ terraform output acm_certificate_status
"PENDING_VALIDATION"
```

이제 Gabia에 ACM 인증서 검증을 위한 레코드와 사용할 도메인 레코드를 추가해야 합니다.

output으로 출력된 레코드 정보를 확인하고,

```bash
$ terraform output acm_dns_validation_records
[
  {
    "name" = "xxxxx.www.example.com."
    "type" = "CNAME"
    "value" = "yyyyy.acm-validations.aws."
  },
]

$ terraform output cloudfront_domain
"zzzzz.cloudfront.net"
```

ACM 인증서 검증을 위한 레코드와 사용할 도메인 레코드를 직접 추가합니다.

![](https://velog.velcdn.com/images/heiler/post/1aa4d429-649b-460d-b3d9-f1999090626f/image.png)

![](https://velog.velcdn.com/images/heiler/post/aa534970-9723-4bef-a77f-322ce6dae041/image.png)

ACM 인증이 완료될 때까지 10분 정도 기다렸다가, Terraform 코드를 재실행해서 리소스 생성을 마무리했습니다.

```bash
$ terraform apply -auto-approve

module.static_website.aws_cloudfront_distribution.cdn: Creating...
module.static_website.aws_cloudfront_distribution.cdn: Creation complete after 3m36s [id= ]
module.static_website.data.aws_iam_policy_document.bucket_policy_document: Reading...
module.static_website.data.aws_iam_policy_document.bucket_policy_document: Read complete after 0s [id= ]
module.static_website.aws_s3_bucket_policy.bucket_policy: Creating...
module.static_website.aws_s3_bucket_policy.bucket_policy: Creation complete after 1s [id= ]

Apply complete! Resources: 2 added, 0 changed, 0 destroyed.  
```

Terraform으로 대부분의 인프라를 자동화할 수 있지만, Gabia처럼 provider를 지원하지 않는 서비스는 예외였습니다.

이 과정이 번거롭다 생각이 들면, 도메인 비용을 조금 더 지불하고 Route 53을 사용하면 될 것 같습니다. 😂

---

# 🎬 마무리하며

Terraform을 사용하며 몇 가지 어려움이 있었지만 실제 프로젝트에 도입해본 소감은 다음과 같습니다.

> 한 번도 Terraform을 사용해보지 않은 사람은 있어도, 한 번만 사용해본 사람은 없~~을 것 같~~다.
>

Terraform을 사용하면, 이미 한 번 구축해본 인프라를 전과 똑같이 만들어야 할 때, 웹 콘솔에서 복붙을 반복하는 작업에서 오는 스트레스를 줄일 수 있다는 것이 가장 큰 장점 같습니다.

또 사용해본 적 없는 새로운 리소스를 사용하게 된다 해도, 수동으로 한 번 구축해보고 Terraform으로 자동화할 수 있다면 바로 자동화할 것 같습니다 ㅎㅎ

다만, 제가 진행한 프로젝트는 백엔드 2명만으로 진행 중인 소규모 프로젝트고, 

레거시 인프라가 없는 깔끔한 상태(?)에서 인프라 구축을 시작했고, 

인프라를 담당한 인원이 저 혼자였기에 도입이 비교적 수월했다고 생각합니다.

아마 규모가 있는 팀 차원에서의 도입은 레거시 인프라의 상황과 러닝 커브 등의 진입장벽을 고려해 신중하게 결정해야 할 것 같습니다.

동작하는 온전한 Terraform 코드가 궁금하신 분들은 [코스잇다 레포지토리](https://github.com/courseitda/courseitda-backend/tree/develop/terraform)를 참고해주시면 감사하겠습니다.

감사합니다. 🙌🏻

# 레퍼런스
- [Terraform 공식 문서](https://developer.hashicorp.com/terraform)
- [HCL](https://developer.hashicorp.com/terraform/language)
- [Terraform - 모듈 구조](https://developer.hashicorp.com/terraform/language/modules/develop/structure)
- [Terraform - 상태 관리](https://developer.hashicorp.com/terraform/language/state)
- [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest)

+@
Infracost 이야기 추가
