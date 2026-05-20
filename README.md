# dante-skills

> Claude Code / Claude AI용 에이전트 스킬 모음 by [단테(Dante)](https://dante-labs.com)

[![skills.sh](https://img.shields.io/badge/skills.sh-dante--skills-blue)](https://skills.sh/dandacompany/dante-skills)

## 설치 방법

```bash
# 특정 스킬 설치 (글로벌)
skills add dandacompany/dante-skills@<skill-name> -g -y --copy -a claude-code

# 전체 스킬 설치
skills add dandacompany/dante-skills -g -y --copy -a claude-code
```

## 스킬 목록

| 스킬 | 설명 | 설치 |
|------|------|------|
| [dream](./dream/) | 프로젝트 메모리 정리 및 최적화. 메모리 파일 중복·충돌·오래된 항목을 정리하고 인덱스를 재구성합니다. | `skills add dandacompany/dante-skills@dream` |
| [paperclip-gpt55-patch](./paperclip-gpt55-patch/) | Paperclip의 codex_local 어댑터에 GPT-5.5 모델을 핫패치합니다. npx 캐시의 adapter + UI bundle을 직접 수정. | `skills add dandacompany/dante-skills@paperclip-gpt55-patch` |

## 스킬 상세

### 🌙 dream

메모리 통합 및 최적화 스킬. 뇌가 수면 중 기억을 통합하듯, 오래된 메모리를 정리하고 신호를 강화합니다.

**사용 시점:**
- 여러 세션 이후 메모리 파일이 쌓였을 때
- 삭제된 코드나 변경된 파일을 참조하는 메모리가 있을 때
- MEMORY.md 인덱스가 실제 파일과 맞지 않을 때
- 사용자가 명시적으로 메모리 최적화를 요청할 때

```bash
skills add dandacompany/dante-skills@dream -g -y --copy -a claude-code
```

### 📎 paperclip-gpt55-patch

`paperclipai/paperclip`의 `codex_local` 어댑터가 GPT-5.5 모델을 dropdown에 노출하지 않는 문제를 npx 캐시 레벨에서 직접 패치합니다. upstream에는 동일한 패치 PR이 7개 누적되어 있지만 머지가 진행되지 않아 자체 패치가 필요합니다.

**사용 시점:**
- Paperclip 운영 중인데 codex_local 모델 선택에 `gpt-5.5`가 안 보일 때
- Fast mode를 활성화했는데 `service_tier="fast"`가 적용되지 않을 때
- 새 codex_local 에이전트 default를 `gpt-5.5`로 고정하고 싶을 때

**주요 기능:**
- adapter `index.js` + UI minified bundle 동시 패치
- 자동 백업(`~/.paperclip-patches/<timestamp>/`) + `--dry-run` 사전 확인
- ESM import로 패치 결과 자동 검증, `revert.sh`로 1초 롤백
- systemd 서비스 자동 재시작 (선택)

```bash
skills add dandacompany/dante-skills@paperclip-gpt55-patch -g -y --copy -a claude-code
```

---

## 기여 / 문의

- **YouTube**: [@dante-labs](https://youtube.com/@dante-labs)
- **Discord**: [Dante Labs Community](https://discord.com/invite/rXyy5e9ujs)
- **Email**: dante@dante-labs.com
