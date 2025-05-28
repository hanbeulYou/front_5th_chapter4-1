# 9주차 과제 - 인프라 관점의 성능 최적화

## 기본 과제

### 과제 목표

#### 기본과제

> S3와 CloudFront를 활용한 정적 웹사이트 배포 파이프라인을 구축하고, GitHub Actions를 통해 자동화된 CI/CD를 경험

#### 심화과제

> CDN 도입 전후의 성능 차이를 분석하여 CloudFront의 캐싱 효과와 정적 자산 최적화의 중요성을 이해

### 주요 링크

- S3 정적 웹사이트 엔드포인트: http://hanghaeplus-aws-test.s3-website.ap-northeast-2.amazonaws.com
- CloudFront 배포 도메인: https://d1uc75g1cu3aph.cloudfront.net

### 프론트엔드 배포 파이프라인

![프론트엔드 배포 파이프라인](./docs/frontend-pipeline.png)

### 배포 프로세스

GitHub Actions를 이용해 `main` 브랜치에 push되거나 수동 실행(`workflow_dispatch`) 시 자동으로 배포가 진행됨.

#### 1. 코드 체크아웃

저장소의 코드를 runner 환경에 다운로드함.

```yaml
- name: Checkout repository
  uses: actions/checkout@v4
```

#### 2. 의존성 설치

package-lock.json을 기준으로 프로젝트 의존성을 설치함.

```yaml
- name: Install dependencies
  run: npm ci
```

#### 3. 프로젝트 빌드

Next.js 프로젝트를 빌드함.
next export 기반의 설정으로 out/ 디렉토리에 정적 파일을 생성함.

```yaml
- name: Build
  run: npm run build
```

#### 4. AWS 자격 증명 구성

GitHub Secrets에 등록된 값을 통해 AWS CLI 자격 증명을 구성함.

```yaml
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v1
  with:
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws-region: ${{ secrets.AWS_REGION }}
```

#### 5. S3 버킷 업로드

빌드된 정적 파일을 S3 버킷에 업로드함.

`--delete` 옵션으로 불필요한 파일을 정리함.

```yaml
- name: Upload to S3
  run: aws s3 cp out/ s3://hanghaeplus-aws-test --recursive --delete
```

#### 6. CloudFront 캐시 무효화

모든 경로(/\*)에 대해 CloudFront 캐시를 무효화함.

```yaml
- name: Invalidate CloudFront cache
  run: |
    aws cloudfront create-invalidation --distribution-id ${{ secrets.CLOUDFRONT_DISTRIBUTION_ID }} --paths "/*"
```

### 주요 개념

- GitHub Actions과 CI/CD 도구:  
  GitHub Actions는 코드 변경을 트리거로 자동화된 작업을 실행할 수 있는 CI/CD 도구로, 이번 과제에서는 코드 푸시 시 자동으로 빌드하고 배포하는 파이프라인 구성에 사용함.

- S3와 스토리지:  
  S3(Simple Storage Service)는 AWS에서 제공하는 객체 스토리지 서비스로, 이번 과제에서는 정적 파일을 업로드하고 웹사이트로 제공하기 위한 정적 웹 호스팅 기능을 활용함.

- CloudFront와 CDN:  
  CloudFront는 AWS의 콘텐츠 전송 네트워크(CDN) 서비스로, 전 세계 엣지 서버를 통해 S3의 정적 파일을 빠르게 전달하고 사용자 경험을 개선함.

- 캐시 무효화(Cache Invalidation):  
  CloudFront는 성능을 위해 파일을 캐시하지만, 새로 빌드된 파일을 사용자에게 즉시 보여주기 위해서는 기존 캐시를 무효화하는 작업이 필요함.

- Repository secret과 환경변수:  
  GitHub Actions에서 사용하는 민감한 정보(AWS 자격 증명, 버킷 이름 등)는 저장소의 Secret으로 안전하게 보관하며, 워크플로우 실행 시 환경변수로 주입하여 보안을 유지함.

### 학습 내용 정리

- 정적 사이트 배포를 위한 Next.js 빌드 구조

  `next export` 명령어를 통해 Next.js 프로젝트를 정적으로 변환하고, S3에 업로드 가능한 형태로 빌드 결과물을 구성함.

  `next build` 이후 `next export`를 실행하면 `out/` 디렉토리에 HTML, CSS, JS 파일이 생성되며, 이는 서버 없이도 정적 웹사이트로 배포 가능한 구조를 가짐.

  export 방식은 `getServerSideProps`, `API Routes`와 같이 서버 기능이 포함된 구성은 사용할 수 없고, `getStaticProps` 기반의 페이지만 지원함.

  이러한 제약을 이해하고 라우팅 및 데이터 처리 방식을 사전에 설계해야 함.

- AWS 리소스 간 연동 구조 구성  
  S3 버킷을 오리진으로 사용하는 CloudFront 배포를 생성하고, 두 리소스 간 연결을 통해 퍼블릭 정적 웹사이트를 글로벌 CDN으로 확장하는 구조를 직접 구성함.

  단순 파일 업로드를 넘어서, 웹 인프라 간 의존성과 흐름을 실습함.

- 워크플로우 단위의 CI/CD 구성  
  GitHub Actions를 활용해 배포 자동화를 구현함.

  `checkout`, `install`, `build`, `upload`, `invalidate` 각 단계가 명확히 분리된 워크플로우를 작성하고, 이를 통해 CI/CD 구성 시 단계별 역할 분리와 실패 지점을 명확히 관리하는 법을 학습함.

- CloudFront 캐시 전략 및 무효화 타이밍
  정적 자산 배포 시 캐시 정책에 따라 변경 사항이 반영되지 않을 수 있다는 점을 직접 경험하고, 이를 해결하기 위한 캐시 무효화 전략(`create-invalidation`)의 필요성과 적용 타이밍에 고민함.

- 배포 환경의 보안 구성과 유지 관리 전략
  민감 정보(AWS 자격 증명, 버킷 이름 등)를 GitHub Secrets로 안전하게 관리하고, 외부 노출 없이 워크플로우에서 활용하는 방식을 실습함.

## 심화 과제

### 성능 개선 보고서

### 학습 내용 정리
