---
title: "tokburn: 하루 1억 토큰을 위한 병렬 Claude Code 환경 구축"
date: "2026-02-20"
tokens: "측정 중"
problems: 1
tools: ["Claude Code", "tmux", "tokscale", "fzf", "Python rich"]
---

## 뭘 했는가

하루 1억 토큰 목표를 달성하기 위해 tmux 기반 병렬 Claude Code 세션 환경을 구축했다. 명령어 하나(`tokburn`)로 여러 계정/프로젝트의 Claude Code 세션을 동시에 띄우고, 실시간 토큰 대시보드(`tokwatch`)로 사용량을 모니터링하는 시스템을 만들었다.

![tokburn 실행 화면: 3개의 Claude Code 세션 + tokwatch 대시보드](/ax-token-challenge/images/tokburn-dashboard.png)

### 해결한 문제

**토큰 처리량의 물리적 한계** — 세션 하나로는 활동일 기준 6,500만 토큰이 한계. 병렬 세션 + 자동화된 환경 셋업으로 1억 돌파를 노린다.

## 어떻게 했는가

1. **현재 사용량 분석**: `bunx tokscale@latest`로 토큰 사용 패턴 파악. 62일간 17억 토큰, 활동일 평균 6,500만.

2. **tmux 설정 최적화** (`~/.tmux.conf`): 마우스 지원, Alt+화살표 pane 이동, 50,000줄 스크롤백, pane 상단에 현재 경로 표시(`pane-border-status top`).

3. **tokburn 런처** (`~/bin/tokburn`): 한 명령어로 tmux 세션 생성 → pane 분할 → 계정별 프로필(`use wishket`/`use yogjin`) 자동 적용 → claude 실행. fzf로 프로젝트 폴더를 인터랙티브하게 선택하되, 최근 선택 이력을 상단에 표시.

4. **tokwatch 대시보드** (`~/bin/tokwatch`): Python + rich로 구현. `bunx tokscale@latest monthly --json --today --claude`에서 실시간 토큰 데이터를 가져오고, 디버그 로그 파싱으로 도구 사용 현황, 활성 세션 목록, 프로젝트별 메시지 수를 30초마다 갱신.

5. **멀티 계정 지원**: SSH 키 + Git 프로필을 `use` 함수로 전환. tokburn 기본값을 wishket 2 + yogjin 1로 설정.

## 사용한 프롬프트

환경 구축 과정에서 실제로 사용한 프롬프트들. 큰 방향을 잡고 → 세부 요구사항을 점진적으로 추가하는 흐름이었다.

**방향 잡기**
> 토큰을 많이 태우기 위한 나만의 환경을 구축하고싶어.

**구체화 (질문 응답 형태)**
- 작업 유형? → "위 모두 + 기타"
- 관리 방식? → "tmux 화면 분할"
- 시각화? → "터미널 대시보드"
- 세션 시작? → "명령어 하나로 한번에"

**데이터 소스 교정**
> bunx tokscale@latest 의 통계를 가져와서 사용하면 안되나?

`stats-cache.json`이 실시간이 아닌 걸 발견한 후, 직접 더 나은 데이터 소스를 제안했다.

**계정/레이아웃 지정**
> tokburn 실행하면 왼쪽 두개 터미널은 wishket계정을, 오른쪽 위 계정은 yogjin을 쓰고싶어.

**UX 개선 요청들**
> 현재 그 터미널에서 어떤 경로에서 실행중인지 그 터미널 맨위에 띄워줄수있나?

> tokburn 사용하면 yogjin 계정은 ~/repo/yogjin 내부 폴더중에서 고를 수 있는 기능을 만들어줘. wishket 계정은 ~/repo에서 고를수있도록. 그리고 각 최근에 골랐던 폴더를 선택지 맨 위에 배치되도록.

> 탭?으로 내부 폴더 보이게끔 가능?

## 깨달음

- **stats-cache.json은 실시간이 아니다.** Claude Code가 주기적으로 갱신하는 캐시 파일이라 오늘 데이터가 비어있을 수 있다. `tokscale --json --today`가 정확한 실시간 소스.
- **이미 8세션을 돌리고 있었다.** 환경 구축 전에 프로세스를 확인해보니 이미 8개의 Claude Code가 떠있었다. 문제는 세션 수가 아니라 세션을 띄우는 마찰과 모니터링의 부재.
- **Claude Code 세션 내에서 작업 디렉토리 변경은 불가능하다.** `/cd` 같은 명령이 없어서, 처음부터 올바른 경로에서 시작하는 게 중요. tokburn의 fzf 폴더 선택이 이 문제를 해결.
