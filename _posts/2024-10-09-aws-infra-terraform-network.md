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
```
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
```
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


### 4. ALB 모듈화

ALB 모듈은 VPC 내에서 트래픽을 분산시키기 위해 Load Balancer를 생성합니다. ALB와 관련된 보안 그룹도 함께 정의합니다.

`modules/alb/main.tf`:

```hcl
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

보안 그룹 설정 및 서브넷 연결을 통해 ALB를 생성합니다.

### 5. Route53 모듈화

Route53 모듈은 도메인과 관련된 호스팅 존 및 레코드를 설정합니다. VPC와의 연결을 통해 프라이빗 및 퍼블릭 호스팅 존을 정의합니다.

`modules/route53/main.tf`:

```hcl
resource "aws_route53_zone" "public" {
  name    = var.domain_name
  comment = "Public Hosted Zone for ${var.domain_name}"
}

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
```

도메인 네임과 ALB를 연결하여 www 레코드를 설정합니다.

### 6. ACM 모듈화

ACM 모듈은 도메인 검증을 위한 인증서를 생성합니다. Route53 레코드를 사용해 DNS 검증을 설정합니다.

`modules/acm/main.tf`:

```hcl
resource "aws_acm_certificate" "cert" {
  domain_name       = var.domain_name
  validation_method = "DNS"
  subject_alternative_names = [
    "*.${var.domain_name}"
  ]
}

resource "aws_route53_record" "cert_validation" {
  for_each = {
    for dvo in aws_acm_certificate.cert.domain_validation_options : dvo.domain_name => {
      name   = dvo.resource_record_name
      type   = dvo.resource_record_type
      record = dvo.resource_record_value
    }
  }

  zone_id = var.route53_zone_id
  name    = each.value.name
  type    = each.value.type
  ttl     = 60
  records = [each.value.record]
}
```

### 7. 루트 모듈 설정

각 모듈을 루트에서 호출하여 인프라를 구성합니다. 예를 들어 VPC, ALB, Route53, ACM 모듈을 연결합니다.

`main.tf`:

```hcl
module "dev_vpc" {
  source          = "./modules/vpc"
  vpc_name        = "${var.project_name}-dev"
  vpc_cidr_block  = "10.0.0.0/16"
  public_subnets  = var.public_subnets
  private_subnets = var.private_subnets
}

module "prod_alb" {
  source            = "./modules/alb"
  alb_name          = "${var.project_name}-prod-alb"
  vpc_id            = module.dev_vpc.vpc_id
  public_subnet_ids = module.dev_vpc.public_subnet_ids
}

module "route53" {
  source       = "./modules/route53"
  domain_name  = var.domain_name
  alb_dns_name = module.prod_alb.dns_name
  alb_zone_id  = module.prod_alb.zone_id
}
```

### 8. 변수 설정

`variables.tf`를 사용해 재사용 가능한 변수들을 설정합니다.

`variables.tf`:

```hcl
variable "project_name" {
  description = "프로젝트 이름"
  type        = string
}

variable "domain_name" {
  description = "Route53 도메인 이름"
  type        = string
}

variable "public_subnets" {
  description = "퍼블릭 서브넷 목록"
  type = list(object({
    cidr_block        = string
    availability_zone = string
  }))
}
```

### 결론

이번 글에서는 AWS 인프라를 Terraform으로 모듈화하여 VPC, ALB, Route53, ACM을 설정하는 방법을 다뤘습니다. 모듈화를 통해 코드의 재사용성을 높이고, 각 구성 요소를 독립적으로 관리함으로써 유지보수를 쉽게 할 수 있습니다. 이를 블로그에 올려서 많은 분들이 모듈화된 인프라 구성을 손쉽게 따라해볼 수 있도록 해보세요.

이후 더 궁금한 점이나 추가적인 도움이 필요하면 언제든지 말씀해주세요!

