**AWS VPC, ALB, Route53, ACM 모듈화 Terraform 구성 가이드**

Terraform을 사용해 AWS 인프라를 모듈화하여 손쉽게 관리하는 과정을 다뤄보려고 합니다. 이번 글에서는 AWS VPC, Application Load Balancer(ALB), Route53, ACM을 Terraform으로 모듈화하고 설치하는 과정까지 차근차근 설명해볼게요. 각 단계별로 모듈화하는 방법과 설정, 그리고 실제 인프라 구성에 대한 팁을 소개합니다.

뉴오프는 현재 도메인만 있고 인프라가 아무것도 없는 백지상태 입니다. 그 기준으로 작성된 글이기에 여타 다른 세팅방식과 약간의 차이가 존재할 수 있습니다. 

### 1. Terraform 설치 (CloudShell 기준)

먼저 AWS CloudShell에서 Terraform을 설치하는 과정부터 시작합니다. CloudShell을 사용하면 별도의 인프라 설정 없이 AWS 내에서 바로 사용할 수 있어 편리합니다.

물론 세션이 끊기면 설치된 Terraform이 초기화 되기 때문에 개인적으로는 그렇게 추천하지는 않습니다. 스토리지도 1G 제한이 있기도하고요. 그럼에도 선택한 이유는 로컬에서 이미 AWS Config가 다른 계정으로 세팅되어있기도 했고 Cloud9을 사용하려고 보니 Deprecated되서 CloudShell을 한번 사용해보았습니다.  

```bash
# Terraform 설치
cur사l -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
sudo apt-get update && sudo apt-get install terraform

# 설치 확인
terraform -v
```

CloudShell에서 위 명령어를 실행하여 Terraform을 설치하고, 버전이 잘 표시되는지 확인합니다.

### 2. 프로젝트 구조 설정

Terraform 프로젝트는 각 리소스를 모듈화하여 재사용성을 높이고, 유지보수를 쉽게 할 수 있도록 구조화합니다. 프로젝트 폴더는 아래와 같은 구조를 가지도록 설정합니다.

```
my-terraform-project/
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

VPC 모듈은 네트워크 리소스를 정의하는 핵심 모듈입니다. VPC, 서브넷, 라우팅 테이블, 인터넷 게이트웨이 등을 포함합니다.

`modules/vpc/main.tf`:

```hcl
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr_block
  tags = {
    Name = var.vpc_name
  }
}

# 퍼블릭 및 프라이빗 서브넷 생성
resource "aws_subnet" "public" {
  count = length(var.public_subnets)
  vpc_id = aws_vpc.main.id
  cidr_block = var.public_subnets[count.index].cidr_block
  availability_zone = var.public_subnets[count.index].availability_zone
  tags = {
    Name = "${var.vpc_name}-public-subnet-${count.index + 1}"
  }
}
```

VPC와 서브넷을 각각 생성하고, 각 리소스에 대해 이름 태그를 설정합니다. 이 모듈은 VPC 이름, CIDR 블록, 서브넷 설정 등을 변수로 받아 유연하게 사용할 수 있습니다.

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

