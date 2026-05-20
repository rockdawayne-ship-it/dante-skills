---
name: paperclip-gpt55-patch
description: Paperclip(paperclipai/paperclip)의 codex_local 어댑터에 GPT-5.5 모델을 핫패치하는 스킬. npx 캐시에 설치된 @paperclipai/adapter-codex-local과 @paperclipai/server UI 번들을 직접 수정해 dropdown에 gpt-5.5를 노출하고 default + fast mode 화이트리스트에 추가한다. 사용자가 "paperclip gpt-5.5", "paperclip 모델 패치", "codex_local 5.5 추가", "paperclip-gpt55-patch", "paperclip 핫패치", "paperclip patch gpt5.5" 등을 요청할 때 사용. upstream PR이 누적된 채 머지되지 않아 자체 패치가 필요한 운영 환경 대상.
---

# Paperclip GPT-5.5 Hotpatch

`paperclipai/paperclip`의 `codex_local` 어댑터가 GPT-5.5 모델을 dropdown에 노출하지 않는 문제를 npx 캐시 레벨에서 직접 패치합니다.

## 배경 — 왜 핫패치인가

2026-05 기준, paperclipai upstream에는 GPT-5.5를 추가하는 PR이 **7개** 누적되어 있지만 모두 open 상태로 머지되지 않았습니다(#4357, #4646, #5022, #5575, #5898, #6044, #6045). 그 사이 paperclip을 운영 중인 사용자는 다음 한계에 부딪힙니다:

- `codex_local` 어댑터의 모델 dropdown에 `gpt-5.5` 미노출
- `DEFAULT_CODEX_LOCAL_MODEL`이 여전히 `gpt-5.3-codex`
- Fast mode(`service_tier="fast"`) 화이트리스트에 5.5 없음 → fastMode 토글해도 무시됨

Codex CLI 0.130 이상은 `--model gpt-5.5`를 정상 인식하므로, paperclip 어댑터 메타데이터만 손보면 즉시 작동합니다.

## 어떤 파일을 수정하나

npx 캐시(`~/.npm/_npx/<hash>/node_modules/`) 안의 두 파일:

| 파일 | 변경 |
|---|---|
| `@paperclipai/adapter-codex-local/dist/index.js` | `DEFAULT_CODEX_LOCAL_MODEL` → `"gpt-5.5"`, fast list에 `"gpt-5.5"` 추가 |
| `@paperclipai/server/ui-dist/assets/index-*.js` (minified) | 동일 효과를 갖는 상수 `Px`, `wNe` 치환 |

`@paperclipai/server/dist/routes/agents.js`는 ESM import만 사용하므로 자동 전파됩니다.

자세한 변경 명세는 [`references/whats-changed.md`](references/whats-changed.md) 참조.

## 실행

### 사전 조건

- paperclip이 npx 기반으로 설치/운영 중 (Docker 배포는 미지원)
- 호스트에 `bash`, `sed`, `node`가 있을 것
- (선택) `sudo systemctl` 권한 — 서비스 자동 재시작 시 필요

### Step 1 — 경로 확인 (read-only)

먼저 패치 대상이 정확히 발견되는지 확인합니다.

```bash
bash scripts/patch.sh --dry-run
```

출력 예:
```
[discover] paperclipai @ /home/dante/.npm/_npx/43414d9b790239bb/node_modules/paperclipai (2026.517.0)
[discover] adapter-codex-local @ .../@paperclipai/adapter-codex-local/dist/index.js
[discover] ui bundle @ .../@paperclipai/server/ui-dist/assets/index-DKYfatQR.js
[check] DEFAULT_CODEX_LOCAL_MODEL = "gpt-5.3-codex" (will become "gpt-5.5")
[check] CODEX_LOCAL_FAST_MODE_SUPPORTED_MODELS = ["gpt-5.4"] (will add "gpt-5.5")
[check] UI Px = "gpt-5.3-codex" (will become "gpt-5.5")
[check] UI wNe = ["gpt-5.4"] (will add "gpt-5.5")
[dry-run] no changes written
```

### Step 2 — 패치 적용

```bash
bash scripts/patch.sh
```

자동으로:
1. 대상 파일 백업 → `~/.paperclip-patches/<timestamp>/`
2. adapter `index.js` 두 줄 sed 치환
3. UI bundle minified 상수 두 곳 sed 치환
4. node로 adapter import → DEFAULT/FAST_LIST/models 배열 검증
5. (있으면) `sudo systemctl restart paperclip` 자동 재시작

### Step 3 — 검증

```bash
node -e 'import("@paperclipai/adapter-codex-local").then(m => {
  console.log("DEFAULT:", m.DEFAULT_CODEX_LOCAL_MODEL);
  console.log("FAST:", m.CODEX_LOCAL_FAST_MODE_SUPPORTED_MODELS);
  console.log("known(gpt-5.5):", m.isCodexLocalKnownModel("gpt-5.5"));
  console.log("fast(gpt-5.5):", m.isCodexLocalFastModeSupported("gpt-5.5"));
})'
```

기대 출력:
```
DEFAULT: gpt-5.5
FAST: [ 'gpt-5.4', 'gpt-5.5' ]
known(gpt-5.5): true
fast(gpt-5.5): true
```

UI는 브라우저로 paperclip을 열어 codex_local 에이전트 생성 화면의 모델 dropdown에 **gpt-5.5**가 보이는지 확인합니다.

## 롤백

```bash
bash scripts/revert.sh
```

가장 최근 백업(`~/.paperclip-patches/<timestamp>/`)을 원본 위치로 복원하고 서비스를 재시작합니다.

## 휘발성 주의

`~/.npm/_npx/<hash>/`는 다음 번 npx가 새 `paperclipai` 버전을 받으면 덮어쓰일 수 있습니다. 영구화하려면:

- **권장**: systemd unit의 `ExecStart`를 `npx --yes paperclipai@<현재버전> run --no-repair`로 **버전 핀** → npx 캐시 변경 차단
- 또는 `dante-skills/paperclip-gpt55-patch` 스킬을 시스템 부팅 hook으로 등록

## 트러블슈팅

| 증상 | 원인 / 대응 |
|---|---|
| `[discover] adapter-codex-local not found` | npx 캐시가 아직 안 받아짐. `systemctl start paperclip` 한 번 → 다시 시도 |
| 패치 후 dropdown 그대로 | 브라우저 캐시. 강제 새로고침(`Cmd+Shift+R`/`Ctrl+Shift+R`) |
| `Command not found: codex` (paperclip 로그) | systemd unit의 PATH에 `~/.npm-global/bin` 누락. `/etc/systemd/system/paperclip.service.d/path.conf` drop-in 추가 필요 |
| UI 번들 sed 치환 실패 | paperclipai 버전이 다름. `references/whats-changed.md`에서 대상 상수 패턴 확인 후 수동 적용 |

## 관련 upstream 이슈/PR

- Issue [#4481](https://github.com/paperclipai/paperclip/issues/4481) — "Add ChatGPT 5.5"
- Issue [#4405](https://github.com/paperclipai/paperclip/issues/4405) — "No model auto-detection"
- PR [#5898](https://github.com/paperclipai/paperclip/pull/5898) — 가장 깔끔한 동일 패치 PR (Claude Opus 4.7 작성)

## 모델 차원에서 무엇을 바꾸나

GPT-5.5는 `~/.codex/config.toml`에 이미 `[tui.model_availability_nux]`로 등록되어 있는 Codex CLI의 정식 인식 모델입니다. paperclip은 단지 어댑터의 메타데이터(allowlist)만 막고 있는 상태이며, 이 스킬은 그 메타데이터를 확장하는 것이지 모델 자체를 추가하는 것이 아닙니다.
