## 개요

root 계정이 아닌 일반 사용자 계정에 `sudo` 권한을 부여한 뒤 설치를 진행합니다.

## 단계별 문서 (샘플환경)

1. [서버 기본 설정 및 Docker 설치](./샘플환경/1.서버%20기본%20설정%20및%20Docker%20설치.md)
2. [Kubernetes (Minikube) 설치](./샘플환경/2.Kubernetes%20%28Minikube%29%20설치.md)
3. [Docker Registry 설치 (로컬)](./샘플환경/3.Docker%20Registry%20설치%20%28로컬%29.md)
4. [Jenkins 설치 및 설정 (OpenJDK 17)](./샘플환경/4.Jenkins%20설치%20및%20설정%20%28OpenJDK%2017%29.md)
5. [SVN 서버 설치 및 Jenkins 연동](./샘플환경/5.SVN%20서버%20설치%20및%20Jenkins%20연동.md)
6. [config 서비스 배포 파이프라인 구성 (Gradle)](./샘플환경/6.config%20서비스%20배포%20파이프라인%20구성%20%28Gradle%29.md)
7. [discovery 서비스 배포 파이프라인 구성 (Eureka Server)](./샘플환경/7.discovery%20서비스%20배포%20파이프라인%20구성%20%28Eureka%20Server%29.md)
8. [apigateway 서비스 배포 파이프라인 구성](./샘플환경/8.apigateway%20서비스%20배포%20파이프라인%20구성.md)
9. [sample-service 배포 가이드](./샘플환경/9.sample-service%20배포%20가이드.md)

## 기타 문서

- [kubeadm 단일 노드 설치 (Rocky Linux + containerd)](./kubeadm/kubeadm-rocky-single-node-containerd.md)

## 1) wheel 그룹에 사용자 추가 (권장)

```bash
usermod -aG wheel $USER
```

## 2) (선택) 비밀번호 없이 sudo 허용

보안상 주의해서 사용하세요.

### 2-1. visudo 실행

```bash
visudo
```

### 2-2. 아래 한 줄 추가

```bash
jnet ALL=(ALL) NOPASSWD:ALL
```

### 2-3. 또는 wheel 그룹 전체에 적용

```bash
%wheel ALL=(ALL) NOPASSWD:ALL
```
