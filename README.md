## 개요

root 계정이 아닌 일반 사용자 계정에 `sudo` 권한을 부여한 뒤 설치를 진행합니다.

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
