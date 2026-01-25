# EC2에서 GitHub Actions 배포하기

## Summary
GitHub Actions를 활용하여 EC2 인스턴스에 애플리케이션을 자동 배포하는 방법을 알아봅니다. CI/CD 파이프라인을 구축하여 코드 변경 시 수동 배포 없이 효율적으로 운영할 수 있도록 돕는 실용적인 가이드입니다.

## Body

### GitHub Actions와 EC2 자동 배포의 이점
애플리케이션 개발에서 배포는 반복적이고 번거로운 작업일 수 있습니다. GitHub Actions는 코드 변경이 푸시될 때마다 자동으로 빌드, 테스트, 배포를 수행하는 CI/CD (지속적 통합/지속적 배포) 워크플로우를 제공합니다. EC2에 배포를 자동화하면 개발자는 코드 작성에만 집중하고, 안정적이고 빠른 서비스 업데이트가 가능해집니다.

### 사전 준비 사항
배포 워크플로우를 구성하기 전에 몇 가지 준비가 필요합니다.

1.  **EC2 인스턴스**: 배포할 애플리케이션이 실행될 EC2 인스턴스가 준비되어 있어야 합니다. Public IP 주소 또는 Public DNS를 통해 접근 가능해야 합니다.
2.  **SSH 키 페어**: GitHub Actions가 EC2에 접속하기 위한 SSH 키 페어가 필요합니다.
    *   로컬에서 `ssh-keygen -t rsa -b 4096 -C "your_email@example.com"` 명령어로 키를 생성합니다.
    *   생성된 퍼블릭 키 (`id_rsa.pub`) 내용을 EC2 인스턴스의 `~/.ssh/authorized_keys` 파일에 추가합니다.
    *   프라이빗 키 (`id_rsa`)는 **매우 중요**하니 안전하게 보관하세요.
3.  **GitHub Repository**: 배포할 코드가 저장된 GitHub Repository가 필요합니다.

### GitHub Secrets 설정
GitHub Actions 워크플로우에서 직접 민감한 정보를 노출하는 것은 위험합니다. GitHub Secrets를 활용하여 안전하게 정보를 관리합니다.

Repository 설정에서 `Secrets and variables` -> `Actions` -> `New repository secret`을 클릭하여 다음 Secret들을 추가합니다.

*   `EC2_SSH_PRIVATE_KEY`: 위에서 생성한 프라이빗 키(`id_rsa`)의 전체 내용을 복사하여 붙여넣습니다. (-----BEGIN OPENSSH PRIVATE KEY----- 부터 -----END OPENSSH PRIVATE KEY----- 까지)
*   `EC2_HOST`: EC2 인스턴스의 Public IP 주소 또는 Public DNS를 입력합니다.
*   `EC2_USER`: EC2 인스턴스에 접속할 사용자 이름 (예: `ubuntu`, `ec2-user`)

### GitHub Actions Workflow 작성
`.github/workflows` 디렉토리 아래에 `deploy.yml` 파일을 생성합니다. 이 파일에 배포 워크플로우를 정의합니다.

다음은 Node.js 애플리케이션을 EC2에 배포하는 예시입니다. 다른 언어나 프레임워크를 사용한다면 빌드 스텝만 적절히 변경하면 됩니다.

```yaml
name: Deploy to EC2

on:
  push:
    branches:
      - main # main 브랜치에 푸시될 때 워크플로우 실행

env:
  NODE_VERSION: '18' # 사용할 Node.js 버전

jobs:
  deploy:
    runs-on: ubuntu-latest # 워크플로우를 실행할 가상 환경

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4 # GitHub 리포지토리 체크아웃

      - name: Setup Node.js environment
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm' # npm 캐시 활성화

      - name: Install dependencies
        run: npm install

      - name: Build project
        run: npm run build # 프로젝트 빌드 명령 (package.json에 정의)

      - name: Deploy to EC2
        uses: appleboy/ssh-action@v1.0.0 # SSH를 통해 EC2에 배포하는 액션 사용
        with:
          host: ${{ secrets.EC2_HOST }} # EC2 호스트 (GitHub Secret에서 가져옴)
          username: ${{ secrets.EC2_USER }} # EC2 사용자 (GitHub Secret에서 가져옴)
          key: ${{ secrets.EC2_SSH_PRIVATE_KEY }} # SSH 프라이빗 키 (GitHub Secret에서 가져옴)
          script: |
            # EC2 서버에서 실행될 명령들
            cd /home/${{ secrets.EC2_USER }}/your-app-directory # 배포할 애플리케이션 디렉토리로 이동
            rm -rf ./* # 기존 파일 삭제 (필요에 따라 주석 처리 또는 다른 전략 사용)
            # 여기에서 Git clone 대신, 위에서 빌드된 결과물을 S3에 업로드하고 EC2에서 다운로드하거나,
            # scp를 통해 직접 전송하는 등의 방법을 사용할 수 있습니다.
            # 이 예시에서는 단순화를 위해 'git pull'을 사용하지만, 실제로는 빌드된 아티팩트를 전송하는 것이 일반적입니다.
            git pull origin main # 최신 코드 pull
            npm install --production
            npm run build # EC2에서 다시 빌드하는 경우 (로컬 빌드 결과를 전송했다면 필요 없음)
            pm2 restart your-app # PM2로 애플리케이션 재시작 (PM2가 설치되어 있다고 가정)
            # 또는 docker-compose up -d --build 등 컨테이너 배포 명령

```

### 배포 스크립트 상세 설명

`appleboy/ssh-action`의 `script` 부분에는 EC2 인스턴스에서 실행될 쉘 스크립트를 작성합니다.
위 예시에서는 EC2 서버에서 직접 `git pull`을 통해 코드를 받아와 빌드하고 재시작하는 방식입니다.
더 견고한 방식으로는 GitHub Actions에서 빌드된 결과물(아티팩트)을 S3에 업로드하고, EC2에서 S3로부터 해당 아티팩트를 다운로드하여 배포하는 방식이 있습니다. 이 경우 EC2에 AWS CLI가 설치되어 있어야 합니다.

*   `cd /home/${{ secrets.EC2_USER }}/your-app-directory`: 애플리케이션이 위치할 디렉토리로 이동합니다. 미리 디렉토리를 생성해두어야 합니다.
*   `rm -rf ./*`: 기존 배포된 파일을 모두 삭제합니다. 이 명령은 주의해서 사용해야 하며, 중요한 데이터가 포함된 디렉토리에는 사용하지 않도록 합니다.
*   `git pull origin main`: 최신 코드를 받습니다. (이 예시에서는 리포지토리를 clone한 후 이 명령을 사용한다고 가정)
*   `npm install --production`: 필요한 의존성을 설치합니다.
*   `npm run build`: 애플리케이션을 빌드합니다.
*   `pm2 restart your-app`: `pm2`와 같은 프로세스 매니저를 사용하여 애플리케이션을 재시작합니다. `pm2`가 설치되어 있지 않다면 `npm start` 등으로 직접 실행할 수 있습니다.

### 마무리
이제 `main` 브랜치에 코드를 푸시할 때마다 GitHub Actions 워크플로우가 자동으로 실행되어 EC2 인스턴스에 애플리케이션이 배포됩니다. 이를 통해 수동 배포의 번거로움을 줄이고 개발 생산성을 크게 향상시킬 수 있습니다. 더 복잡한 환경에서는 AWS CodeDeploy나 Docker 컨테이너 배포 등을 고려할 수 있습니다.

## Tags
- GitHubActions
- EC2
- CI/CD
- 배포
- 자동화