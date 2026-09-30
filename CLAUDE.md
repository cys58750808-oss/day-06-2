# CLAUDE.md

이 파일은 이 저장소에서 작업하는 Claude Code(claude.ai/code)에게 제공하는 안내 문서입니다.

## 프로젝트

영어 단어 플래시카드 앱(영어 단어장)으로, 한 페이지짜리 웹앱입니다. 앱 전체가 [index.html](index.html) 하나에 들어 있으며 — 빌드 단계, package.json, 프레임워크가 전혀 없습니다. 전체 기능 명세는 [PRD.md](PRD.md), 단계별 개발 계획은 [STEPS.md](STEPS.md)를 참고하세요 (1단계 추가/체크/삭제 → 2단계 탭/개수 → 3단계 디자인). 현재 세 단계 모두 구현이 완료된 상태입니다.

## 명령어

이 저장소에는 빌드, 린트, 테스트 도구가 없습니다. 변경 사항을 미리 보려면:

```bash
npx serve -l 5173 .
```

(`.claude/launch.json`의 `wordbook` 설정과 동일하며 포트는 5173입니다.) `index.html`은 단일 정적 파일이므로 별도의 컴파일 없이 브라우저를 새로고침하면 바로 변경 사항이 반영됩니다.

## 아키텍처

- **단일 파일 구조**: 마크업, `<style>`, `<script>`가 모두 `index.html` 하나에 들어 있습니다. 번들러나 모듈 시스템이 없으며, 사용자가 구조 변경을 요청하지 않는 한 이 구조를 유지하세요.
- **백엔드**: Supabase `day 06` 프로젝트(project ref `qlqusyzaezadkagvefqj`)를 사용하며, CDN에서 불러온 `@supabase/supabase-js`(`<script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2/...">`)로 클라이언트에서 직접 접근합니다. Supabase URL과 anon/publishable 키는 `<script>` 블록에 하드코딩되어 있는데, 클라이언트 전용 앱에서 공개용 anon 키를 그대로 쓰는 것은 의도된 설계이지 키가 유출된 것이 아닙니다.
- **테이블**: `public.words` — `id`(uuid), `word`(text), `meaning`(text), `memorized`(boolean, 기본값 false), `created_at`(timestamptz). RLS가 활성화되어 있고 `anon` 역할에 대해 `for all`을 허용하는 정책이 걸려 있습니다(로그인 없는 단일 사용자 앱). 여기에 더해 `grant select/insert/update/delete ... to anon`도 별도로 필요합니다 — RLS 정책만으로는 부족하고, anon 역할에 테이블 단위 권한(grant)도 함께 부여해야 동작합니다.
- **상태 흐름**: `loadWords()`가 Supabase에서 전체 행을 가져와 메모리상의 `allWords` 배열에 담은 뒤 `renderCurrentView()`를 호출합니다. `addWord`, `toggleMemorized`, `deleteWord` 등 모든 변경 작업은 Supabase에 먼저 반영한 뒤 `loadWords()`를 다시 호출해 동기화하는 방식이며, 낙관적(optimistic) 로컬 업데이트는 사용하지 않습니다. 탭 필터링([전체]/[외울 것])은 별도의 Supabase 쿼리가 아니라 클라이언트 쪽에서 `allWords`를 걸러서 처리합니다.
- **렌더링**: `renderWords()`는 렌더링할 때마다 `#word-list`의 DOM을 처음부터 다시 만듭니다. 사용자가 입력한 단어/뜻 텍스트는 항상 `textContent`로만 넣고(`innerHTML` 사용 안 함) 있어서, 입력값이 항상 글자 그대로 표시되고 HTML로 해석되지 않습니다.
- **입력 검증**: 추가 폼은 단어와 뜻을 trim한 값이 모두 비어 있지 않을 때만 `addWord()`를 호출합니다. 빈 칸으로 제출하면 클라이언트 단에서 조용히 무시되며(별도 안내 메시지 없음), Supabase 쪽의 `not null` 제약에 의한 처리는 아닙니다.
