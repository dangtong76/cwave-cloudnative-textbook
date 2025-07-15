---
title: "🤖 8. CICD 기본"
weight: 8
date: 2025-03-18
draft: false
---
---
## 구축을 위한 디자인 컨셉

  {{< embed-pdf url="/cwave-cloudnative-textbook/pdfs/codedeploy.pdf" >}}

---
## 1. Simple Web 무식하게 배포하기
---

Simple Web 은 정적 웹 페이지로 구성된 프로젝트입니다. 이번장에서는 가장 단순한 현태의 애플리케이션을 AWS 에 배포하는 파이프라인을 구성합니다.

파이프라인 구성시에 사용되는 **기술 스택**은 아래와 같습니다.

- Terraform
- GitHub Actions
- AWS S3
- AWS CodeDeploy

### 1. 프로젝트 포크 하기

1. GitHub 에서 프로젝트 포크
   
   GitHub [simple-web](https://github.com/dangtong76/simple-web) 에서 프로젝트 포크 → 포크 버튼 클릭 → 포크 이름 입력 → 포크 생성

2. gh 명령어를 이용한 포크
   
   ```bash
   gh auth status
   
   gh auth switch # 필요 하다면 수행 
   
   gh repo fork https://github.com/dangtong76/simple-web.git

   Would you like to clone the fork? Yes # 클론까지 한번에

   # git clone  https://github.com/<your-github-id>/simple-web.git simple-web
   ```


3. 워크플로우 디렉토리 생성
   
   ```bash
   mkdir -p .github/workflows
   mkdir -p xinfra/aws-ec2-single
   ```

5. Git 브랜치 표시 설정
  
    vi ~/.bashrc 
   ```bash
   # .bashrc 파일에 추가
   parse_git_branch() {
     git branch 2> /dev/null | sed -e '/^[^*]/d' -e 's/* \(.*\)/ (\1)/'
   }
   
   # Git 브랜치를 컬러로 표시
   export PS1="\u@\h \[\033[32m\]\w\[\033[33m\]\$(parse_git_branch)\[\033[00m\] $ "

   ```
    설정 적용하기
   ```bash
   source ~/.bashrc
   ```

### 2. Terraform 코드 생성
EC2 인스턴스 생성
보안 그룹 추가 (웹서비스 80, ssh 배포 22, 외부접속)
퍼블릭 IP 주소 추가 (서비스위한 공인IP)
ssh 배포를 위한 키 생성

1. 600_ec2.tf **신규작성** (infra/cwave-aws-eks/600_ec2.tf)

    ```hcl
    # TLS 프라이빗 키 생성 (공개 키 포함)
    resource "tls_private_key" "example" {
        algorithm = "RSA"
        rsa_bits  = 2048
    }

    # AWS에서 키 페어 생성
    resource "aws_key_pair" "ec2_key" {
        key_name   = "ec2-key" # AWS에서 사용할 키 페어 이름
        public_key = tls_private_key.example.public_key_openssh
    }

    # EC2 인스턴스 생성
    resource "aws_instance" "nginx_instance" {
        ami             = "ami-08b09b6acd8d62254" # Amazon Linux 2 AMI (리전별로 AMI ID가 다를 수 있음)
        instance_type   = "t2.micro"
        key_name        = aws_key_pair.ec2_key.key_name # AWS에서 생성한 SSH 키 적용
        security_groups = [aws_security_group.nginx_sg.name]

        # user_data 변경시 무조건 인스턴스를 다시 만들게 하기 위한 옵셤
        metadata_options {
            http_tokens = "required"
        }
        # 인스턴스를 다시 만들때 빠르게 교체 하기 위한 옵션
        lifecycle {
            create_before_destroy = true
        }

        # EC2 시작 시 Nginx 설치 및 실행을 위한 User Data
        user_data = <<-EOF
                    #!/bin/bash
                    yum update -y
                    amazon-linux-extras install nginx1 -y
                    systemctl start nginx
                    systemctl enable nginx
                    EOF
        tags = {
        Name = "nginx-server"
        }
    }


    # 출력: EC2 인스턴스의 퍼블릭 IP 주소
    output "nginx_instance_public_ip" {
        value       = aws_instance.nginx_instance.public_ip
        description = "Public IP of the Nginx EC2 instance"
    }

    # 출력: SSH 접속에 사용할 Private Key
    output "ssh_private_key_pem" {
        value       = tls_private_key.example.private_key_pem
        description = "Private key for SSH access"
        sensitive   = true
    }
    ```
2. 200_sg.tf **추가작성** (infra/cwave-aws-eks/200_sg.tf)
    ```hcl
    # 보안 그룹 설정: SSH(22) 및 HTTP(80) 트래픽 허용
    resource "aws_security_group" "nginx_sg" {
        name_prefix = "nginx-sg"

        ingress {
        description = "Allow SSH"
        from_port   = 22
        to_port     = 22
        protocol    = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
        }

        ingress {
        description = "Allow HTTP"
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
    }
    ```

3. 실행
    ```bash
    terraform init
    terraform plan
    terraform apply
    ```
4. .gitignore 파일 생성 (이미 있으면 SKIP)

    ```bash
    .terraform
    .terraform.lock.hcl
    terraform.tfstate
    .DS_Store
    terraform.tfstate.backup
    ```
5. 리포지토리 동기화 
    ```bash
    git add .
    git commit -am "add terraform code"
    git push origin main
    ```

### 3. Github Secret 만들기

- Secret 목록
  
  - IAM User Credential 에 서 생성
    
    - AWS_ACCESS_KEY_ID
    - AWS_SECRET_ACCESS_KEY
  
  - AWS_HOST (EC2 console 에서 복사)
  
  - AWS_KEY (terraform 명령으로 output 출력)
    
    ```yaml
    # 출력결과 복사해서 입력
    terraform output ssh_private_key_pem
    ```
  
  - AWS_USER : ec2-user (EC2 인스턴스의 OS 계정)

### 4. Github Actions 워크플로우 생성

2. 파일명 : .github/workflows/simple-web-ec2-workflow.yaml 작성
      ```yaml
      name: Deploy Simple-Web to AWS EC2 using AWS CLI
      on:
        push:
          branches:
            - main
      jobs:
        deploys:
          runs-on: ubuntu-latest
          steps:
            - name: 1.소스코드 가져오기
              uses: actions/checkout@v3
            - name: 2.AWS 접속정보 설정
              uses: aws-actions/configure-aws-credentials@v3
              with:
                aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
                aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
                aws-region: ap-northeast-2
            - name: 2.SSH키 설정
              uses: webfactory/ssh-agent@v0.5.3
              with:
                ssh-private-key: ${{ secrets.AWS_KEY}}
            - name: 3.파일 목록보기
              run: |
                ssh -o StrictHostKeyChecking=no ${{ secrets.AWS_USER }}@${{ secrets.AWS_HOST }}  "ls -al /home/${{ secrets.AWS_USER }}"
            - name: 4.파일 서버로 복사
              run: |
                ssh -o StrictHostKeyChecking=no ${{ secrets.AWS_USER }}@${{ secrets.AWS_HOST }}  "sudo chown -R ec2-user:ec2-user /usr/share/nginx/html"
                scp -o StrictHostKeyChecking=no -r ./* ${{ secrets.AWS_USER }}@${{ secrets.AWS_HOST }}:/usr/share/nginx/html
            - name: Restart Web Server
              run: |
                ssh -o StrictHostKeyChecking=no ${{ secrets.AWS_USER }}@${{ secrets.AWS_HOST }} "sudo systemctl restart nginx || sudo systemctl restart httpd"
      ```
3. 리포지토리 동기화
  
   ```bash
   git add .
   git commit -am "add github actions workflow"
   git push origin start
   ```

4. 워크플로우 실행 결과 확인

   - http://<EC2 인스턴스의 퍼블릭 IP 주소> 에 접속하여 웹 페이지 확인
    
    

---
## 2. Simple Web 현명하게 배포하기
### 1. EC2용 IAM 역할 생성하기

1. EC2용 IAM 역할 생성하기
   AWS 콘솔에서 : IAM → 역할 → 역할생성
  {{< figure src="/cwave-cloudnative-textbook/images/1-iam.png" alt="IAM 이미지" class="img-fluid" width="80%">}}

2. 역할 생성 화면에서 아래와 같이 [ AWS 서비스 | 서비스 또는 사용사례 = EC2 |  사용사례 = EC2 ]  선택 → 다음
  {{< figure src="/cwave-cloudnative-textbook/images/2-iam.png" alt="IAM 이미지" class="img-fluid" width="80%">}}

3. AWSCodeDeployFullAccess  및 AmazonS3FullAccess 정책 추가 → 다음
  {{< figure src="/cwave-cloudnative-textbook/images/3-iam.png" alt="IAM 이미지" class="img-fluid" width="80%">}}

4. 역할이름 : simple-web-ec2-deploy-role → 다음
  {{< figure src="/cwave-cloudnative-textbook/images/4-iam.png" alt="IAM 이미지" class="img-fluid" width="80%">}}

### 2. CodeDeploy용 IAM 역할 생성

2. AWS 콘솔에서 : IAM → 역할 → 역할생성
  역할생성 화면에서 : [AWS 서비스 | 서비스 또는 사용사례 = CodeDeploy  | 사용사례 = CodeDeploy ] 
  {{< figure src="/cwave-cloudnative-textbook/images/5-iam.png" alt="IAM 이미지" class="img-fluid" width="80%">}}

2. 내용 확인하고 → 다음
  {{< figure src="/cwave-cloudnative-textbook/images/6-iam.png" alt="IAM 이미지" class="img-fluid" width="80%">}}

2. 역할 이름 : simple-web-codedeploy-role 입력 → 역할생성 버튼 클릭
  {{< figure src="/cwave-cloudnative-textbook/images/7-iam.png" alt="IAM 이미지" class="img-fluid" width="80%">}}

### 3. Terrafrom 코드로 EC2 인스턴스 수정
코드 에이전트 설치를 위한 Ruby 를 설치 합니다.
Codedeploy Agent를 설치 후 서비스를 시작합니다.

1.  600_ec2.tf **수정** (infra/cwave-aws-eks/600_ec2.tf)
    ```terraform
    resource "aws_instance" "nginx_instance" {
    subnet_id = aws_subnet.dangtong-vpc-public-subnet["a"].id
    ami             = "ami-08b09b6acd8d62254" # Amazon Linux 2 AMI (리전별로 AMI ID가 다를 수 있음)
    instance_type   = "t2.micro"
    key_name        = aws_key_pair.ec2_key_pair.key_name # AWS에서 생성한 SSH 키 적용
    vpc_security_group_ids = [aws_security_group.nginx_sg.id]
    iam_instance_profile   = aws_iam_instance_profile.ec2_profile.name
     
    # user_data 변경시 무조건 인스턴스를 다시 만들게 하기 위한 옵셤
    metadata_options {
        http_tokens = "required"
    }
    # 인스턴스를 다시 만들때 빠르게 교체 하기 위한 옵션
    lifecycle {
        create_before_destroy = true
    }

    # EC2 시작 시 Nginx 설치 및 실행을 위한 User Data
    user_data = <<-EOF
                    #!/bin/bash
                    yum update -y

                    # Ruby 설치
                    yum install -y ruby wget

                    # CodeDeploy Agent 설치
                    cd /home/ec2-user
                    wget https://aws-codedeploy-ap-northeast-2.s3.ap-northeast-2.amazonaws.com/latest/install
                    chmod +x ./install
                    ./install auto

                    # CodeDeploy Agent 서비스 시작
                    systemctl start codedeploy-agent
                    systemctl enable codedeploy-agent

                    # nginx 설치
                    amazon-linux-extras install nginx1 -y
                    systemctl start nginx
                    systemctl enable nginx
                    EOF
    tags = {
        Name        = "nginx-server"
        Environment = "Production"
    }
    }
    ```

### 4. EC2 인스턴스에서 역할 연결하기

EC2 → 인스턴스  → 인스턴스 ID  → 작업  → 보안  → IAM 역할수정  → IAM 역할 선택 → simple-web-ec2-deploy-role 선택  → IAM 역할 업데이트

  {{< figure src="/cwave-cloudnative-textbook/images/8-iam.png" alt="IAM 이미지" class="img-fluid" width="80%" >}}

### 5. S3 버킷 만들기
  S3 화면에서  → 버킷 만들기 클릭
  버킷이름 : simple-web-content 
  나머지는 모두 Default 로 생성

  {{< figure src="/cwave-cloudnative-textbook/images/9-s3.png" alt="S3 이미지" class="img-fluid" width="80%">}}

### 6. CodeDeploy 애플리케이션 만들기
  codedeploy  → 애플리케이션  → 애플리케이션 생성
  애플리케이션 이름 : simple-web-content 
  {{< figure src="/cwave-cloudnative-textbook/images/10-codedeploy.png" alt="CodeDeploy 이미지" class="img-fluid" width="80%">}}

### 7. CodeDeploy 배포 그룹 만들기
- 배포 그룹 생성

  codedeploy  → 애플리케이션   → 배포 그룹 생성

  배포그룹이름입력 : simple-web-deploy-group

  서비스역할 : simple-web-codedeploy-role 

  {{< figure src="/cwave-cloudnative-textbook/images/11-codedeploy-group.png" alt="CodeDeploy 이미지" class="img-fluid" width="80%">}}
  
- 계속

  애플리케이션 배포 방법 : 현재 위치

  환경구성 : Amazon EC2 인스턴스

  태그 그룹 : 키 = Environment | 값 = Production

  **로드 밸런서 체크 박스 비활성화**

  {{< figure src="/cwave-cloudnative-textbook/images/12-codedeploy-group.png" alt="CodeDeploy 이미지" class="img-fluid" width="80%">}}



### 8. Github Action Secret 업데이트
아래 SECRET 업데이트 해주기

AWS_HOST

AWS_BUCKET
  
### 9. Github Action Workflow 작성하기
AWS CodeDploy를 사용하려면 appspec.yml 반드시 있어야 합니다.
서비스를 중지,시작 하기 위한 스크립트는 필요하면 작성합니다.

1. workflow 파일 작성

    ```yml
    ## ec2 s3 deploy12
    name: Deploy to AWS
    on:
      push:
        branches: [main]

    jobs:
      deploy:
        runs-on: ubuntu-latest
        steps:
          - name: 1.소스코드 다운로드 (simple-web)
            uses: actions/checkout@v2

          - name: 2.AWS CLI 접속정보 설정
            uses: aws-actions/configure-aws-credentials@v4
            with:
              aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
              aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
              aws-region: ap-northeast-2

          - name: 3.아티팩트 만들기
            run: |
              pwd
              zip -r deploy.zip ./*

          - name: 4.S3 아티팩트 업로드
            run: |
              aws s3 cp deploy.zip s3://${{ secrets.S3_BUCKET }}/deploy.zip

          - name: 5.현재 진행중인 AWS Deploy ID 가져오고 중단 시킨다. 
            run: |
              DEPLOYMENTS=$(aws deploy list-deployments \
                --application-name simple-web-content \
                --deployment-group-name simple-web-deploy-group \
                --include-only-statuses "InProgress" \
                --query 'deployments[]' \
                --output text)
              
              if [ ! -z "$DEPLOYMENTS" ]; then
                for deployment in $DEPLOYMENTS; do
                  echo "Stopping deployment $deployment"
                  aws deploy stop-deployment --deployment-id $deployment
                done
                # 잠시 대기하여 취소가 완료되도록 함
                sleep 10
              fi

          - name: 6. AWS Deploy를 통해 배포한다
            id: deploy
            run: |
              DEPLOYMENT_ID=$(aws deploy create-deployment \
                --application-name simple-web-content \
                --deployment-group-name simple-web-deploy-group \
                --s3-location bucket=${{ secrets.S3_BUCKET }},key=deploy.zip,bundleType=zip \
                --output text \
                --query 'deploymentId')
              #echo "::set-output name=deployment_id::$DEPLOYMENT_ID"
              #echo "{name}=deployment_id" >> $GITHUB_OUTPUT
              echo "deployment_id=${DEPLOYMENT_ID}" >> $GITHUB_OUTPUT

          - name: Wait for deployment to complete
            run: |
              aws deploy wait deployment-successful --deployment-id ${{ steps.deploy.outputs.deployment_id }}

    ```
2. 스크립트 등록
    
    scripts/before_install.sh
    ```bash
    #!/bin/bash
    if [ -d /usr/share/nginx/html ]; then
        rm -rf /usr/share/nginx/html/*
    fi
    ```
    scripts/after_install.sh
    ```bash
    #!/bin/bash
    chmod -R 755 /usr/share/nginx/html
    chown -R nginx:nginx /usr/share/nginx/html
    systemctl restart nginx
    ```
3. appspec.yml 등록하기
    ```
    version: 0.0
    os: linux
    files:
      - source: /
        destination: /usr/share/nginx/html/
        overwrite: true
    permissions:
      - object: /usr/share/nginx/html
        pattern: "**"
        owner: nginx
        group: nginx
        mode: 755
        type:
          - directory
      - object: /usr/share/nginx/html
        pattern: "**/*"
        owner: nginx
        group: nginx
        mode: 644
        type:
          - file
    hooks:
      BeforeInstall:
        - location: scripts/before_install.sh
          timeout: 300
          runas: root
      AfterInstall:
        - location: scripts/after_install.sh
          timeout: 300
          runas: root
    ```
3. 리포지토리 동기화 하기

    ```bash
    git add .
    git commit -am "add github actions workflow"
    git push origin start
    ```

4. AWS EC2 인스턴스에서 Agent 배포 로그 확인하기

    ```bash
    sudo tail -f /var/log/aws/codedeploy-agent/codedeploy-agent.log
    ```
---
## 3. Simple Web 완벽하게 배포 하기
### 1. 폴더 생성
```bash
mkdir -p xinfra/aws-ec2-single-greate
```
### 2. Gitignore 작성 
파일명 : xinfra/aws-ec2-single-greate/.gitignore 파일 작성
```gitignore
.terraform
.terraform.lock.hcl
terraform.tfstate
.DS_Store
terraform.tfstate.backup
```
### 3. 프로바이더 작성 
파일명 : xinfra/aws-ec2-single-greate/00_provider.tf 작성
```terraform
provider "aws" {
region = "ap-northeast-2" # 사용할 AWS 리전
} 
```
### 4. VPC 생성
파일명 : xinfra/aws-ec2-single-greate/10_vpc.tf 작성
```terraform
# VPC 생성
resource "aws_vpc" "dangtong-vpc" {
  cidr_block           = "10.0.0.0/16" 
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "dangtong-vpc"
  }
}
# 퍼블릭 서브넷 생성
resource "aws_subnet" "dangtong-vpc-public-subnet" {
  for_each = {
    a = { cidr = "10.0.1.0/24", az = "ap-northeast-2a" }
    b = { cidr = "10.0.2.0/24", az = "ap-northeast-2b" }
    c = { cidr = "10.0.3.0/24", az = "ap-northeast-2c" }
    d = { cidr = "10.0.4.0/24", az = "ap-northeast-2d" }
  }

  vpc_id                  = aws_vpc.dangtong-vpc.id
  cidr_block              = each.value.cidr
  availability_zone       = each.value.az
  map_public_ip_on_launch = true

  tags = {
    Name = "dangtong-vpc-public-subnet-${each.key}"
  }
}

# 인터넷 게이트웨이 생성
resource "aws_internet_gateway" "dangtong-igw" {
  vpc_id = aws_vpc.dangtong-vpc.id

  tags = {
    Name = "dangtong-igw"
  }
}

# 라우팅 테이블 생성
resource "aws_route_table" "dangtong-vpc-public-rt" {
  vpc_id = aws_vpc.dangtong-vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.dangtong-igw.id
  }

  tags = {
    Name = "dangtong-vpc-public-rt"
    
  }
}

resource "aws_route_table_association" "dangtong-vpc-public-rt" {
  for_each = {
    a = aws_subnet.dangtong-vpc-public-subnet["a"].id
    b = aws_subnet.dangtong-vpc-public-subnet["b"].id
    c = aws_subnet.dangtong-vpc-public-subnet["c"].id
    d = aws_subnet.dangtong-vpc-public-subnet["d"].id
  }
  
  subnet_id      = each.value
  route_table_id = aws_route_table.dangtong-vpc-public-rt.id
}

```
### 5. Security Group 생성
파일명 : xinfra/aws-ec2-single-greate/20_provider.tf 작성
```terraform
resource "aws_security_group" "nginx_sg" {
  name_prefix = "nginx-sg"
  vpc_id      = aws_vpc.dangtong-vpc.id 

  ingress {
    description = "Allow SSH"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "Allow HTTP"
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
}
```
### 6. EC2 인스턴스 생성
파일명 : xinfra/aws-ec2-single-greate/30_ec2.tf 작성
```terraform
# TLS 프라이빗 키 생성 (공개 키 포함)
resource "tls_private_key" "ec2_private_key" {
  algorithm = "RSA"
  rsa_bits  = 2048
}

# AWS에서 키 페어 생성
resource "aws_key_pair" "ec2_key_pair" {
  key_name   = "ec2-key_pair" # AWS에서 사용할 키 페어 이름
  public_key = tls_private_key.ec2_private_key.public_key_openssh
}

resource "aws_instance" "nginx_instance" {
  subnet_id = aws_subnet.dangtong-vpc-public-subnet["a"].id
  ami             = "ami-08b09b6acd8d62254" # Amazon Linux 2 AMI (리전별로 AMI ID가 다를 수 있음)
  instance_type   = "t2.micro"
  key_name        = aws_key_pair.ec2_key_pair.key_name # AWS에서 생성한 SSH 키 적용
  vpc_security_group_ids = [aws_security_group.nginx_sg.id]
  iam_instance_profile   = aws_iam_instance_profile.ec2_profile.name

  # EC2 시작 시 Nginx 설치 및 실행을 위한 User Data
  user_data = <<-EOF
                #!/bin/bash
                yum update -y

                # Ruby 설치
                yum install -y ruby wget

                # CodeDeploy Agent 설치
                cd /home/ec2-user
                wget https://aws-codedeploy-ap-northeast-2.s3.ap-northeast-2.amazonaws.com/latest/install
                chmod +x ./install
                ./install auto

                # CodeDeploy Agent 서비스 시작
                systemctl start codedeploy-agent
                systemctl enable codedeploy-agent

                # nginx 설치
                amazon-linux-extras install nginx1 -y
                systemctl start nginx
                systemctl enable nginx
                EOF
  tags = {
    Name        = "nginx-server"
    Environment = "Production"
  }
}

# 출력: EC2 인스턴스의 퍼블릭 IP 주소
output "nginx_instance_public_ip" {
  value       = aws_instance.nginx_instance.public_ip
  description = "Public IP of the Nginx EC2 instance"
}

# 출력: SSH 접속에 사용할 Private Key
output "ssh_private_key_pem" {
  value       = tls_private_key.ec2_private_key.private_key_pem
  description = "Private key for SSH access"
  sensitive   = true
}
```
### 7. IAM  생성
파일명 : xinfra/aws-ec2-single-greate/40_iam.tf 작성
```terraform
# GitHub Actions용 IAM 역할 생성
resource "aws_iam_role" "github_actions_role" {
  name = "GithubActionsRole"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:root"
        }
      }
    ]
  })

  tags = {
    Name = "github-actions-role"
  }
}

# CodeDeploy를 위한 EC2 IAM 역할
resource "aws_iam_role" "ec2_codedeploy_role" {
  name = "EC2CodeDeployRole"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "ec2.amazonaws.com"
        }
      }
    ]
  })

  tags = {
    Name = "ec2-codedeploy-role"
  }
}

# CodeDeploy 서비스 역할
resource "aws_iam_role" "codedeploy_service_role" {
  name = "CodeDeployServiceRole"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "codedeploy.amazonaws.com"
        }
      }
    ]
  })

  tags = {
    Name = "codedeploy-service-role"
  }
}

# GitHub Actions 역할에 정책 연결
resource "aws_iam_role_policy_attachment" "github_actions_s3" {
  role       = aws_iam_role.github_actions_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonS3FullAccess"
}

resource "aws_iam_role_policy_attachment" "github_actions_codedeploy" {
  role       = aws_iam_role.github_actions_role.name
  policy_arn = "arn:aws:iam::aws:policy/AWSCodeDeployFullAccess"
}

# EC2 인스턴스 역할에 정책 연결
resource "aws_iam_role_policy_attachment" "ec2_codedeploy_s3" {
  role       = aws_iam_role.ec2_codedeploy_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess"
}

resource "aws_iam_role_policy_attachment" "ec2_codedeploy" {
  role       = aws_iam_role.ec2_codedeploy_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSCodeDeployRole"
}

# CodeDeploy 서비스 역할에 정책 연결
resource "aws_iam_role_policy_attachment" "codedeploy_service" {
  role       = aws_iam_role.codedeploy_service_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSCodeDeployRole"
}

# EC2 인스턴스 프로파일 생성
resource "aws_iam_instance_profile" "ec2_profile" {
  name = "EC2CodeDeployProfile"
  role = aws_iam_role.ec2_codedeploy_role.name
}

# 현재 AWS 계정 ID를 가져오기 위한 데이터 소스
data "aws_caller_identity" "current" {}

# 출력: GitHub Actions 역할 ARN
output "github_actions_role_arn" {
  value       = aws_iam_role.github_actions_role.arn
  description = "ARN of the GitHub Actions IAM Role"
}
```

### 8. S3  생성
파일명 : xinfra/aws-ec2-single-greate/50_s3.tf
```terraform
# S3 버킷 생성
resource "aws_s3_bucket" "deploy_bucket" {
  bucket = "simple-web-deploy-bucket-${data.aws_caller_identity.current.account_id}"  # 고유한 버킷 이름 필요
}

# S3 버킷 버전 관리 설정
resource "aws_s3_bucket_versioning" "deploy_bucket_versioning" {
  bucket = aws_s3_bucket.deploy_bucket.id
  versioning_configuration {
    status = "Enabled"
  }
}

# S3 버킷 이름 출력
output "deploy_bucket_name" {
  value       = aws_s3_bucket.deploy_bucket.id
  description = "Name of the S3 bucket for deployments"
}
```
### 9. codedeploy 생성
파일명 : xinfra/aws-ec2-single-greate/60_codedeploy.tf
```terraform
# CodeDeploy 애플리케이션 생성
resource "aws_codedeploy_app" "web_app" {
  name = "simple-web-content"
}

# CodeDeploy 배포 그룹 생성
resource "aws_codedeploy_deployment_group" "web_deploy_group" {
  app_name               = aws_codedeploy_app.web_app.name
  deployment_group_name  = "simple-web-deploy-group"
  service_role_arn      = aws_iam_role.codedeploy_service_role.arn

  deployment_style {
    deployment_option = "WITHOUT_TRAFFIC_CONTROL"
    deployment_type   = "IN_PLACE"
  }

  ec2_tag_set {
    ec2_tag_filter {
      key   = "Environment"
      type  = "KEY_AND_VALUE"
      value = "Production"
    }
  }

  auto_rollback_configuration {
    enabled = true
    events  = ["DEPLOYMENT_FAILURE"]
  }

  alarm_configuration {
    enabled = false
  }
}

# CodeDeploy 애플리케이션 이름 출력
output "codedeploy_app_name" {
  value       = aws_codedeploy_app.web_app.name
  description = "Name of the CodeDeploy application"
}

# CodeDeploy 배포 그룹 이름 출력
output "codedeploy_deployment_group_name" {
  value       = aws_codedeploy_deployment_group.web_deploy_group.deployment_group_name
  description = "Name of the CodeDeploy deployment group"
} 
```

## 4. Simple WEB 쿠버네티스에 메뉴얼 배포하기

### 1. Docker 파일 작성
```dockerfile
FROM nginx:alpine

# 기존 nginx 기본 페이지 제거
RUN rm -rf /usr/share/nginx/html/*

# 웹 파일들을 컨테이너로 복사
COPY index.html /usr/share/nginx/html/
COPY style.css /usr/share/nginx/html/
COPY assets/ /usr/share/nginx/html/assets/
COPY scripts/ /usr/share/nginx/html/scripts/

# nginx 포트 노출
EXPOSE 80

# nginx 실행
CMD ["nginx", "-g", "daemon off;"]
```
### 2. 컨테이너 이미지 빌드
```bash
docker build -t <your-dockerhub-id>/simple-web .
```
### 3. 컨테이너 이미지 푸시
```bash
docker login --username <your-dockerhub-id>
docker push <your-dockerhub-id>/simple-web
```
### 4. 쿠버네티스 배포
1. simple-web-deploy.yaml 파일 생성
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: simple-web
  labels:
    app: simple-web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: simple-web
  template:
    metadata:
      labels:
        app: simple-web
    spec:
      containers:
        - name: simple-web
          image: <your-dockerhub-id>/simple-web
          ports:
            - containerPort: 80 
```
3. simple-web-ingress.yaml 파일 생성
```yaml
apiVersion: v1
kind: Service
metadata:
  name: simple-web-lb
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: external
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
    service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
spec:
  type: LoadBalancer
  selector:
    app: simple-web
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 800
  ports:
    - name: simple-web
      protocol: TCP
      port: 80
      targetPort: 80
```
4. 

### 5. Github workflow 작성
```yml
name: simple-web-eks-ci

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

permissions:
  contents: read
  actions: read

jobs:
  build:

    runs-on: ubuntu-latest
    env:
      DOCKER_IMAGE: ${{ secrets.DOCKER_USERNAME }}/simple-web
      DOCKER_TAG: ${{ github.run_number }}

    steps:
      - name: 1.소스코드 다운로드
        uses: actions/checkout@v4 

      - name: 7.Docker Image Build
        run: docker build  -t ${{ secrets.DOCKER_USERNAME }}/simple-web:${{ env.DOCKER_TAG }} .

      - name: 8.Docker Login
        uses: docker/login-action@v3.0.0
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_TOKEN }}
          logout: true

      - name: 9.Docker Push
        run: |
          docker push ${{ secrets.DOCKER_USERNAME }}/simple-web:${{ env.DOCKER_TAG }}

      # 서비스 리포지토리 체크아웃
      - name: 10.서비스 리포지토리 체크아웃
        uses: actions/checkout@v4
        with:
          repository: dangtong-s-inc/simple-service  
          ref: main
          path: .
          token: ${{ secrets.PAT }} 
      
      # 이미지 태그 업데이트
      - name: 11.쿠버네티스 매니페스트 파일 이미지 태그 업데이트
        run: |
          # 파일이 존재하는지 확인
          ls -la
          # 현재 파일 내용 확인
          cat simple-deploy.yaml
          sed -i "s|image: ${{ secrets.DOCKER_USERNAME }}\/simple-web.*|image: ${{ secrets.DOCKER_USERNAME }}\/simple-web:${{ env.DOCKER_TAG }}|g" simple-deploy.yaml
          # 변경된 내용 확인
          cat simple-deploy.yaml
      
      # 변경사항 커밋 및 푸시
      - name: 12.서비스 리포지토리 변경사항 커밋 및 푸시
        run: |
          git config --global user.name 'github-actions[bot]'
          git config --global user.email 'github-actions[bot]@users.noreply.github.com'
          git commit -am "Update image tag to ${{ env.DOCKER_TAG }}"
          git remote set-url origin https://${{ secrets.PAT }}@github.com/dangtong-s-inc/simple-service.git
          
          git push origin main
```

## 5. Simple WEB 쿠버네티스에 자동배포 하기



