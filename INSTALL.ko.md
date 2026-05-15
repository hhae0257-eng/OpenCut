# OpenCut 로컬 설치 가이드 (Windows)

이 문서는 Windows 환경에서 OpenCut을 처음부터 끝까지 로컬에 띄우는 한국어 안내입니다. 공식 README의 영문 가이드를 보완하며, 이 포크에서 실제로 검증된 절차와 트러블슈팅을 담고 있습니다.

> macOS / Linux 사용자는 명령 일부만(특히 PowerShell → bash) 바꾸면 그대로 적용됩니다.

---

## 1. 사전 요구사항

| 도구 | 용도 | 권장 버전 |
| --- | --- | --- |
| **Bun** | 패키지 매니저 & 런타임 | 1.3+ |
| **Docker Desktop** | Postgres / Redis 컨테이너 | 4.70+ |
| **Git** | 저장소 클론 | 2.40+ |
| **Node.js** | (선택) Next.js dev 서버 백업 런타임 | **LTS 권장**, v25 비권장 (아래 트러블슈팅 참고) |

### winget으로 한 번에 설치 (Windows 10/11)

```powershell
winget install --id Oven-sh.Bun -e
winget install --id Docker.DockerDesktop -e
winget install --id Git.Git -e
winget install --id OpenJS.NodeJS.LTS -e   # 선택, LTS 권장
```

설치 후 **새 PowerShell 창**을 열어야 PATH가 갱신됩니다.

---

## 2. 저장소 클론

```powershell
git clone https://github.com/hhae0257-eng/OpenCut.git
cd OpenCut
```

> 원본 저장소를 직접 쓰고 싶다면 `https://github.com/OpenCut-app/OpenCut.git` 을 클론하세요. 단, 이 포크에 포함된 두 가지 수정(`drizzle.config.ts` 경로 버그, `react-scan` dev 오버레이 제거)을 별도로 적용해야 합니다.

---

## 3. 환경 변수 파일 생성

```powershell
Copy-Item apps/web/.env.example apps/web/.env.local
```

기본값이 아래 `docker-compose.yml` 설정과 그대로 맞물려 있어 별도 수정 없이 동작합니다.

---

## 4. DB / Redis 컨테이너 기동

Docker Desktop이 실행 중인 상태에서:

```powershell
docker compose up -d db redis serverless-redis-http
```

세 컨테이너가 모두 `healthy` 상태가 될 때까지 대기하세요.

```powershell
docker compose ps
```

| 컨테이너 | 포트 | 비고 |
| --- | --- | --- |
| `opencut-db-1` | 5432 | Postgres 17 |
| `opencut-redis-1` | 6379 | Redis 7 |
| `opencut-serverless-redis-http-1` | 8079 | Upstash 호환 HTTP 프록시 |

---

## 5. 의존성 설치

저장소 루트에서:

```powershell
bun install
```

약 150초, 패키지 1900개 이상이 설치됩니다.

---

## 6. DB 스키마 적용

```powershell
cd apps\web
bun run db:push:local
cd ..\..
```

`[✓] Changes applied` 메시지가 보이면 성공입니다.

---

## 7. 개발 서버 실행

이 가이드의 권장 실행 명령(포트 **3001**):

```powershell
cd apps\web
bun node_modules/next/dist/bin/next dev --port 3001
```

[http://localhost:3001](http://localhost:3001) 에서 OpenCut이 열립니다.

> **왜 3001인가요?** 포트 3000은 다른 dev 서버나 백그라운드 앱(예: 일부 데스크톱 앱의 내장 웹서버)이 흔히 차지하고 있어 충돌 가능성이 높습니다. 또한 위 명령은 Next를 **Bun 런타임으로 직접 실행**해서 Node 25 환경에서 발생하는 Turbopack 워커 크래시(트러블슈팅 #1 참고)도 자동으로 회피합니다.
>
> 원본 저장소의 표준 명령은 `bun dev:web`(포트 3000)이며, Node LTS + 포트 3000 가용 환경에서는 그대로 동작합니다.

---

## 8. 종료 / 재실행

```powershell
# 종료: Ctrl+C 로 dev 서버 중지 후
docker compose down

# 다시 시작
cd C:\Users\User\OpenCut
docker compose up -d db redis serverless-redis-http
cd apps\web
bun node_modules/next/dist/bin/next dev --port 3001
```

볼륨(`postgres_data`)은 그대로 유지되므로 DB 데이터는 보존됩니다.

---

## 트러블슈팅

### 1) `bun dev:web` 실행 시 Turbopack 패닉 (HTTP 500)

증상 — 페이지 요청 시 다음 로그가 반복:
```
FATAL: An unexpected Turbopack error occurred. A panic log has been written to ...
Caused by:
- evaluate_webpack_loader failed
- failed to receive message
- 현재 연결은 원격 호스트에 의해 강제로 끊겼습니다. (os error 10054)
```

원인 — Node.js v25(비-LTS) + Next 16 Turbopack 조합에서 PostCSS 워커 프로세스가 죽는 현상.

해결책 (둘 중 택1):

**A. Node LTS로 다운그레이드 (권장)**
```powershell
winget install --id OpenJS.NodeJS.LTS -e
```
설치 후 모든 터미널을 닫고 다시 시도하세요.

**B. Next를 Bun 런타임으로 직접 실행 (Node를 건드리지 않는 회피책)**

이 가이드의 7단계 명령이 정확히 이 회피책을 적용한 것입니다:
```powershell
cd apps\web
bun node_modules/next/dist/bin/next dev --port 3001
```
Bun이 PostCSS 워커도 자신의 런타임으로 처리하여 크래시가 사라집니다.

### 2) 포트 3001도 점유됨 (`EADDRINUSE`)

이 가이드의 기본 포트는 3001입니다. 그마저 다른 프로세스가 쓰고 있다면 임의의 빈 포트로 바꿔주세요(예: 3002).
```powershell
cd apps\web
bun node_modules/next/dist/bin/next dev --port 3002
```
이후 [http://localhost:3002](http://localhost:3002) 로 접속하세요.

### 3) `db:push:local` 실패 — `No schema files found for path config ['./src/lib/db/schema.ts']`

이 포크에는 이미 수정되어 있지만, 원본 저장소에서 발생하면 `apps/web/drizzle.config.ts`의 `schema` 값을 다음과 같이 고쳐주세요:
```ts
schema: "./src/db/schema.ts",
```

### 4) 마우스를 움직일 때마다 보라색 박스/라벨이 화면을 뒤덮음

이 포크에서는 비활성화되어 있습니다. 원본 저장소에서 동일 증상이 보이면 `apps/web/src/app/layout.tsx`의 `react-scan` `<Script>` 블록을 제거하세요.

### 5) Docker Desktop이 시작되지 않음

- Windows 기능에서 **WSL 2** 와 **가상 머신 플랫폼**이 켜져 있어야 합니다.
- Docker Desktop을 처음 실행할 때 약관 동의와 한 차례 재부팅이 필요할 수 있습니다.

---

## 이 포크가 원본과 다른 점

| 파일 | 변경 |
| --- | --- |
| `apps/web/drizzle.config.ts` | `schema` 경로를 실제 위치(`./src/db/schema.ts`)로 수정 |
| `apps/web/src/app/layout.tsx` | dev 빌드에서 자동 주입되던 `react-scan` 오버레이 제거 |
| `INSTALL.ko.md` (이 파일) | 한국어 설치 가이드 추가 |

---

## 라이선스 / 원저작자

OpenCut은 OpenCut 팀의 오픈소스 프로젝트이며 MIT 형식 라이선스를 따릅니다. 이 포크 역시 동일한 라이선스로 배포되며, 원본 `LICENSE` 파일은 그대로 유지됩니다. 자세한 내용은 [원본 저장소](https://github.com/OpenCut-app/OpenCut)와 루트의 `LICENSE` 파일을 참고하세요.
