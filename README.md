# Codyssey Week 1 — 개발 환경 구축 실습

## 0) 프로젝트 개요
터미널, Docker, Git을 직접 다뤄 보며 개발 환경을 구축하고, 그 과정과 결과를 이 문서 하나로 확인할 수 있게 정리한다.

**미션 목표**
- 터미널 기본 조작과 파일 권한 변경을 익힌다.
- Docker를 설치·점검하고 컨테이너를 실행·관찰한다(hello-world, ubuntu, exec/attach).
- Dockerfile로 커스텀 이미지를 빌드하고 포트 매핑으로 접속한다.
- 바인드 마운트와 볼륨으로 데이터 반영·영속성을 확인한다.
- Git을 설정하고 GitHub 저장소와 연동해 결과를 제출한다.

**저장소 구성**
```text
week1/
├── README.md        # 기술 문서 (수행 결과 전체)
├── Dockerfile       # 커스텀 nginx 이미지 정의
└── site/
    └── index.html   # 이미지에 복사되는 정적 웹 페이지
```

## 1) 실행 환경
- OS: Windows 11 Pro
- Shell: PowerShell 5.1 / Git Bash
- Docker: 29.x (Docker Desktop)
- Git: 2.56.x

## 2) 수행 체크리스트
- [ ] 터미널 기본 조작 및 폴더 구성
- [ ] 권한 변경 실습
- [ ] Docker 설치/점검
- [ ] hello-world 실행
- [ ] Dockerfile 빌드/실행
- [ ] 포트 매핑 접속(2회)
- [ ] 바인드 마운트 반영
- [ ] 볼륨 영속성
- [ ] Git 설정 + VSCode GitHub 연동

## 3) Dockerfile 커스텀 예시
```dockerfile
FROM nginx:alpine
LABEL org.opencontainers.image.title="my-custom-nginx"
ENV APP_ENV=dev
COPY site/ /usr/share/nginx/html/
```

- `FROM nginx:alpine`: 경량 Alpine 기반 nginx 이미지를 베이스로 사용
- `LABEL`: 이미지 메타데이터(제목) 지정
- `ENV APP_ENV=dev`: 컨테이너 환경 변수 설정
- `COPY site/ ...`: 로컬 `site/` 폴더의 정적 파일을 nginx 웹 루트로 복사

빌드 및 실행:
```powershell
docker build -t my-custom-nginx .
docker run -d --name my-nginx -p 8080:80 my-custom-nginx
```
브라우저에서 http://localhost:8080 접속

## 4) 컨테이너 실행 실습

### 4-1. hello-world 실행
```bash
$ docker run --rm hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
...
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.
```
→ 로컬에 이미지가 없어 Docker Hub에서 자동으로 pull한 뒤 실행 성공. 메시지 출력 후 컨테이너는 바로 종료됨.

### 4-2. ubuntu 컨테이너 실행 및 내부 명령 수행
```bash
$ docker run -dit --name my-ubuntu ubuntu bash
6d5674f2f4fb...

$ docker exec my-ubuntu ls /
bin  boot  dev  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var

$ docker exec my-ubuntu echo "Hello from ubuntu container"
Hello from ubuntu container

$ docker exec my-ubuntu cat /etc/os-release
PRETTY_NAME="Ubuntu 26.04.1 LTS"
NAME="Ubuntu"
```
→ 직접 들어가서 작업하려면 `docker exec -it my-ubuntu bash`로 내부 셸에 진입.

### 4-3. 컨테이너 종료/유지 관찰 (exec vs attach)
```bash
# exec로 새 bash를 띄운 뒤 exit
$ docker exec -i my-ubuntu bash   # 입력: echo inside-exec; exit
inside-exec
$ docker ps -a --filter name=my-ubuntu
my-ubuntu   Up 2 minutes            ← 컨테이너 계속 실행 중

# attach로 메인 프로세스(bash)에 붙은 뒤 exit
$ docker run -di --name my-ubuntu-attach ubuntu bash
$ docker attach my-ubuntu-attach  # 입력: echo inside-attach; exit
$ docker ps -a --filter name=my-ubuntu-attach
my-ubuntu-attach   Exited (0) 2 seconds ago   ← 컨테이너 종료됨
```

| 구분 | `docker exec` | `docker attach` |
|---|---|---|
| 연결 대상 | 컨테이너 안에 **새 프로세스**를 추가로 실행 | 컨테이너의 **메인 프로세스(PID 1)** 입출력에 연결 |
| `exit` 시 | 추가로 띄운 프로세스만 종료 → 컨테이너 **유지** | 메인 프로세스가 종료 → 컨테이너 **종료** |
| 종료 없이 빠져나오기 | `exit` 해도 무방 | `Ctrl+P` → `Ctrl+Q` (detach, `-it`로 실행한 경우) |
| 주 용도 | 실행 중인 컨테이너 점검·디버깅 | 메인 프로세스 출력 확인, 대화형 프로세스 조작 |

**정리:** 컨테이너는 메인 프로세스가 살아 있는 동안만 실행된다. 작업용으로는 `exec`가 안전하고, `attach` 상태에서 `exit`하면 컨테이너가 멈춘다.

## 5) 검증 방법 및 증거 위치

| 항목 | 검증 명령 | 확인한 내용 | 결과/증거 위치 |
|---|---|---|---|
| 실행 환경 | `git --version`, `docker --version` | Git 2.56.1, Docker 29.9.3 설치 확인 | [1) 실행 환경](#1-실행-환경) |
| Docker 동작 | `docker run --rm hello-world` | `Hello from Docker!` 메시지 출력 | [4-1. hello-world 실행](#4-1-hello-world-실행) |
| ubuntu 컨테이너 | `docker exec my-ubuntu ls /`, `echo` | 루트 디렉터리 목록, echo 문자열 출력, Ubuntu 26.04.1 LTS | [4-2. ubuntu 컨테이너](#4-2-ubuntu-컨테이너-실행-및-내부-명령-수행) |
| exec vs attach | `docker ps -a --filter name=...` | exec 후 exit → `Up`, attach 후 exit → `Exited (0)` | [4-3. exec vs attach](#4-3-컨테이너-종료유지-관찰-exec-vs-attach) |
| Dockerfile 빌드 | `docker build -t my-custom-nginx .` | `naming to docker.io/library/my-custom-nginx:latest done` | [Dockerfile](Dockerfile), [3) Dockerfile 예시](#3-dockerfile-커스텀-예시) |
| 포트 매핑 접속 | `docker run -d -p 8080:80 my-custom-nginx` 후 http://localhost:8080 요청 | `Hello from my-custom-nginx!` 페이지 응답 | [site/index.html](site/index.html) |
| 환경 변수 | `docker exec my-nginx printenv APP_ENV` | `dev` 출력 | [3) Dockerfile 예시](#3-dockerfile-커스텀-예시) |

검증 로그:
```bash
$ docker build -t my-custom-nginx .
#7 [2/2] COPY site/ /usr/share/nginx/html/
#7 DONE 0.1s
#8 naming to docker.io/library/my-custom-nginx:latest done

$ docker run -d --name my-nginx -p 8080:80 my-custom-nginx
aada730b830b...

$ curl http://localhost:8080
<h1>Hello from my-custom-nginx!</h1>

$ docker exec my-nginx printenv APP_ENV
dev
```

> 미수행 항목(권한 변경, 포트 매핑 2회차, 바인드 마운트, 볼륨, Git/GitHub 연동)은 실습 후 위 표에 행을 추가한다.
