**AWS VPC, ALB, Route53, ACM 모듈화 Terraform 구성 가이드**

Terraform을 사용해 AWS 인프라를 모듈화하여 손쉽게 관리하는 과정을 다뤄보려고 합니다. 이번 글에서는 AWS VPC, Application Load Balancer(ALB), Route53, ACM을 Terraform으로 모듈화하고 설치하는 과정까지 차근차근 설명해볼게요. 각 단계별로 모듈화하는 방법과 설정, 그리고 실제 인프라 구성에 대한 팁을 소개합니다. 추후 뉴오프에 입사하게 될 동료분들께 도움이 되었으면 좋겠습니다.

뉴오프는 현재 도메인만 있고 인프라가 아무것도 없는 백지상태 입니다. 그 기준으로 작성된 글이기에 여타 다른 세팅방식과 약간의 차이가 존재할 수 있습니다. 

### 1. Terraform 설치 (CloudShell 기준)

먼저 AWS CloudShell에서 Terraform을 설치하는 과정부터 시작합니다. CloudShell을 사용하면 별도의 인프라 설정 없이 AWS 내에서 바로 사용할 수 있어 편리합니다.

물론 세션이 종료되면 설치된 Terraform이 초기화되기 때문에 개인적으로는 추천하지 않습니다. 또한 스토리지 용량에도 1GB 제한이 있습니다. 그럼에도 CloudShell을 선택한 이유는, 로컬 환경의 AWS Config가 다른 계정으로 설정되어 있었고, Cloud9이 Deprecated 되어 CloudShell을 사용하라는 추천 메시지가 떴기 때문입니다. 그래서 의식의 흐름대로 AWS CloudShell을 사용해 보았습니다.

```bash
# 패키지 설치 업데이트 및 필요한 도구 설치
sudo yum update -y
sudo yum install -y yum-utils

# HashiCorp 리포지토리 추가
sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo

# yum을 통해 테라폼을 설치
sudo yum install -y terraform

# 테라폼 설치 확인
terraform -v
```

CloudShell에서 위 명령어를 실행하여 Terraform을 설치한 후, 버전이 올바르게 표시되는지 확인합니다. 단, CloudShell은 세션이 종료되면 설치된 환경이 초기화되므로 다시 설치해야 하는 번거로움이 있습니다. 소스 설치를 통해 휘발되지 않는 홈 디렉토리에 Terraform을 위치시키는 것도 하나의 방법이지만, 그보다는 로컬 환경에서 사용하는 것이 더 나을 것입니다.


### 2. 프로젝트 구조 설정

Terraform 프로젝트는 각 리소스를 모듈화하여 재사용성을 높이고, 유지보수를 쉽게 할 수 있도록 구조화합니다. 프로젝트 폴더는 아래와 같은 구조를 가지도록 설정합니다.

```
newoff/
  |- main.tf
  |- variables.tf
  |- outputs.tf
  |- modules/
      |- vpc/
          |- main.tf
          |- variables.tf
          |- outputs.tf
      |- alb/
          |- main.tf
          |- variables.tf
          |- outputs.tf
      |- route53/
          |- main.tf
          |- variables.tf
          |- outputs.tf
      |- acm/
          |- main.tf
          |- variables.tf
          |- outputs.tf
```

각 서비스(VPC, ALB, Route53, ACM)를 모듈화하여 `modules` 폴더 내에 관리합니다. 이를 통해 반복되는 설정을 줄이고, 손쉽게 변경 사항을 적용할 수 있습니다.

### 3. VPC 모듈화

VPC 모듈은 네트워크 리소스를 정의하는 핵심 모듈입니다. 여기에는 VPC, 서브넷, 라우팅 테이블, 인터넷 게이트웨이 등이 포함됩니다. 저희는 Dev Network와 Production Network를 분리하여 사용하고 있으며, 이를 위해 2개의 VPC를 생성했습니다. 서브넷은 각각 프라이빗과 퍼블릭으로 나누어 2개의 가용영역에 배치하였고, NAT 게이트웨이도 개발/운영용으로 각각 두 개를 설정하여 EIP 역시 별도로 관리하고 있습니다. 현재는 VPC 엔드포인트로 S3만 설정한 상태입니다.

`modules/vpc/main.tf`:

```hcl
# VPC 리소스 생성
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr_block

  tags = {
    Name = var.vpc_name
  }
}

# 퍼블릭 서브넷 생성
resource "aws_subnet" "public" {
  count             = length(var.public_subnets)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.public_subnets[count.index].cidr_block
  availability_zone = var.public_subnets[count.index].availability_zone

  tags = {
    Name = "${var.vpc_name}-public-subnet-${var.public_subnets[count.index].availability_zone_name}"
  }
}

# 프라이빗 서브넷 생성
resource "aws_subnet" "private" {
  count             = length(var.private_subnets)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.private_subnets[count.index].cidr_block
  availability_zone = var.private_subnets[count.index].availability_zone

  tags = {
    Name = "${var.vpc_name}-private-subnet-${var.private_subnets[count.index].availability_zone_name}"
  }
}

# 인터넷 게이트웨이 생성
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "${var.vpc_name}-igw"
  }
}

# 기본 라우팅 테이블 생성
resource "aws_route_table" "default" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "${var.vpc_name}-default-rtb"
  }
}

# 기본 라우팅 테이블을 VPC의 기본으로 설정
resource "aws_main_route_table_association" "main_rt_assoc" {
  vpc_id         = aws_vpc.main.id
  route_table_id = aws_route_table.default.id
}

# 퍼블릭 라우팅 테이블 생성
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = {
    Name = "${var.vpc_name}-public-rt"
  }
}

# 프라이빗 라우팅 테이블 생성
resource "aws_route_table" "private" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_nat_gateway.main.id
  }

  tags = {
    Name = "${var.vpc_name}-private-rt"
  }
}

# 퍼블릭 서브넷과 퍼블릭 라우팅 테이블 연결
resource "aws_route_table_association" "public_association" {
  count         = length(var.public_subnets)
  subnet_id     = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

# 프라이빗 서브넷과 프라이빗 라우팅 테이블 연결
resource "aws_route_table_association" "private_association" {
  count         = length(var.private_subnets)
  subnet_id     = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private.id
}

# NAT 게이트웨이 생성
resource "aws_nat_gateway" "main" {
  allocation_id = aws_eip.main.id
  subnet_id     = aws_subnet.public[0].id

  tags = {
    Name = "${var.vpc_name}-nat-gw"
  }
}

# Elastic IP 생성 (NAT 게이트웨이용)
resource "aws_eip" "main" {
  domain = "vpc"

  tags = {
    Name = "${var.vpc_name}-nat-eip"
  }
}

# S3 VPC 엔드포인트 생성
resource "aws_vpc_endpoint" "s3" {
  vpc_id       = aws_vpc.main.id
  service_name = "com.amazonaws.${var.aws_region}.s3"

  route_table_ids = [
    aws_route_table.public.id,
    aws_route_table.private.id
  ]

  tags = {
    Name = "${var.vpc_name}-s3-endpoint"
  }
}
```

`modules/vpc/outputs.tf`:

```hcl
# VPC ID 출력
output "vpc_id" {
  description = "VPC의 ID"
  value       = aws_vpc.main.id
}

# 퍼블릭 서브넷 IDs 출력
output "public_subnet_ids" {
  description = "퍼블릭 서브넷의 IDs"
  value       = aws_subnet.public[*].id
}

# 프라이빗 서브넷 IDs 출력
output "private_subnet_ids" {
  description = "프라이빗 서브넷의 IDs"
  value       = aws_subnet.private[*].id
}
```

`modules/vpc/variables.tf`:

```hcl
# VPC CIDR 블록 변수
variable "vpc_cidr_block" {
  description = "VPC의 CIDR 블록 범위 지정"
  type        = string
}

# VPC 이름 변수
variable "vpc_name" {
  description = "VPC의 이름 지정"
  type        = string
}

# 퍼블릭 서브넷 정보 변수
variable "public_subnets" {
  description = "퍼블릭 서브넷의 CIDR 블록 및 가용 영역 정보"
  type = list(object({
    cidr_block        = string
    availability_zone = string
    availability_zone_name = string
  }))
}

# 프라이빗 서브넷 정보 변수
variable "private_subnets" {
  description = "프라이빗 서브넷의 CIDR 블록 및 가용 영역 정보"
  type = list(object({
    cidr_block        = string
    availability_zone = string
    availability_zone_name = string
  }))
}

# AWS 리전 변수 (모듈에서 S3 엔드포인트 생성에 필요)
variable "aws_region" {
  description = "AWS 리전 (예: ap-northeast-2)"
  type        = string
}
```

VPC와 서브넷을 각각 생성하고, 각 리소스에 대해 이름 태그를 설정합니다. 이 모듈은 VPC 이름, CIDR 블록, 서브넷 설정 등을 변수로 받아 유연하게 사용할 수 있습니다.

라우팅 테이블을 생성하고 서브넷에 연결한 후, NAT 게이트웨이와 EIP, 그리고 S3 VPC 엔드포인트까지 순차적으로 생성합니다.

`main.tf`:

```hcl
# AWS 제공자 설정
provider "aws" {
  region = var.aws_region # 리전을 변수로 설정
}

module "dev_vpc" {
  source          = "./modules/vpc"
  vpc_name        = "${var.project_name}-dev"
  vpc_cidr_block  = "10.0.0.0/16"
  aws_region      = var.aws_region
  public_subnets  = [
    {
      cidr_block        = "10.0.1.0/24"
      availability_zone = "${var.aws_region}${var.availability_zones[0]}"
      availability_zone_name = var.availability_zones[0]
    },
    {
      cidr_block        = "10.0.3.0/24"
      availability_zone = "${var.aws_region}${var.availability_zones[1]}"
      availability_zone_name = var.availability_zones[1]
    }
  ]
  private_subnets = [
    {
      cidr_block        = "10.0.2.0/24"
      availability_zone = "${var.aws_region}${var.availability_zones[0]}"
      availability_zone_name = var.availability_zones[0]
    },
    {
      cidr_block        = "10.0.4.0/24"
      availability_zone = "${var.aws_region}${var.availability_zones[1]}"
      availability_zone_name = var.availability_zones[1]
    }
  ]
}

module "prod_vpc" {
  source          = "./modules/vpc"
  vpc_name        = "${var.project_name}-prod"
  vpc_cidr_block  = "10.1.0.0/16"
  aws_region      = var.aws_region
  public_subnets  = [
    {
      cidr_block        = "10.1.1.0/24"
      availability_zone = "${var.aws_region}${var.availability_zones[0]}"
      availability_zone_name = var.availability_zones[0]
    },
    {
      cidr_block        = "10.1.3.0/24"
      availability_zone = "${var.aws_region}${var.availability_zones[1]}"
      availability_zone_name = var.availability_zones[1]
    }
  ]
  private_subnets = [
    {
      cidr_block        = "10.1.2.0/24"
      availability_zone = "${var.aws_region}${var.availability_zones[0]}"
      availability_zone_name = var.availability_zones[0]
    },
    {
      cidr_block        = "10.1.4.0/24"
      availability_zone = "${var.aws_region}${var.availability_zones[1]}"
      availability_zone_name = var.availability_zones[1]
    }
  ]
}
```

`variables.tf`:

```hcl
variable "project_name" {
  description = "프로젝트 이름 지정"
  type        = string
  default     = "newoff"
}

variable "aws_region" {
  description = "AWS 리전 (예: ap-northeast-2)"
  type        = string
  default     = "ap-northeast-2"
}

variable "availability_zones" {
  description = "가용 영역 목록"
  type        = list(string)
  default     = ["a", "c"]
}

variable "domain_name" {
  description = "도메인 이름"
  type        = string
  default     = "new-off.com"
}
```
루트 모듈에 **dev_vpc**,**prod_vpc** 를 선언하고 공통으로 사용하는 변수까지 정의 했다면 초기 VPC 설정은 완료입니다.
모듈 내에서 변수를 받아서 사용해야하기 때문에 variables 선언이 필요하고, 모듈간에 출력내용을 공유해야하는 상황이 있기 때문에 output까지 미리 정의해 두었습니다.

이제 실행 해보도록 하겠습니다.
```
terraform init
```
모듈이 추가 될때는 init을 해주셔야 합니다.
초기화가 문제없이 진행되었다면 실행계획을 살펴보도록 합시다.
```
terraform plan -target=module.prod_vpc -target=module.dev_vpc
```
저 같은 경우는 모듈별로 설치할 예정이기 때문에 target 옵션을 활용하도록 하겠습니다. target 옵션은 [공식 가이드](https://developer.hashicorp.com/terraform/cli/commands/plan)를 참고해보시면 좋습니다. (특히 destroy 사용시에는 target 옵션을 조심히 사용해야 합니다.)

```
terraform apply -target=module.prod_vpc -target=module.dev_vpc
```
실행계획에 문제가 없다면 apply로 설치를 시작합니다. 


### 4. ALB 모듈화

이번 글에서는 VPC-Route53-ALB-ACM 순서에 따라 진행하기 때문에 Route53 설정후에 ALB 설정을 시작합니다.
ALB 모듈은 VPC 내에서 트래픽을 분산시키기 위해 Load Balancer를 생성합니다. ALB와 관련된 보안 그룹도 함께 정의합니다.

`modules/alb/main.tf`:

```hcl
# ALB 보안 그룹 생성
resource "aws_security_group" "alb_sg" {
  name   = "${var.alb_name}-sg"
  vpc_id = var.vpc_id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "${var.alb_name}-sg"
  }
}

# Application Load Balancer (ALB) 생성
resource "aws_lb" "main" {
  name               = var.alb_name
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb_sg.id]
  subnets            = var.public_subnet_ids

  tags = {
    Name = var.alb_name
  }
}
```

`modules/alb/outputs.tf`:

```hcl
# ALB ID 출력
output "alb_zone_id" {
  description = "생성된 ALB의 ID"
  value       = aws_lb.main.zone_id
}

# ALB DNS 이름 출력
output "alb_dns_name" {
  description = "생성된 ALB의 DNS 이름"
  value       = aws_lb.main.dns_name
}
```

`modules/alb/variables.tf`:

```hcl
# ALB 이름 변수
variable "alb_name" {
  description = "ALB의 이름"
  type        = string
}

# VPC ID 변수
variable "vpc_id" {
  description = "ALB를 생성할 VPC ID"
  type        = string
}

# 퍼블릭 서브넷 ID 목록 변수
variable "public_subnet_ids" {
  description = "퍼블릭 서브넷 ID 목록"
  type        = list(string)
}

# AWS 리전 변수
variable "aws_region" {
  description = "AWS 리전"
  type        = string
}
```

보안 그룹 설정 및 서브넷 연결을 통해 ALB를 생성합니다.
그리고 Root모듈에 ALB를 선언해줍니다. 

`main.tf`:

```hcl
# 루트 모듈에서 ALB 모듈을 호출
module "alb" {
  source            = "./modules/alb"
  alb_name          = "${var.project_name}-prod-alb"
  vpc_id            = module.prod_vpc.vpc_id
  public_subnet_ids = module.prod_vpc.public_subnet_ids
  aws_region        = var.aws_region
}
```

이제 실행 해보도록 하겠습니다.
```
terraform init
```

모듈이 추가 될때는 init을 해주셔야 한다고 말씀드렸는데요. 저는 단계적인 세팅을 해나가고 있는 도중이라 이런것이고 문제가 없다면 이런 과정들을 반복할 필요가 없습니다.
초기화가 문제없이 진행되었다면 실행계획/설치를 수행하시면 됩니다.
```
terraform plan -target=module.alb
terraform apply -target=module.alb
```

### 5. Route53 모듈화

Route53 모듈은 도메인과 관련된 호스팅 존 및 레코드를 설정합니다. VPC와의 연결을 통해 프라이빗 및 퍼블릭 호스팅 존을 정의합니다.

`modules/route53/main.tf`:

```hcl
# Public Hosted Zone 생성
resource "aws_route53_zone" "public" {
  name = var.domain_name
  comment = "Public Hosted Zone for ${var.domain_name}"
}

# Private Hosted Zone 생성
resource "aws_route53_zone" "private_zone" {
  name          = var.domain_name
  vpc {
    vpc_id     = var.vpc_ids[0] # 첫 번째 VPC와 연결
    vpc_region = var.aws_region
  }
  vpc {
    vpc_id     = var.vpc_ids[1] # 두 번째 VPC와 연결
    vpc_region = var.aws_region
  }
  comment       = "Private Hosted Zone for ${var.domain_name}"

  tags = {
    Name = "${var.domain_name}-private-zone"
  }
}

# Route53 A 레코드 생성 (www)
resource "aws_route53_record" "default-www" {
  zone_id = aws_route53_zone.public.zone_id
  name    = var.domain_name
  type    = "A"
  alias {
    name                   = "www.${var.domain_name}"
    zone_id                = aws_route53_zone.public.zone_id
    evaluate_target_health = false
  }
}

# Route53 A 레코드 생성 (www) - ALB와 연결
resource "aws_route53_record" "www" {
  zone_id = aws_route53_zone.public.zone_id
  name    = "www.${var.domain_name}"
  type    = "A"

  alias {
    name                   = var.alb_dns_name
    zone_id                = var.alb_zone_id
    evaluate_target_health = false
  }
}

resource "aws_route53_record" "mx_record" {
  zone_id = aws_route53_zone.public.zone_id
  name    = var.domain_name
  type    = "MX"
  ttl     = 300
  records = [
    "10 ASPMX.L.GOOGLE.COM.",
    "20 ALT1.ASPMX.L.GOOGLE.COM.",
    "30 ALT2.ASPMX.L.GOOGLE.COM.",
    "40 ASPMX2.GOOGLEMAIL.COM.",
    "50 ASPMX3.GOOGLEMAIL.COM."
  ]
}
```

`modules/route53/outputs.tf`:

```hcl
output "public_hosted_zone_id" {
  description = "The ID of the public hosted zone"
  value       = aws_route53_zone.public.id
}

output "private_hosted_zone_id" {
  description = "The ID of the private hosted zone"
  value       = aws_route53_zone.private_zone.id
}
```

`modules/route53/variables.tf`:

```hcl
variable "domain_name" {
  description = "The domain name for the Route53 hosted zone"
  type        = string
}

# VPC ID 목록 변수
variable "vpc_ids" {
  description = "Route53 Hosted Zone에 연결할 VPC ID 목록"
  type        = list(string)
}

# AWS 리전 변수 (모듈에서 S3 엔드포인트 생성에 필요)
variable "aws_region" {
  description = "AWS 리전 (예: ap-northeast-2)"
  type        = string
}

# ALB DNS 이름 변수
variable "alb_dns_name" {
  description = "ALB의 DNS 이름"
  type        = string
}

# ALB Hosted Zone ID 변수
variable "alb_zone_id" {
  description = "ALB의 호스팅 존 ID"
  type        = string
}
```
백지 상태에서 설정하고 있기 때문에 당연하게도 Route53의 호스트존도 없습니다. 2개의 호스트존을 생성하고 A Record, MX Record 를 설정합니다. MX Record는 필요에 따라 진행하면 되겠습니다.
저희의 경우에는 www A 레코드를 ALB와 연결해야하기 때문에 alias 설정에 alb 관련 내용이 포함되어있습니다.

Private zone이 사실 고민되는 부분인데 EKS + [istio](https://istio.io/)를 사용할 예정입니다. 클러스터 내부 서비스 간의 통신은 Istio와 Kubernetes 기본 DNS만으로 충분히 처리할 수 있으므로, Route 53 Private Hosted Zone이 반드시 필요하지는 않는데 계획과 달라질수 있어서 우선은 보편적인 형태로 구성했습니다. 

여기까지 되었으면 루트에 모듈을 추가합니다.

`main.tf`:

```hcl
module "route53" {
  source       = "./modules/route53"
  domain_name  = var.domain_name
  vpc_ids      = [module.dev_vpc.vpc_id, module.prod_vpc.vpc_id]
  aws_region   = var.aws_region
  alb_dns_name = module.alb.alb_dns_name
  alb_zone_id  = module.alb.alb_zone_id
}
```

이제 실행 해보도록 하겠습니다.
```
terraform init
terraform plan -target=module.route53
terraform apply -target=module.route53
```

### 6. ACM 모듈화

ACM 모듈은 도메인 검증을 위한 인증서를 생성합니다. Route53 레코드를 사용해 DNS 검증을 설정합니다.

`modules/acm/main.tf`:

```hcl
# ACM 인증서 생성
resource "aws_acm_certificate" "cert" {
  domain_name       = var.domain_name
  validation_method = "DNS" # DNS 검증 방식 선택

  # 추가 도메인 등록 (와일드카드 포함)
  subject_alternative_names = var.alternative_names

  tags = {
    Name = "${var.domain_name}-certificate"
  }
}

# DNS 검증을 위한 Route 53 레코드 생성 (와일드카드가 없는 도메인만)
resource "aws_route53_record" "cert_validation" {
  for_each = {
    for dvo in aws_acm_certificate.cert.domain_validation_options :
    dvo.domain_name => {
      name   = dvo.resource_record_name
      type   = dvo.resource_record_type
      record = dvo.resource_record_value
    } if !startswith(dvo.domain_name, "*.")
  }

  zone_id = var.route53_zone_id
  name    = each.value.name
  type    = each.value.type
  ttl     = 60
  records = [each.value.record]
}

resource "aws_acm_certificate_validation" "cert_validation" {
  certificate_arn         = aws_acm_certificate.cert.arn
  validation_record_fqdns = [for record in aws_route53_record.cert_validation : record.fqdn]
}
```

`modules/acm/outputs.tf`:

```hcl
output "certificate_arn" {
  description = "생성된 ACM 인증서 ARN"
  value       = aws_acm_certificate.cert.arn
}

output "validation_status" {
  description = "ACM 인증서 검증 상태"
  value       = aws_acm_certificate_validation.cert_validation.id
}
```

`modules/acm/variables.tf`:

```hcl
variable "domain_name" {
  description = "인증서에 사용할 기본 도메인 이름"
  type        = string
}

variable "route53_zone_id" {
  description = "Route53 호스팅 존 ID"
  type        = string
}

variable "alternative_names" {
  description = "추가로 인증할 서브 도메인들 (예: *.example.com)"
  type        = list(string)
  default     = []
}

variable "aws_region" {
  description = "AWS 리전"
  type        = string
}
```

ACM 을 생성하면 CNAME을 Route53에 등록하고 검증하는 단계를 거치기 때문에 시간이 조금 더 걸립니다.
살짝 조심해야할게 와일드카드가 포함된 도메인까지 인증서가 발급되는데 2개의 CNAME이 생성됩니다. 
2개를 모두 CNAME 레코드로 등록 하면 좋겠지만 동일한 value 값으로 나오기 때문에 그중 한개만 등록하지 않으면 오류가 발생합니다.

이제 루트모듈에 적용하겠습니다.

`main.tf`:

```hcl
# SSL 보안 인증서 
module "acm" {
  source            = "./modules/acm"
  domain_name       = var.domain_name
  aws_region        = var.aws_region
  route53_zone_id   = module.route53.public_hosted_zone_id
  alternative_names = ["*.${var.domain_name}"]
}
```

위에서 반복된 init,plan,apply 삼형제 실행해주시면 되겠습니다.


### 결론

이번 글에서는 AWS 인프라를 Terraform으로 모듈화하여 VPC, ALB, Route 53, ACM을 설정하는 방법을 다뤘습니다. 모듈화를 통해 코드의 재사용성을 높이고, 각 구성 요소를 독립적으로 관리함으로써 유지보수를 쉽게 할 수 있습니다.

과거 온프레미스 환경에서 약 60여 개의 서버와 인프라를 AWS로 마이그레이션한 경험이 있습니다. 당시 인프라의 히스토리 관리가 너무 어려워 작은 변경도 큰 부담이었습니다. DNS 레코드를 수정하는 일조차 상당한 부담이었고, 보안 그룹의 규칙도 설명이 부족해 의미를 파악하는 데 많은 시간이 소모되었습니다.

이런 가시성 부족은 단순히 유지보수를 어렵게 만드는 것을 넘어, 프로젝트의 진행과 품질에 심각한 악영향을 끼쳤습니다. 변경 사항을 신속하게 적용하기 어렵고, 오류가 발생했을 때 문제를 진단하고 해결하는 속도가 크게 떨어졌습니다. 결국, 이러한 상황은 비효율성과 높은 리스크를 초래했습니다.

이러한 문제를 해결하기 위해 현재 Terraform을 도입하여 인프라를 코드로 관리하려고 준비 중입니다. 이를 통해 모든 구성 요소의 상태를 명확하게 가시화하고, 인프라 변경 내역을 코드로서 추적하여 더 안정적이고 일관성 있는 운영이 가능할 것으로 기대됩니다. 이는 인프라의 신뢰성과 가용성을 크게 높이는 데 기여할 것입니다.

인프라 관리의 핵심은 변경 사항에 대한 두려움을 없애는 것입니다. Terraform과 모듈화를 통해 인프라의 모든 부분이 문서화된 코드로 표현되므로, 수정과 확장이 보다 안전하고 예측 가능해졌습니다. 이 방식은 AWS 인프라를 효율적이고 체계적으로 운영하는 데 강력한 도구임을 보여주었습니다.

Terraform을 도입하며 느낀 점은, 실제 콘솔에서 UI로 구성하는 것보다 더 빠르게 인프라를 적용할 수 있었고, 복잡해 보이지만 실제로는 그리 어렵지 않다는 점이었습니다.

추후 EKS를 도입할 계획인데, 잦은 업데이트로 인해 클러스터 업그레이드 전략이 필요할 것입니다. 이때 인프라를 코드로 관리하는 방식이 준비되어 있다면 많은 시간을 절약할 수 있을 것입니다.


