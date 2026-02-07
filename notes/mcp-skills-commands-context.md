# Claude의 MCP, Skills, Custom Commands 컨텍스트 로딩 방식 비교

## 요약

Claude가 사용하는 확장 기능(MCP, Skills, Custom Commands)은 각각 컨텍스트에 올라가는 방식과 시점이 다르다. 이 차이를 이해하면 컨텍스트 윈도우를 효율적으로 관리할 수 있다.

| 구분 | 메타데이터 (항상 로드) | 상세 내용 (on-demand) | 미사용 시 overhead |
|------|----------------------|----------------------|-------------------|
| **MCP** | tool schema (name, description, inputSchema) | tool 호출 결과 | 있음 (tool 수에 비례) |
| **Skills** | name + description (frontmatter) | SKILL.md 본문 | 있음 (하지만 작음) |
| **Custom Commands** | 없음 | 호출 시 프롬프트 주입 | 없음 |
| **CLAUDE.md** | 전체 내용 | - | 있음 (파일 크기에 비례) |

---

## MCP (Model Context Protocol)

### 설정과 실제 컨텍스트는 다르다

설정 파일(`claude_desktop_config.json` 등)에는 서버 실행 정보만 있다.

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_TOKEN": "..." }
    }
  }
}
```

이 설정 자체는 컨텍스트에 올라가지 않는다.

### 런타임에 tool schema를 받아온다

1. Claude가 시작되면 설정에 따라 MCP 서버 프로세스를 실행
2. JSON-RPC(stdio 또는 SSE)로 서버에 `tools/list` 요청
3. 서버가 제공하는 tool들의 **name, description, inputSchema**를 응답
4. 이 응답 내용이 Claude의 **tool 정의 영역에 주입**됨

즉, 컨텍스트를 차지하는 것은 설정 파일이 아니라 **서버가 런타임에 보내주는 tool schema**다.

### 주의사항

- tool을 호출하지 않아도 정의 자체가 매 대화마다 컨텍스트를 차지한다.
- tool이 많은 서버(예: GitHub, Filesystem)는 overhead가 크다.
- 자주 안 쓰는 서버는 꺼두는 것이 좋다.

---

## Skills

### 2단계 로딩 구조

**1단계 — 메타데이터 (항상 로드)**

시스템 프롬프트의 `<available_skills>` 영역에 각 스킬의 이름과 설명이 포함된다. 이 설명은 "이 스킬을 언제 사용해야 하는지"를 판단하는 트리거 조건 역할을 한다.

```xml
<skill>
  <name>docx</name>
  <description>Use this skill whenever the user wants to create, read, edit...</description>
  <location>/mnt/skills/public/docx/SKILL.md</location>
</skill>
```

**2단계 — 상세 지침 (on-demand)**

실제 `SKILL.md` 파일 본문은 해당 스킬이 필요하다고 판단될 때 `view` tool로 읽어서 컨텍스트에 추가된다. 예를 들어 "pptx 만들어줘"라는 요청이 오면 그때 `/mnt/skills/public/pptx/SKILL.md`를 로드한다.

### description의 출처: SKILL.md의 frontmatter

스킬의 description은 SKILL.md 파일 상단의 YAML frontmatter에서 추출된다.

```yaml
---
name: skill-creator
description: Create new skills, improve existing skills, and measure skill performance.
  Use when users want to create a skill from scratch...
---
```

시스템이 스킬 폴더를 스캔하여 frontmatter의 description을 추출하고, 이를 시스템 프롬프트의 `<available_skills>`에 주입하는 구조다. 따라서 사용자가 커스텀 스킬을 만들 때도 frontmatter에 description을 잘 작성하면 자동으로 트리거 판단에 활용된다.

---

## Custom Commands (Slash Commands)

- 사용자가 `/project:deploy` 같은 커맨드를 실행하기 전까지 컨텍스트에 전혀 포함되지 않는다.
- 실행하면 해당 커맨드의 프롬프트 내용이 대화에 주입된다.
- 미사용 시 컨텍스트 비용이 0이므로, 많이 만들어둬도 부담이 없다.

---

## CLAUDE.md (Project Instructions)

- 대화 시작 시 항상 전체 내용이 로드된다.
- MCP와 마찬가지로 상시 overhead가 있다.
- 파일이 길수록 컨텍스트 소비가 증가하므로, 핵심 내용 위주로 간결하게 유지하는 것이 좋다.

---

## 실무 권장사항

1. **MCP 서버는 필요한 것만 켜둔다** — tool schema만으로도 컨텍스트가 상당히 차지될 수 있다.
2. **Custom Commands를 적극 활용한다** — 호출 전까지 overhead가 없어 가장 효율적이다.
3. **CLAUDE.md는 간결하게 유지한다** — 매 대화마다 전체가 로드되므로 불필요한 내용은 줄인다.
4. **커스텀 스킬의 frontmatter description을 신경 써서 작성한다** — 트리거 정확도에 직접 영향을 준다.
