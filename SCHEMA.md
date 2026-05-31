# Wiki Schema

## Domain
카카오톡 발주방 — 식자재/물품 발주 채팅방의 주문 패턴 분석 및 기록

## Conventions
- File names: lowercase-korean, hyphens, no spaces (e.g., `아리식당.md`)
- Every wiki page starts with YAML frontmatter (see below)
- Use `[[wikilinks]]` to link between pages (minimum 2 outbound links per page)
- When updating a page, always bump the `updated` date
- Every new page must be added to `index.md` under the correct section
- Every action must be appended to `log.md`
- **발주 명령어**는 `명령어:` 블록으로 따로 표기
- 메시지 타입: 일반텍스트(1), 명령어, 사진(16386), 첨부파일

## Frontmatter
```yaml
---
title: Page Title
created: YYYY-MM-DD
updated: YYYY-MM-DD
type: entity | concept | comparison | query | summary
tags: [from taxonomy below]
sources: [raw/chatlogs/room-name.md]
---
```

## Tag Taxonomy
- **발주방:** restaurant, open-chat, order-channel
- **주문:** order, order-add, order-cancel, order-confirm
- **사용자:** host, buyer, bot
- **메시지:** command, photo, text, system-message
- **상태:** active, inactive, new

## Entity Pages (발주방)
One page per ordering room. Include:
- 방 이름과 ID (채팅방 고유 ID)
- 호스트 ID
- 참여자 목록
- 주문 방식 (명령어, 템플릿 등)
- 주문 추가/변경 패턴
- 취소 방식
- 발송 확인 절차
- 첫 발견일 / 마지막 활동일

## Concept Pages
One page per concept. Include:
- **주문-패턴:** 발주방에서 사용되는 공통 주문 유형 정의
- **명령어-체계:** 발주방 봇 명령어 정리
- **발송-흐름:** 발송 확인부터 결과까지의 프로세스

## Update Policy
When new information conflicts with existing content:
1. Check the dates — newer sources generally supersede older ones
2. If genuinely contradictory, note both positions with dates and sources
3. Mark the contradiction in frontmatter
