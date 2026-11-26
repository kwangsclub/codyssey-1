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
├── README.md          # 기술 문서 (모든 수행 결과와 증거)
├── Dockerfile         # 커스텀 nginx 이미지 정의
├── site/
│   └── index.html     # Dockerfile의 COPY로 이미지에 구워지는 정적 페이지
├── bind-demo/
│   └── index.html     # 바인드 마운트 실습용 페이지 (호스트에서 수정 → 컨테이너에 즉시 반영)
└── .gitignore         # 볼륨 백업 파일(backup/)은 커밋하지 않음
```
구성 기준:
- **이미지에 포함할 파일(`site/`)과 실행 중 바꿀 파일(`bind-demo/`)을 분리**했다. `site/`는 빌드 시점에 고정되고, `bind-demo/`는 컨테이너를 다시 만들지 않고 고칠 수 있다.
- 모든 경로는 저장소 루트 기준 **상대 경로**로 참조한다. 다른 PC에서 clone해도 같은 명령으로 재현된다.
- 백업 결과물(`backup/*.tar.gz`)은 바이너리이고 실행할 때마다 새로 만들어지므로 `.gitignore`로 제외했다.

## 1) 실행 환경
- OS: Windows 11 Pro
- Shell/터미널: PowerShell 5.1(Docker 명령), MSYS2 UCRT64 bash(파일 조작 실습)
- Docker: Docker Desktop 29.9.3 (Client/Server 동일)
- Git: 2.56.1.windows.1

```powershell
PS> docker --version
Docker version 29.9.3, build 5457199

PS> git --version
git version 2.56.1.windows.1

PS> docker version --format "Client: {{.Client.Version}} / Server: {{.Server.Version}}"
Client: 29.9.3 / Server: 29.9.3

PS> docker info --format "Server Version: {{.ServerVersion}} / OS: {{.OperatingSystem}} / Containers: {{.Containers}} (running {{.ContainersRunning}}) / Images: {{.Images}}"
Server Version: 29.9.3 / OS: Docker Desktop / Containers: 2 (running 2) / Images: 3
```
→ 확인: `docker info`가 Server(데몬) 정보를 반환하므로 Docker 데몬이 실행 중이다. 데몬이 꺼져 있을 때의 오류와 조치는 [7-1](#7-1-docker-데몬-미실행)에 기록했다.

## 2) 수행 체크리스트
- [x] 터미널 기본 조작 및 폴더 구성 → [3-A](#3-a-터미널-기본-조작-생성이동삭제)
- [x] 권한 변경 실습 → [3-B](#3-b-권한-변경-실습)
- [x] Docker 설치/점검 → [1) 실행 환경](#1-실행-환경), [4-4](#4-4-이미지컨테이너-목록)
- [x] hello-world 실행 → [4-1](#4-1-hello-world-실행)
- [x] Dockerfile 빌드/실행 → [3)](#3-dockerfile-커스텀-예시), [5) 검증 로그](#5-검증-방법-및-증거-위치)
- [x] 포트 매핑 접속(2회) → [4-5](#4-5-포트-매핑-2회차-같은-이미지-다른-호스트-포트)
- [x] 바인드 마운트 반영 → [4-7](#4-7-바인드-마운트-호스트-수정--컨테이너-즉시-반영)
- [x] 볼륨 영속성 → [4-6](#4-6-볼륨-영속성-컨테이너-삭제-후에도-데이터-유지)
- [x] Git 설정 + GitHub 연동 → [4-8](#4-8-git-설정-및-github-연동)

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

## 3-A) 터미널 기본 조작 (생성/이동/삭제)
환경: MSYS2 UCRT64 bash (Windows 11)

```bash
$ cd ~/codyssey/week1
$ pwd
/c/Users/이희광/codyssey/week1

$ mkdir -p practice/docs practice/backup
$ ls -l practice
total 0
drwxr-xr-x 1 이희광 197121 0 Nov 21 15:10 backup
drwxr-xr-x 1 이희광 197121 0 Nov 21 15:10 docs

$ touch practice/docs/memo.txt
$ echo "hello codyssey" > practice/docs/note.txt
$ ls -l practice/docs
total 1
-rw-r--r-- 1 이희광 197121  0 Nov 21 15:09 memo.txt
-rw-r--r-- 1 이희광 197121 15 Nov 21 15:09 note.txt
$ cat practice/docs/note.txt
hello codyssey

$ cp practice/docs/note.txt practice/backup/
$ mv practice/docs/memo.txt practice/backup/memo_moved.txt
$ ls -l practice/docs practice/backup
practice/backup:
total 1
-rw-r--r-- 1 이희광 197121  0 Nov 21 15:09 memo_moved.txt
-rw-r--r-- 1 이희광 197121 15 Nov 21 15:10 note.txt

practice/docs:
total 1
-rw-r--r-- 1 이희광 197121 15 Nov 21 15:09 note.txt

$ rm practice/backup/memo_moved.txt
$ ls -l practice/backup
total 1
-rw-r--r-- 1 이희광 197121 15 Nov 21 15:10 note.txt

$ rm -r practice
$ ls -l
total 9
-rw-r--r-- 1 이희광 197121  123 Nov 21 13:55 Dockerfile
-rw-r--r-- 1 이희광 197121 6303 Nov 21 14:24 README.md
drwxr-xr-x 1 이희광 197121    0 Nov 21 13:55 site
```
→ 확인: `cp` 후 backup에 note.txt 복사, `mv` 후 memo.txt가 docs에서 사라지고 backup에 memo_moved.txt로 이동, `rm` 후 memo_moved.txt가 사라지고 `rm -r` 후 practice 폴더가 목록에서 사라짐

## 3-B) 권한 변경 실습
환경: ubuntu 컨테이너 (`docker exec -it my-ubuntu bash`)
Windows 파일 시스템(NTFS)은 리눅스 권한을 저장하지 않아 실제 리눅스 환경인 컨테이너에서 실습함.

**파일 권한**
```bash
# cd /tmp
# echo 'echo "script run OK"' > hello.sh
# ls -l hello.sh
-rw-r--r-- 1 root root 21 Nov 21 06:17 hello.sh
# ./hello.sh
bash: ./hello.sh: Permission denied
# chmod +x hello.sh
# ls -l hello.sh
-rwxr-xr-x 1 root root 21 Nov 21 06:17 hello.sh
# ./hello.sh
script run OK
# chmod 600 hello.sh
# ls -l hello.sh
-rw------- 1 root root 21 Nov 21 06:17 hello.sh
```

**폴더 권한**
```bash
# mkdir secret
# ls -ld secret
drwxr-xr-x 2 root root 4096 Nov 21 06:21 secret
# chmod 700 secret
# ls -ld secret
drwx------ 2 root root 4096 Nov 21 06:21 secret
```
→ 확인: 실행 권한(x)이 없을 때는 `Permission denied`, `chmod +x` 후 `-rwxr-xr-x`로 바뀌며 실행 성공. 폴더는 755(`drwxr-xr-x`) → 700(`drwx------`)으로 변경됨.

**권한 표기**: r=4, w=2, x=1 / 순서는 소유자·그룹·기타 사용자
- `755` = `rwxr-xr-x`, `644` = `rw-r--r--`, `600` = `rw-------`, `700` = `rwx------`

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

### 4-4. 이미지·컨테이너 목록
```powershell
PS> docker images
IMAGE                    ID             DISK USAGE   CONTENT SIZE   EXTRA
hello-world:latest       1e4c05b897a7       25.9kB         9.49kB
my-custom-nginx:latest   5a4729d43130       93.6MB         26.4MB   U
ubuntu:latest            d66b366dac61        163MB         45.9MB   U

PS> docker ps -a
CONTAINER ID   IMAGE             COMMAND                  CREATED      STATUS                       PORTS                  NAMES
6d5674f2f4fb   ubuntu            "bash"                   4 days ago   Exited (255) 7 seconds ago                          my-ubuntu
aada730b830b   my-custom-nginx   "/docker-entrypoint.…"   4 days ago   Exited (255) 7 seconds ago   0.0.0.0:8080->80/tcp   my-nginx

PS> docker start my-nginx my-ubuntu
my-nginx
my-ubuntu
```
→ 확인: 이미지 3개(hello-world, my-custom-nginx, ubuntu)가 로컬에 있다. `hello-world`는 `--rm`으로 실행해 컨테이너가 남지 않았다. `my-ubuntu`, `my-nginx`는 PC 재시작으로 Docker가 꺼지면서 `Exited (255)`가 되었고, `docker start`로 다시 실행했다. `docker ps -a`는 멈춘 컨테이너까지 보여 주고, `docker ps`는 실행 중인 것만 보여 준다.

### 4-5. 포트 매핑 2회차 (같은 이미지, 다른 호스트 포트)
```powershell
PS> docker run -d --name my-nginx-2 -p 8081:80 my-custom-nginx
46908b9837a2142775a0e5ce068770d89feb907211a4420549960c351ea3a5e6

PS> docker ps
CONTAINER ID   IMAGE             COMMAND                  CREATED         STATUS         PORTS                                     NAMES
46908b9837a2   my-custom-nginx   "/docker-entrypoint.…"   3 seconds ago   Up 2 seconds   0.0.0.0:8081->80/tcp, [::]:8081->80/tcp   my-nginx-2
6d5674f2f4fb   ubuntu            "bash"                   4 days ago      Up 3 seconds                                             my-ubuntu
aada730b830b   my-custom-nginx   "/docker-entrypoint.…"   4 days ago      Up 3 seconds   0.0.0.0:8080->80/tcp, [::]:8080->80/tcp   my-nginx

PS> curl.exe -s http://localhost:8080 | findstr h1
  <h1>Hello from my-custom-nginx!</h1>

PS> curl.exe -s http://localhost:8081 | findstr h1
  <h1>Hello from my-custom-nginx!</h1>

PS> docker port my-nginx-2
80/tcp -> 0.0.0.0:8081
80/tcp -> [::]:8081
```
→ 확인: 같은 이미지로 만든 두 컨테이너가 모두 컨테이너 내부 80번을 쓰지만, 호스트 포트(8080, 8081)가 달라 충돌 없이 둘 다 접속된다. PowerShell에서 `curl`은 `Invoke-WebRequest`의 별칭이라 `curl.exe`로 실제 curl을 호출했다.

### 4-6. 볼륨 영속성 (컨테이너 삭제 후에도 데이터 유지)
```powershell
PS> docker volume create mydata
mydata

PS> docker run -d --name vol-test -v mydata:/data ubuntu sleep infinity
f31089759b91a3c48232f468a82790e80929488566ff2a6231328aff44ffb77b

PS> docker exec vol-test bash -c "echo 'persist test' > /data/hello.txt"
PS> docker exec vol-test cat /data/hello.txt
persist test

PS> docker rm -f vol-test
vol-test

PS> docker ps -a --filter name=vol-test
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES

PS> docker run --rm -v mydata:/data ubuntu cat /data/hello.txt
persist test

PS> docker volume ls
DRIVER    VOLUME NAME
local     mydata

PS> docker volume inspect mydata --format '{{.Name}} {{.Driver}} {{.Mountpoint}}'
mydata local /var/lib/docker/volumes/mydata/_data
```
→ 확인: 데이터를 쓴 컨테이너(`vol-test`)를 삭제해 목록에서 사라졌는데도, **새 컨테이너**에 같은 볼륨을 연결하자 `persist test`가 그대로 읽혔다. 데이터는 컨테이너가 아니라 Docker가 관리하는 볼륨(`/var/lib/docker/volumes/mydata/_data`)에 저장된다.

### 4-7. 바인드 마운트 (호스트 수정 → 컨테이너 즉시 반영)
```powershell
PS> Get-Content bind-demo\index.html
<h1>bind mount v1</h1>

PS> docker run -d --name bind-test -p 8082:80 -v "${PWD}\bind-demo:/usr/share/nginx/html" nginx:alpine
c38c7f7b5cb44eb2f8e654cf9b01afeeaf1f95104fb4057a1d5fa40cb5224522

PS> curl.exe -s http://localhost:8082
<h1>bind mount v1</h1>

PS> Set-Content bind-demo\index.html "<h1>bind mount v2 (edited on host)</h1>"

PS> curl.exe -s http://localhost:8082
<h1>bind mount v2 (edited on host)</h1>

PS> docker exec bind-test cat /usr/share/nginx/html/index.html
<h1>bind mount v2 (edited on host)</h1>

PS> docker inspect bind-test --format "{{range .Mounts}}{{.Type}}: {{.Source}} -> {{.Destination}}{{end}}"
bind: C:\Users\이희광\codyssey\week1\bind-demo -> /usr/share/nginx/html
```
→ 확인: 호스트(Windows)에서 `bind-demo/index.html`을 고치자, 이미지를 다시 빌드하거나 컨테이너를 재시작하지 않았는데도 응답이 `v1` → `v2`로 바뀌었다. `docker inspect`에서 마운트 타입이 `bind`이고 호스트 폴더와 컨테이너 경로가 연결된 것을 확인했다.

### 4-8. Git 설정 및 GitHub 연동
```bash
$ git config --get user.name
kwangsclub

$ git remote -v
origin	https://github.com/kwangsclub/codyssey-1.git (fetch)
origin	https://github.com/kwangsclub/codyssey-1.git (push)

$ git log --oneline -3
c405d73 Add Docker practice docs, Dockerfile and sample site
87b6a9e Project Init

$ git push origin main
To https://github.com/kwangsclub/codyssey-1.git
   87b6a9e..c405d73  main -> main
```
→ 확인: 원격 저장소 `origin`이 GitHub에 연결되어 있고, 로컬 커밋이 `main` 브랜치로 push되었다(`87b6a9e..c405d73`). 인증은 Git Credential Manager에 저장된 GitHub 로그인으로 처리되었다.
- 저장소: https://github.com/kwangsclub/codyssey-1
- 커밋 기록: https://github.com/kwangsclub/codyssey-1/commits/main

## 5) 검증 방법 및 증거 위치

| 항목 | 검증 명령 | 확인한 내용 | 결과/증거 위치 |
|---|---|---|---|
| 실행 환경 | `git --version`, `docker --version` | Git 2.56.1, Docker 29.9.3 설치 확인 | [1) 실행 환경](#1-실행-환경) |
| 파일 조작 | `mkdir`, `touch`, `cp`, `mv`, `rm` 후 `ls -l` | 생성·복사·이동·삭제 결과가 목록에 반영됨 | [3-A) 터미널 기본 조작](#3-a-터미널-기본-조작-생성이동삭제) |
| 권한 변경 | `chmod +x`, `chmod 600`, `chmod 700` 후 `ls -l`/`ls -ld` | `-rw-r--r--` → `-rwxr-xr-x`, 실행 성공 / 폴더 `drwxr-xr-x` → `drwx------` | [3-B) 권한 변경 실습](#3-b-권한-변경-실습) |
| Docker 동작 | `docker run --rm hello-world` | `Hello from Docker!` 메시지 출력 | [4-1. hello-world 실행](#4-1-hello-world-실행) |
| ubuntu 컨테이너 | `docker exec my-ubuntu ls /`, `echo` | 루트 디렉터리 목록, echo 문자열 출력, Ubuntu 26.04.1 LTS | [4-2. ubuntu 컨테이너](#4-2-ubuntu-컨테이너-실행-및-내부-명령-수행) |
| exec vs attach | `docker ps -a --filter name=...` | exec 후 exit → `Up`, attach 후 exit → `Exited (0)` | [4-3. exec vs attach](#4-3-컨테이너-종료유지-관찰-exec-vs-attach) |
| Dockerfile 빌드 | `docker build -t my-custom-nginx .` | `naming to docker.io/library/my-custom-nginx:latest done` | [Dockerfile](Dockerfile), [3) Dockerfile 예시](#3-dockerfile-커스텀-예시) |
| 포트 매핑 접속 | `docker run -d -p 8080:80 my-custom-nginx` 후 http://localhost:8080 요청 | `Hello from my-custom-nginx!` 페이지 응답 | [site/index.html](site/index.html) |
| 환경 변수 | `docker exec my-nginx printenv APP_ENV` | `dev` 출력 | [3) Dockerfile 예시](#3-dockerfile-커스텀-예시) |
| 데몬 상태 | `docker info`, `docker version` | Server 29.9.3 응답 → 데몬 실행 중 | [1) 실행 환경](#1-실행-환경), [7-1](#7-1-docker-데몬-미실행) |
| 이미지·컨테이너 목록 | `docker images`, `docker ps -a` | 이미지 3개, 컨테이너 상태(Up/Exited) 확인 | [4-4](#4-4-이미지컨테이너-목록) |
| 포트 매핑 2회차 | `docker run -d -p 8081:80 my-custom-nginx` 후 `curl.exe` | 8080·8081 둘 다 같은 페이지 응답 | [4-5](#4-5-포트-매핑-2회차-같은-이미지-다른-호스트-포트) |
| 볼륨 영속성 | `docker rm -f vol-test` 후 새 컨테이너에서 `cat /data/hello.txt` | 컨테이너 삭제 후에도 `persist test` 유지 | [4-6](#4-6-볼륨-영속성-컨테이너-삭제-후에도-데이터-유지) |
| 바인드 마운트 | 호스트에서 `Set-Content` 후 `curl.exe` | 재빌드 없이 `v1` → `v2` 반영 | [4-7](#4-7-바인드-마운트-호스트-수정--컨테이너-즉시-반영), [bind-demo/index.html](bind-demo/index.html) |
| Git/GitHub 연동 | `git remote -v`, `git log`, `git push` | origin 연결, `87b6a9e..c405d73 main -> main` | [4-8](#4-8-git-설정-및-github-연동), [GitHub 저장소](https://github.com/kwangsclub/codyssey-1) |
| 볼륨 백업·복원 | `tar czf` 백업 → 새 볼륨에 `tar xzf` | 복원된 볼륨에서 `persist test` 확인 | [6-4](#6-4-바인드-마운트-vs-볼륨-백업-전략) |
| 포트 충돌 진단 | `netstat -ano`, `tasklist`, `docker ps --filter publish=8080` | 점유 컨테이너 `my-nginx` 확인 → 중지 후 재실행 성공 | [7-2](#7-2-포트-충돌-진단과-해결) |
| 빌드 실패 대응 | 잘못된 `COPY` 경로로 빌드 | `"/sitee": not found` 원인 확인 및 조치 | [7-3](#7-3-dockerfile-빌드-실패-대응) |

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

## 6) 개념 정리

### 6-1. 이미지 vs 컨테이너
| 구분 | 이미지 | 컨테이너 |
|---|---|---|
| 정체 | 실행에 필요한 파일·설정을 묶은 **읽기 전용 템플릿** | 이미지를 실행한 **프로세스 단위** (이미지 위에 쓰기 가능한 레이어가 추가됨) |
| 만드는 명령 | `docker build`, `docker pull` | `docker run`, `docker create` |
| 변경 | 바뀌지 않음. 바꾸려면 Dockerfile을 고쳐 **다시 빌드** | 실행 중 파일을 바꿀 수 있지만 그 컨테이너에만 적용됨 |
| 삭제 시 | `docker rmi` | `docker rm`. 컨테이너 안에서 바꾼 데이터도 함께 사라짐 |

사례: `my-custom-nginx` **이미지 하나**로 `my-nginx`(8080)와 `my-nginx-2`(8081) **컨테이너 두 개**를 만들었다([4-5](#4-5-포트-매핑-2회차-같은-이미지-다른-호스트-포트)). 컨테이너를 지우면 그 안에 쓴 데이터가 사라지기 때문에, 남겨야 하는 데이터는 볼륨에 둔다([4-6](#4-6-볼륨-영속성-컨테이너-삭제-후에도-데이터-유지)).

### 6-2. 포트 포워딩 원리
컨테이너는 Linux **네트워크 네임스페이스**로 격리되어, 호스트와 다른 자기만의 네트워크 인터페이스·IP·포트 공간을 가진다. 그래서 컨테이너 안의 nginx가 80번을 열어도 호스트의 80번과는 무관하고, 밖에서는 바로 접근할 수 없다. `-p 8080:80`을 주면 Docker가 호스트의 8080번 포트에서 요청을 받아 컨테이너 네임스페이스의 80번으로 전달하는 규칙을 만든다(Docker Desktop에서는 `com.docker.backend.exe`가 호스트 포트를 열어 전달한다. [7-2](#7-2-포트-충돌-진단과-해결)의 `netstat` 결과로 확인). 따라서 여러 컨테이너가 모두 내부 80번을 써도 **호스트 포트만 다르면** 동시에 실행할 수 있고([4-5](#4-5-포트-매핑-2회차-같은-이미지-다른-호스트-포트)), 같은 호스트 포트를 두 번 쓰면 충돌한다([7-2](#7-2-포트-충돌-진단과-해결)).

### 6-3. 호스트·컨테이너 경로 선택 기준
| 위치 | 선택 | 이유 |
|---|---|---|
| 컨테이너 경로 | 항상 **절대 경로** (`/usr/share/nginx/html`, `/data`) | 이미지가 정한 위치라서 컨테이너 안의 작업 디렉터리와 무관하게 항상 같은 곳을 가리켜야 한다 |
| Dockerfile `COPY` 원본 | **빌드 컨텍스트 기준 상대 경로** (`site/`) | `docker build .`의 `.`(저장소 루트)이 기준이라 어느 PC에서 clone해도 같다 |
| 바인드 마운트 호스트 경로 | **`${PWD}` + 상대 경로** (`"${PWD}\bind-demo"`) | `-v`는 호스트 쪽에 절대 경로를 요구한다. `${PWD}`로 현재 폴더에서 절대 경로를 만들면, `C:\Users\이희광\...`을 직접 쓰지 않아도 되어 다른 사용자·PC에서도 그대로 재현된다 |

재현 절차: 저장소를 clone → 저장소 루트로 `cd` → README의 명령을 그대로 실행. 경로에 공백·한글이 있을 수 있으므로 `-v` 값은 항상 큰따옴표로 감싼다. MSYS2/Git Bash는 `-v` 경로를 자동 변환해 오작동할 수 있어 Docker 명령은 PowerShell에서 실행했다.

### 6-4. 바인드 마운트 vs 볼륨, 백업 전략
| 구분 | 바인드 마운트 | 볼륨 |
|---|---|---|
| 저장 위치 | 호스트의 특정 폴더(`bind-demo/`) | Docker가 관리하는 영역(`/var/lib/docker/volumes/...`) |
| 장점 | 호스트에서 바로 수정·확인 가능 → 개발 중 코드 반영 | 호스트 경로·OS에 의존하지 않음, 권한 문제가 적음 |
| 위험 | 호스트 폴더를 지우면 데이터도 사라짐 | `docker volume rm`이나 `docker system prune --volumes`로 삭제될 수 있음 |
| 용도 | 소스 코드, 설정 파일 | DB 데이터, 업로드 파일 등 운영 데이터 |
| 백업 | Git 커밋 또는 폴더 복사 | 아래처럼 `tar`로 압축해 호스트에 저장 |

볼륨 백업 → 새 볼륨으로 복원 검증:
```powershell
PS> docker run --rm -v mydata:/data -v "${PWD}\backup:/backup" ubuntu tar czf /backup/mydata-backup.tar.gz -C /data .
PS> Get-ChildItem backup
Name                 Length
----                 ------
mydata-backup.tar.gz    159

PS> docker volume create mydata-restore
mydata-restore
PS> docker run --rm -v mydata-restore:/data -v "${PWD}\backup:/backup" ubuntu tar xzf /backup/mydata-backup.tar.gz -C /data
PS> docker run --rm -v mydata-restore:/data ubuntu cat /data/hello.txt
persist test
```
→ 확인: 백업 파일로 만든 새 볼륨에서 원본 데이터(`persist test`)가 그대로 복원되었다.

손실 방지 원칙:
- 중요한 데이터는 컨테이너 내부가 아니라 **볼륨**에 저장한다.
- 볼륨은 정기적으로 `tar` 백업하고, 백업 파일은 **다른 저장소**(외장 디스크, 클라우드)에 보관한다.
- `docker volume rm`, `docker system prune --volumes` 전에는 `docker volume ls`로 대상을 확인하고 먼저 백업한다.
- 백업은 만드는 것으로 끝내지 않고 **복원까지 확인**한다(위 `mydata-restore` 검증).

## 7) 트러블슈팅 기록 (가설 → 검증 → 조치)

### 7-1. Docker 데몬 미실행
- **증상**: PC를 다시 켠 뒤 `docker info` 실행 시 오류
  ```powershell
  PS> docker info
  failed to connect to the docker API at npipe:////./pipe/dockerDesktopLinuxEngine; check if the path is correct and if the daemon is running: open //./pipe/dockerDesktopLinuxEngine: The system cannot find the file specified.
  ```
- **가설**: CLI(`docker.exe`)는 설치되어 있지만, 명령을 처리할 데몬(Docker Desktop 엔진)이 실행되지 않았다.
- **검증**: `docker version`에서 Client만 `29.9.3`이 나오고 Server 칸은 비어 있었다.
- **조치**: Docker Desktop 실행 후 재확인
  ```powershell
  PS> Start-Process "$env:LOCALAPPDATA\Programs\DockerDesktop\Docker Desktop.exe"
  PS> docker version --format "Client: {{.Client.Version}} / Server: {{.Server.Version}}"
  Client: 29.9.3 / Server: 29.9.3
  ```
- **결과**: 데몬이 응답했고, 재부팅 전 컨테이너가 `Exited (255)` 상태로 남아 있어 `docker start`로 다시 실행했다([4-4](#4-4-이미지컨테이너-목록)).

### 7-2. 포트 충돌 진단과 해결
- **증상**: 8080 포트로 새 컨테이너를 띄우자 실패
  ```powershell
  PS> docker run -d --name conflict-test -p 8080:80 nginx:alpine
  docker: Error response from daemon: failed to set up container networking: driver failed programming external connectivity on endpoint conflict-test (...): Bind for 0.0.0.0:8080 failed: port is already allocated
  ```
- **가설**: 호스트 8080 포트를 다른 프로세스나 컨테이너가 이미 쓰고 있다.
- **검증**: Windows에는 `lsof`/`ss`가 없어 `netstat -ano`로 포트를 쓰는 PID를 찾고, `tasklist`로 프로세스를 확인했다.
  ```powershell
  PS> netstat -ano | findstr :8080 | findstr LISTENING
    TCP    0.0.0.0:8080           0.0.0.0:0              LISTENING       22996
    TCP    127.0.0.1:8080         0.0.0.0:0              LISTENING       16156
    TCP    [::]:8080              [::]:0                 LISTENING       22996
    TCP    [::1]:8080             [::]:0                 LISTENING       16156

  PS> tasklist /FI "PID eq 22996"
  com.docker.backend.exe       22996 Console                    4    177,512 K
  PS> tasklist /FI "PID eq 16156"
  wslrelay.exe                 16156 Console                    4     11,748 K

  PS> docker ps --filter "publish=8080" --format "table {{.Names}}\t{{.Ports}}\t{{.Status}}"
  NAMES      PORTS                                     STATUS
  my-nginx   0.0.0.0:8080->80/tcp, [::]:8080->80/tcp   Up 42 seconds

  PS> docker ps -a --filter name=conflict-test --format "table {{.Names}}\t{{.Status}}"
  NAMES           STATUS
  conflict-test   Created
  ```
  → 8080은 Docker 백엔드가 열고 있고, 그 포트를 게시한 컨테이너는 `my-nginx`였다. 실패한 `conflict-test`는 `Created` 상태로 남아 있었다.
- **조치**: 실패한 컨테이너를 지우고, 포트를 점유한 컨테이너를 멈춘 뒤 다시 실행
  ```powershell
  PS> docker rm conflict-test
  conflict-test
  PS> docker stop my-nginx
  my-nginx
  PS> docker run -d --name conflict-test -p 8080:80 nginx:alpine
  f663891fcffe8f74b098f87c6feeea4ca28a8870a9563203bc8407e88668300a
  PS> curl.exe -sI http://localhost:8080 | findstr /B "HTTP Server"
  Server: nginx/1.31.10
  PS> docker ps --filter "publish=8080" --format "table {{.Names}}\t{{.Ports}}"
  NAMES           PORTS
  conflict-test   0.0.0.0:8080->80/tcp, [::]:8080->80/tcp
  ```
- **결과**: 포트를 해제하자 새 컨테이너가 8080에서 응답했다(`Server: nginx/1.31.10`, 기본 nginx 페이지). 실습 후 `docker rm -f conflict-test`, `docker start my-nginx`로 원래 상태로 되돌렸다.
- **다른 해결책**: 기존 컨테이너를 멈출 수 없으면 다른 호스트 포트를 쓴다(`-p 8083:80`). Docker가 아닌 프로그램이 포트를 쓰고 있다면 `taskkill /PID <PID> /F`로 종료할 수 있지만, 어떤 프로그램인지 확인한 뒤에만 실행한다.

### 7-3. Dockerfile 빌드 실패 대응
- **증상**: `COPY` 경로를 잘못 쓰면(`site/` → `sitee/`) 빌드가 실패
  ```powershell
  PS> "FROM nginx:alpine`nCOPY sitee/ /usr/share/nginx/html/" | docker build --progress=plain -t build-fail-test -f - .
  #5 ERROR: failed to calculate checksum of ref ...: "/sitee": not found
  ERROR: failed to build: failed to solve: failed to compute cache key: failed to calculate checksum of ref ...: "/sitee": not found
  ```
- **가설**: `COPY` 원본 경로가 빌드 컨텍스트(`.`) 안에 없다.
- **검증**: 오류 메시지의 `"/sitee": not found`와 실제 폴더(`ls`)를 비교 → `site`만 있다.
- **조치**: Dockerfile의 경로를 `site/`로 고쳐 다시 빌드한다. 정상 빌드 로그는 [5) 검증 로그](#5-검증-방법-및-증거-위치)에 있다.
- **일반 절차**: ① `--progress=plain`으로 전체 로그를 보고 `ERROR`가 난 단계 번호(`#5`)를 찾는다. ② 그 단계의 Dockerfile 줄을 확인한다. ③ 경로·이미지 이름·네트워크 등 원인을 고친다. ④ 캐시가 의심되면 `docker build --no-cache`로 다시 빌드한다.
