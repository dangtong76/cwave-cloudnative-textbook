---
title: "👉 Appendix - 명령어 모음"
weight: 999
date: 2025-02-02
draft: false
#url: "/git-reference/"
---

1. 깃 명령어 모음

    ```bash
    # 리포지토리 초기화
    git init 

    # Git Config 파일 수정 
    git config --global --edit

    git config --global user.name "Your Name"
    git config --global user.email you@example.com

    # Stage 에 파일 추가
    git add .  

    # 트래킹에서 제거 (파일)
    git rm --cached <file-name>

    # 트래킹에서 제거 (폴더)
    git rm -r --cached <folder-name>
    
    # 커밋할때 파일추가하고 메시지까지 남기기
    git commit -am <message> 

    # 로컬 커밋들을 원격 저장소로 업로드 및 update, "원격 추적 브렌치 update" 
    git push origin main 

    # 로컬 커밋들을 원격 저장소로 강제 업로드 및 update, "원격 추적 브렌치 update" 
    git push origin main 

    # 원격 저장소 main 브렌치 내용을 "원격 추적 브렌치 fetch", "현재 브렌치 merge", "작업 디렉토리 update" 
    git pull origin main 

    # 원격 저장소 main 브렌치 내용을 "원격 추적 브렌치 fetch" 
    git fetch origin main 

    # 원격의 저장소 내용을 로컬로 업데이트 하고 그위에 로커 커밋을 추가함
    git pull --rebase 

    # 해당 커밋으로 되돌아감 - 이후 커밋 히스토리 보관
    git revert <commit hash ID> 

    # 해당 커밋바로 이전(^)으로 돌아감 - 이후 커밋 히스토리는 삭제됨
    git reset --hard <commit hash ID>^

    # 커밋 히스토리 보기
    git log 

    # 스테이징 영역 전체 삭제
    git reset HEAD .

    # 스테이징 영역 특정 파일 삭제
    git reset HEAD <file>

    # 브렌치 목록 보기 (로컬)
    git branch 

    # 브렌치 목록 보기 (원격격)
    git branch -r

    # 브렌치 삭제
    git branch -d <branch name>

    # 병합되지 않는 브렌치 삭제
    git branch -D <branch name>
    git branch -D origin/tutor --remote
    
    # 원격 브랜치 삭제
    git push origin --delete <branch name>

    git fetch -p # 삭제후 로컬 동기화
    ```


2. gh 명령어 모음

    ```bash
    # github cli 인증
    gh auth login

    # 리포지토리 인증 및 연결 상태 황인
    gh auth status

    # 리포지토리 연결 변경 (다른 계정으로)
    gh auth switch 

    # 워크플로우 조회
    gh workflow list

    # 워크플로우 활성화
    gh workflow enable "Change Working Dir & Shell"

    # 워크플로우 비활성화
    gh workflow disable  "Change Working Dir & Shell"

    # 워크플로우 run 리스트
    gh run list

    # 실패한 run 리스트만 보기
    gh run list --status failure

    # 워크플로우 run Cancel
    gh run cancel <workflow-hash-id>

    # 워크플로우 run Delete
    gh run delete <workflow-hash-id>

    # 리포지토리 생성
    gh repo create <REPO_NAME> --public

    # 리포지토리 Fork
    gh repo fork <github https url>
    ```
3. Terraform 명령어 모음

    ```bash
    # 초기화    
    terraform init
    terraform init --upgrade

    # 계획 확인
    terraform plan

    # 적용
    terraform apply

    # 인스턴스 리스트
    terraform state list

    # 인스턴스 세부사항 조회
    terraform state show aws_instance.nginx_instance
    ```

4. AWS CLI 명령어 모음

    ```bash
    # 인증정보 설정
    aws configure

    # 인증정보 확인
    aws sts get-caller-identity

    # 인증정보 삭제
    aws configure delete profile --profile <profile-name> 

    # EC2 키확인 및 삭제
    aws ec2 describe-key-pairs --query 'KeyPairs[*].KeyName'
    aws ec2 delete-key-pair --key-name ec2-key

    # deployment 리스트
    aws deploy list-deployments

    # aws eks 로 부터 kubectl config 업데이트 하기
    aws eks update-kubeconfig --region ap-northeast-2 --name istory

    # aws 로드밸런서 상태 확인
    aws elbv2 describe-load-balancers

    # default SC 설정
    kubectl patch storageclass gp2 -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

    # Deploy Agent 로그 확인
    tail -100 /var/log/aws/codedeploy-agent/codedeploy-agent.log

    # Deploy Agent 상태 확인
    sudo systemctl status codedeploy-agent
    
    # Deploy Agent 재가동
    sudo systemctl restart codedeploy-agent

    ``` 



5. Hugo 명령어

    ```bash
    # 컨테이너 내부에서 사용
    hugo server --bind 0.0.0.0 --baseURL=http://localhost --port 1313 -D

    # 로컬머신에서 사용
    hugo server -D 
    ```
5. Java 관련
    ```bash
    # 애플리케이션 실행
    ./gradlew bootRun

6. Docker 명령어
    ```bash
    # manifest 확인
    docker manifest inspect dangtong76/cicd-devops-ide:latest

    # buildx 이용한 멀티 플랫폼 빌드
    docker buildx build  --platform linux/amd64,linux/arm64  -t <your-dockerhub-id>/<image-name> --push .
    
    ```