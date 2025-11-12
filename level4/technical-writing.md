# 💬 들어가며

안녕하세요, 헤일러입니다.

아마 많은 백엔드 개발자분들이 이런 고민을 해보시지 않았을까 싶어요.

> 이거 서비스로 만들면 사람들이 쓸 것 같은데...

머릿속을 스쳐간 아이디어는 있지만, 화면 설계, 디자인, UI 구현 등 프론트엔드 구현이 장벽이 되어 상상에만 그치고 구현으로는 이어지지 못하는 경험이 한 번쯤은 있으실 거에요.
저 또한 그런 개발자 중 한 명이었습니다.

사람의 언어(자연어)로 AI에게 프롬프팅해서 코드를 생성하는 방식을 바이브 코딩이라고 하죠. 요즘은 Lovable, Bolt, v0와 같은 no-code AI를 이용하면, 바이브 코딩으로 빠르게 UI까지 갖춰진 프로토타입을 만들어볼 수 있습니다.

Lovable로 만든 프로토타입을 주변에 보여주고 사용성을 어느 정도 검증했다고 가정하겠습니다.

Lovable 플랫폼에서 동작하는 프로토타입은 새로운 기능 추가, 유지보수, 보안, 모니터링 등에 제약이 많습니다. 그래서 프로토타입을 실제 서비스로 발전시키기 위해서는 결국 자체 인프라 환경으로 마이그레이션해야 할 필요가 있습니다.

![](https://velog.velcdn.com/images/heiler/post/c7f898f5-e8b2-40cb-ac05-108e0d96203e/image.png)

추석 연휴 즈음부터 "코스잇다"라는 프로젝트를 시작했습니다.
이 프로젝트를 통해 `기획 → AI를 활용한 프로토타이핑 → 프로토타입을 AWS 인프라로 마이그레이션 및 런칭` 사이클을 직접 구축하고 개선해보는 작업을 진행해보고 있습니다.
이 사이클을 자동화하고 효율적으로 개선해서, 개발자가 기획과 비즈니스 로직을 구현하는 데 더 집중할 수 있는 환경을 만드는 게 목표입니다.

프로토타이핑은 코스잇다 팀만의 프롬프트 엔지니어링 전략을 세워 어느 정도 반복하고 개선할 수 있는 구조를 만들었습니다.
프로토타입을 자체 인프라로 마이그레이션하는 일만 남았는데요.
마이그레이션을 진행하던 중에 이런 의문이 들었습니다.

> 새로운 프로토타입을 만들 때마다 인프라를 매번 웹 콘솔에서 수동으로 반복해서 만들어야 할까?

운영/모니터링이 가능한 최소 비용의 인프라 구조는 어떤 프로토타입이든 비슷할 것이라 생각했고, 이미 설계해 놓은 인프라를 템플릿으로 만들어 쉽게 재현할 수 있는 방법으로 Terraform을 선택하게 됐습니다.

그리고 Terraform을 프로젝트에 도입하면서 "이 정도만 이해하고 있어도 누구든 충분히 Terraform을 사용할 수 있겠다" 느꼈던 내용들을 정리해 보았습니다.

Terraform을 처음 접하거나, 프로젝트에 이제 막 도입하려는 분들께 도움이 되길 바랍니다. ☺️

---    
# ✅ Terraform 핵심 개념

## Terraform이 무엇인가요?

> HashiCorp Terraform is an **infrastructure as code tool** that lets you define both cloud and on-prem resources in **human-readable configuration files** that you can version, reuse, and share.
>

공식 문서의 Terraform 첫 소개 문장입니다.

요약하면 Terraform은 **사람이 읽기 쉬운 설정 파일**로 인프라를 정의하고 관리하는 **IaC(Infrastructure As Code) 도구**입니다.

IaC는 영어 뜻 그대로 인프라를 코드로 정의하고 관리하는 방법론이에요.

코드를 실행하면 작성한 코드대로 인프라가 짠하고 만들어지는 거죠.

아래에서 대표적인 IaC 도구인 Terraform의 특징 3가지와 핵심 키워드 7가지를 본격적으로 소개하겠습니다.

## Terraform의 특징

### 1) 사람이 읽기 쉬운 언어(**human-readable)**

Terraform은 HCL(HashiCorp Configuration Language) 언어를 사용합니다.

HCL에서는 block, argument 구조로 코드를 작성하는데요.

![](https://velog.velcdn.com/images/heiler/post/c49c7bf9-d6a7-4d98-adb8-c9b94b0a088d/image.png)

HCL을 학습하면서 느낌 점은 사용하는 클라우드(예: AWS)에 대한 이해만 있다면, 해당 클라우드에서 사용할 리소스를 HCL문법으로 작성하는 것은 크게 어렵지 않다는 것입니다.

다만, HCL도 코드다 보니 추상화, 모듈 구조 설계가 필요합니다.

HCL 문법보다, 클라우드에 대한 이해가 필요한 점과 모듈 구조 설계를 해야한다는 점이 더 어렵고 큰 진입 장벽인 것 같습니다.

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

같은 Terraform 코드를 여러 번 실행한다고 해서 인프라가 계속 새롭게 만들어지지 않습니다.

현재 인프라 상태를 추적해서, 코드 변경에 의해 발생한 차이만 실행에 반영합니다.

## Terraform 핵심 키워드

### 1) provider

![](https://velog.velcdn.com/images/heiler/post/50ee0b15-000c-4819-b5c7-898859d6a488/image.png)

provider는 Terraform과 Target API(예: AWS) 사이를 연결해주는 플러그인 역할을 합니다.

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

익숙하지 않은 data, var와 같은 키워드들이 등장해 혼란스러울 수 있으실 텐데요. 자세한 내용은 아래에서 살펴보겠습니다.

### 3) variable

variable 블록은 같은 모듈(폴더) 내 다른 Terraform 코드에서 사용할 수 있는 변수를 정의합니다.

가령 같은 모듈에 `variables.tf`와 `main.tf`가 존재한다고 하겠습니다.

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

위 코드와 같이 `main.tf`에서 `variables.tf`에 정의한 변수를 사용할 수 있습니다.

별다른 조치 없이 variable에 default 값을 입력하지 않으면, Terraform 코드를 실행하는 시점에 콘솔로 사용자 입력을 받게 됩니다.

default 값을 variable 선언부에 두지 않고, 실행 시점에 외부에서 값을 주입해서 사용할 수 있는 방법이 여러 가지가 있는데요.

그중에 자주 사용되는 방법 중 하나가 module을 사용하는 방법입니다.

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

예를 들어 dev 환경과 prod 환경에서 EC2의 인스턴스 타입, 볼륨 타입, 볼륨 크기에만 차이를 두고 싶다면, 아래 코드와 같이 그 차이를 두고 싶은 값을 모듈에서 `variables.tf`로 분리해두고, 각 환경의 `main.tf`에서 다른 값을 주입하여 사용할 수 있습니다.

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

backend는 **Terraform 상태 파일(이하 상태 파일)을 저장하고 관리하는 위치**를 의미합니다.

**상태 파일**을 쉽게 설명하면, 현재 Terraform으로 관리중인 리소스들의 상태를 JSON 형식으로 기록한 파일로, Terraform의 멱등성을 보장하기 위해 사용되는 아주 아주 중요한 파일입니다.

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
부트스트랩 상태 파일 관리 문제와, Terraform에서 지원하지 않는 외부 서비스를 Terraform과 연계해서 사용해야 하는 문제였는데요. 차례대로 설명드리겠습니다.

## 1) 부트스트랩 상태 파일 관리 문제

부트스트랩 상태 파일은 제가 임의로 붙인 이름이고, 명확히는 "백엔드 인프라 상태 파일을 저장하는 S3 버킷을 생성하는 Terraform 코드에 대한 상태 파일"을 말합니다.

예를 들어 백엔드 인프라의 상태 파일을 `courseitda-backend-dev-terraform-state` 라는 이름의 S3 버킷에 저장한다고 해보겠습니다. 이 `courseitda-backend-dev-terraform-state` S3 버킷을 만드는 Terraform 코드가 따로 존재합니다.

그 코드에 대한 상태 파일을 앞으로 "부트스트랩 상태 파일"이라 부르겠습니다.

S3를 상태 파일의 원격 저장소로 사용하려면 이 부트스트랩 상태 파일을 수동으로 관리해야 했습니다.

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

`courseitda-backend-dev-terraform-state` S3 버킷을 만드는 Terraform 코드는 bootstrap 경로에 두었습니다.

`bootstrap/terraform.tfstate`가 바로 부트스트랩 상태 파일입니다.
이 파일을 잃어버리면 더 이상 `courseitda-backend-dev-terraform-state` S3 버킷을 Terraform으로 관리할 수 없습니다.

상태 파일을 로컬에서 관리하다 보면 실수로 잃어버릴 수도 있는데요.

복원할 수 있는 살짝의 요령이 있습니다.

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

네, 제가 실수로 백업하지 않아 잃어버린 적이 있었는데요.

조금 번거롭지만 `terraform import <리소스명>` 명령어로 리소스를 하나씩 tfstate 파일에 다시 가져와 복원할 수 있습니다.

Terraform Cloud를 사용하면 이런 부트스트랩 상태 파일 관리 문제를 줄일 수 있다고 해서,
현재는 원격 상태 저장소를 S3에서 Terraform Cloud로 이전하는 것을 고민하고 있습니다.

## 2) CloudFront와 커스텀 도메인 연결

다음은 CloudFront와 Gabia에서 구매한 도메인을 연결하면서 겪었던 문제입니다.

도메인 구매/유지 비용을 조금이라도 줄이고자, AWS Route 53이 아닌 Gabia를 이용했는데요. 하필 Terraform이 수천 개의 provider를 지원하고 있음에도, 그 안에 Gabia는 없었습니다. 😭

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

새로운 사이드 프로젝트를 시작할 때마다, 아이디어만 바뀌고 인프라는 템플릿으로 꺼내 쓰는 흐름을 만드는 것이 목표였는데요.

현재는 프론트엔드 + 백엔드 실행 배포 환경 자동화까지 진행된 상황이고, 다음 단계로 모니터링 시스템을 템플릿화하려 하고 있습니다.

인프라의 템플릿화를 위해 Terraform을 프로젝트에서 사용해 본 소감은 다음과 같습니다.

> 한 번도 Terraform을 사용해 보지 않은 사람은 있어도, 한 번만 사용해 본 사람은 없을 것이다.

Terraform을 사용하면 이미 한 번 구축해 본 인프라를 전과 똑같이 만들어야 할 때 웹 콘솔에서 복붙을 반복하는 작업에서 오는 스트레스를 줄일 수 있다는 것이 가장 큰 장점 같습니다.

단순한 예로는 리소스에 붙인 태그명을 일괄적으로 바꾸는 작업을 수동으로 한다면 리소스 개수만큼 웹 콘솔에서 수정해 주어야 하는데, Terraform을 사용한다면 코드 한 줄 수정으로 해결할 수 있습니다.

아마 앞으로 사용해 본 적 없는 새로운 클라우드 리소스를 사용하게 된다 해도,
수동으로 한 번 구축해 보고 Terraform으로 자동화할 수 있는지 바로 확인해 볼 것 같아요. 😋

다만, 제가 진행한 프로젝트는 백엔드 2명만으로 진행 중인 소규모 프로젝트고,
레거시 인프라가 없는 깔끔한 상태(?)에서 인프라 구축을 시작했고,
인프라 자동화를 담당한 인원이 저 혼자였기에 복잡한 권한 분리 작업이 따로 필요 없어
비교적 도입이 수월했다고 생각합니다.

규모가 있는 팀 차원에서의 Terraform 도입은 레거시 인프라의 상황과 러닝커브 등의 진입장벽을 고려해 신중하게 결정해야 할 것 같아요.

혹시 동작하는 전체 Terraform 코드가 궁금하신 분들은 [코스잇다 레포지토리](https://github.com/courseitda/courseitda-backend/tree/develop/terraform)를 참고해 주세요.

감사합니다. 🙌🏻

---

# Reference
- [Terraform 공식 문서 - Intro to Terraform](https://developer.hashicorp.com/terraform/intro)
- [Terraform 공식 문서 - HCL](https://developer.hashicorp.com/terraform/language)
- [Terraform 공식 문서 - Standard Module Structure](https://developer.hashicorp.com/terraform/language/modules/develop/structure)
- [Terraform 공식 문서 - State](https://developer.hashicorp.com/terraform/language/state)
- [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest)
- [10분 테코톡 - 헤일러의 Terraform으로 인프라 자동화하기](https://www.youtube.com/watch?v=57FJoXsbf2s)