# 터미널 에뮬레이터 — 설계 문서

## 프로젝트 개요

세션 복구 기능을 1급 기능으로 갖춘 크로스플랫폼 터미널 에뮬레이터.  
개인 사용 목적의 오픈소스 프로젝트. 내부 셸은 PowerShell(Windows) / bash(Linux).

---

## 핵심 목표

- 탭 / 분할 창 레이아웃을 JSON으로 저장하고 복구
- 각 탭에 실행할 명령어(예: `dotnet watch`, `docker compose logs -f`)를 세션에 포함
- 한글 IME 완성도 확보 (preedit 분리 처리)
- Windows 완성 → Linux 포팅 → macOS 포팅 순서

---

## 기술 스택

| 항목 | 선택 | 비고 |
|---|---|---|
| 언어 | Rust | |
| PTY | ConPTY (Windows) / posix_openpt (Linux/macOS) | 직접 구현 |
| VT 파서 | 직접 구현 | Paul Williams 스펙 기반 |
| 화면 버퍼 | 직접 구현 | Grid = Vec\<Row\>, Row = Vec\<Cell\> |
| GPU 렌더링 | OpenGL | WGL(Win) / GLX(Linux) / CGL(macOS) |
| 폰트 파싱 | ttf-parser | thin 파서, 로직 없음 |
| 글리프 캐시 | 직접 구현 | 텍스처 아틀라스 + LRU |
| IME | 직접 구현 | Win32 WM_IME_COMPOSITION / IBus / TSM |
| 세션 관리 | 직접 구현 | JSON |

### 허용 외부 의존성 (thin FFI / 선언부 수준)

```
windows-sys      Win32 / ConPTY / WGL FFI
ttf-parser       TTF/OTF 파일 파싱
raw-window-handle 윈도우 핸들 추상화 (OpenGL surface 연결)
libc             POSIX PTY (Linux/macOS)
```

로직을 포함한 크레이트(portable-pty, vte, wezterm-term, wgpu 등)는 사용하지 않음.

---

## 크레이트 구조

```
crates/
├── pty/
│   ├── src/lib.rs          # PtyProcess trait + 공통 타입
│   ├── src/windows.rs      # ConPTY 구현
│   └── src/unix.rs         # posix_openpt (Linux + macOS 공유)
│
├── vt/
│   ├── src/lib.rs          # VT 파서 + Grid (플랫폼 무관)
│   ├── src/parser.rs       # 상태 머신 (Ground/Escape/Csi/Osc...)
│   ├── src/grid.rs         # Grid / Row / Cell
│   └── src/sequence.rs     # SGR, CUP, ED 등 시퀀스 핸들러
│
├── font/
│   ├── src/lib.rs          # 폰트 로딩 + 글리프 캐시 (플랫폼 무관)
│   ├── src/rasterizer.rs   # TTF 래스터라이징
│   └── src/atlas.rs        # 텍스처 아틀라스 (shelf packing + LRU)
│
├── render/
│   ├── src/lib.rs          # Renderer trait + 공통 타입
│   ├── src/gl/             # OpenGL 드로우 코드 (플랫폼 공유)
│   │   ├── pipeline.rs     # VAO / VBO / 셰이더
│   │   └── instance.rs     # instanced draw 로직
│   └── src/context/
│       ├── wgl.rs          # Windows OpenGL 컨텍스트
│       ├── glx.rs          # Linux/X11
│       └── cgl.rs          # macOS (Phase 3)
│
├── input/
│   ├── src/lib.rs          # InputEvent 공통 타입
│   ├── src/windows.rs      # Win32 키보드 + WM_IME_COMPOSITION
│   ├── src/linux.rs        # X11/Wayland + IBus
│   └── src/macos.rs        # Cocoa IME (Phase 3)
│
└── session/
    ├── src/lib.rs          # 세션 저장/로드 (플랫폼 무관)
    └── src/schema.rs       # JSON 스키마 타입 정의

src/
└── main.rs                 # 이벤트 루프 + 크레이트 조립
```

---

## 세션 JSON 스키마

```json
{
  "name": "CXP",
  "tabs": [
    {
      "title": "API",
      "dir": "~/work/cxp/main/src/CXP.Api",
      "panes": [
        { "type": "main",    "size": 65 },
        { "type": "command", "size": 35, "command": "dotnet watch run" }
      ]
    },
    {
      "title": "Logs",
      "dir": "~/work/cxp/main",
      "panes": [
        { "type": "command", "size": 100, "command": "docker compose logs -f" }
      ]
    }
  ]
}
```

- `panes[].size` — 해당 창이 차지하는 비율 (%)
- `panes[].type` — `main`(빈 셸) / `command`(명령어 즉시 실행)
- 레이아웃은 BSP(Binary Space Partitioning) 트리로 처리

---

## 렌더링 파이프라인

```
Cell Grid (CPU)
    │
    ▼
Instance VBO  [x, y, uv_x, uv_y, fg_rgba, bg_rgba] × 셀 수
    │
    ▼
glDrawArraysInstanced()  ← 프레임당 1번
    │
    ▼
Fragment Shader  글리프 아틀라스 샘플링 + 색상 적용
```

### Cell 구조

```
Cell {
    ch:    char,    // 문자 (한글은 더블 와이드 → 2셀 점유)
    fg:    Color,
    bg:    Color,
    attrs: Attrs,   // Bold / Italic / Underline / Dim 등
}
```

### 글리프 아틀라스

- 단일 Texture2D (256×256 ~ 512×512)
- `HashMap<(char, FontStyle), UvRect>` 로 UV 좌표 조회
- 아틀라스 포화 시 LRU 교체
- 한글은 요청 시점에 래스터라이징 후 삽입

---

## VT 파서 상태 머신

Paul Williams 스펙 기반: https://vt100.net/emu/dec_ansi_parser

```
Ground → Escape → CsiEntry → CsiParam → CsiIntermediate → Dispatch
                → OscString → ...
                → SsxString → ...
```

### Phase 1 필수 구현 시퀀스

| 시퀀스 | 기능 |
|---|---|
| SGR (`ESC[...m`) | 색상, Bold, Underline |
| CUP (`ESC[H`) | 커서 이동 |
| ED (`ESC[J`) | 화면 지우기 |
| EL (`ESC[K`) | 줄 지우기 |
| DEC 모드 | Alt buffer, 커서 숨김 |
| OSC 0 | 창 타이틀 변경 |

---

## IME 처리 — 한글 핵심

preedit(조합 중)과 commit(확정)을 반드시 분리.

```
WM_IME_COMPOSITION + GCS_COMPSTR  → PTY 전송 금지, 렌더러에만 임시 표시
WM_IME_COMPOSITION + GCS_RESULTSTR → PTY stdin 전송, preedit 제거
```

PTY 전송 경로와 렌더 전용 경로를 컴파일 타임에 타입으로 구분할 것.

---

## 플랫폼 로드맵

```
Phase 1  Windows   ConPTY + WGL + Win32 IME
Phase 2  Linux     posix_openpt + GLX + IBus
Phase 3  macOS     posix_openpt 공유 + CGL + TSM
```

플랫폼 무관 크레이트: `vt`, `font`, `session`  
플랫폼 분기 크레이트: `pty`, `render/context`, `input`

---

## 개발 마일스톤

```
Week 1-2   pty      ConPTY → pwsh 연결, stdout 스트리밍 확인
Week 3-4   vt       기본 VT 파서 + Grid (색상, 커서)
Week 5-6   render   WGL 컨텍스트 초기화 + 배경색 렌더링
Week 7-8   font     TTF 로딩 + 아틀라스 + 글리프 렌더링
Week 9-10  input    키 입력 → PTY stdin + 한글 IME
           ──────────────────────────────────────────────
           동작하는 터미널 프로토타입 (Phase 1 완료 기준)
Week 11-12 탭/분할  BSP 레이아웃 + 다중 PtyInstance
Week 13-14 session  JSON 저장/복구 + CLI
```

---

## 참조 코드베이스

| 프로젝트 | 참조 목적 |
|---|---|
| [WezTerm](https://github.com/wez/wezterm) | IME 처리, 글리프 캐시 설계 |
| [Alacritty](https://github.com/alacritty/alacritty) | VT 파서 상태 머신 구조 |
| [Rio](https://github.com/raphamorim/rio) | OpenGL 렌더링 파이프라인 |

직접 구현이 원칙이며, 위 프로젝트는 레퍼런스 용도로만 참조.

---

## 차별점

세션 복구(탭 구조 + 실행 명령어)가 1급 기능으로 설계된 터미널.  
기존 프로젝트 비교:

```
Alacritty   심플함 추구, 세션 관리 없음
WezTerm     기능 방대, 복잡함
Rio         wgpu 기반

이 프로젝트  세션 복구 특화 + Windows/한글 IME 완성도
```
