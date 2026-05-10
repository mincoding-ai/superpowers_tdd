# TDD Skill Guide

- Coding Agent에서 TDD를 강제하기 위한 스킬입니다.
- Superpowers TDD를 기반으로 만들어졌습니다.
- 원본 : https://github.com/obra/superpowers/tree/main/skills/test-driven-development

## 문서 구성

두 종류의 문서로 이루어져 있습니다.

**`SKILL.md`** — TDD 방법론 가이드
Red-Green-Refactor 사이클, 철칙, 흔한 합리화와 반박, 버그 수정 예제, 검증 체크리스트를 담고 있습니다. 기능 구현이나 버그 수정을 시작할 때 로드합니다.

**`testing-anti-patterns.md`** — 테스팅 안티패턴 레퍼런스
Mock을 추가하거나 테스트 유틸리티를 작성할 때 참조합니다. 다섯 가지 안티패턴과 각각의 게이트 함수(체크리스트)를 담고 있습니다.

| 안티패턴 | 핵심 |
|---------|------|
| Mock 동작 테스트 | mock이 호출됐는지가 아닌 실제 출력을 검증하라 |
| 프로덕션의 테스트 전용 메서드 | test-utils로 분리하라 |
| 이해 없는 Mock 사용 | 부수 효과를 파악하고 나서 mock하라 |
| 불완전한 Mock | 실제 API 구조 전체를 반영하라 |
| 사후 처리 테스트 | 테스트는 구현의 일부다, 선택이 아니다 |


## 사용 방법

### Coding Agent 스킬로 등록후 사용

`SKILL.md`의 frontmatter에 `name`과 `description`이 정의되어 있어 Claude Code가 자동으로 스킬로 인식합니다.

### 레퍼런스 참조

`SKILL.md` 내에서 안티패턴 문서를 참조합니다.
SKILL.md 에는 아래와 같은 멘트를 확인할 수 있습니다.
```
When adding mocks or test utilities, read @testing-anti-patterns.md
```

### 한국어 통합본

`SKILL(Korean).md`와 `testing-anti-patterns(Korean).md`는 Java, C++, Python 세 언어의 코드 예제를 한 문서에 나란히 담고 있습니다. SKILL 내용을 이해하기 위해 만들어져 있으며, 실제 사용은 각 언어별 폴더에 있는 파일을 사용해주세요.


## 핵심 원칙

```
실패하는 테스트 없이는 프로덕션 코드도 없다
```

```
Mock은 격리를 위한 도구이지, 테스트 대상이 아니다
```
