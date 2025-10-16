```markdown
# 🧠 n8n + GPT로 만드는 Daily Calendar Report 자동화 가이드

![thumbnail](./assets/banner.png)

## 📋 목차

- [개요](#개요)
- [기존 일정 관리의 문제점](#기존-일정-관리의-문제점)
- [Daily Calendar Report 워크플로우 소개](#daily-calendar-report-워크플로우-소개)
- [Node별 구성 및 역할](#node별-구성-및-역할)
  - [Node 1: Daily Trigger](#node-1-daily-trigger)
  - [Node 2: Set Vars](#node-2-set-vars)
  - [Node 3: Google Calendar - List Events](#node-3-google-calendar---list-events)
  - [Node 4: Normalize Events](#node-4-normalize-events)
  - [Node 5: Analyze Calendar and Generate Report (OpenAI)](#node-5-analyze-calendar-and-generate-report-openai)
  - [Node 6: Send Report via Gmail](#node-6-send-report-via-gmail)
- [워크플로우 구조 요약](#워크플로우-구조-요약)
- [적용 시나리오](#적용-시나리오)
- [결론](#결론)

---

## 개요

캘린더에 일정이 많아질수록 하루를 객관적으로 되돌아보고 개선 포인트를 찾기 어려워집니다.  
매일 회의와 집중 시간, 컨텍스트 전환(context switch) 등을 수동으로 정리하는 것은 번거롭고,  
이 때문에 **일정 관리 자동화**의 수요가 점점 높아지고 있습니다.

이 가이드는 n8n과 OpenAI를 이용해, 매일 18:00에 **자동으로 일정 데이터를 분석하여 HTML 보고서를 생성하고 메일로 발송**하는 방법을 다룹니다.

---

## 기존 일정 관리의 문제점

### 일정/회의 관리에서 흔히 발생하는 문제점

**1. 시간 사용에 대한 피드백 부족**  
- 회의와 Deep Work 시간이 적절히 배분되었는지 알기 어려움  
- 하루가 바쁘게 지나갔지만 무엇을 했는지 정리되지 않음

**2. 회의의 질적 문제 파악 곤란**  
- 백투백(Back-to-back) 회의나 안건 없는 미팅을 찾기 어려움

**3. 일정 패턴 분석의 자동화 부재**  
- 수동 리포트는 번거롭고 재현 불가  
- 데이터를 모아도 가공이 어려워 실시간 인사이트 도출이 안 됨

---

## Daily Calendar Report 워크플로우 소개

이 워크플로우는 아래의 6개 노드로 구성됩니다.

```

[Node 1] Daily Trigger
↓
[Node 2] Set Vars
↓
[Node 3] Google Calendar - List Events
↓
[Node 4] Normalize Events
↓
[Node 5] Analyze Calendar and Generate Report (OpenAI)
↓
[Node 6] Send Report via Gmail ✅

````

매일 18:00에 Google Calendar에서 당일 이벤트를 불러오고,  
GPT-4(o)를 이용해 일정 데이터를 분석한 후 HTML 리포트를 생성,  
Gmail로 자동 발송합니다.

---

## Node별 구성 및 역할

### Node 1: Daily Trigger

**설명**  
매일 18:00에 워크플로우를 실행하는 스케줄 트리거입니다.

**설정 요약**
- Interval: Days  
- Days Between Triggers: 1  
- Hour: 18  
- Minute: 00

```text
Add a Schedule Trigger node that runs every day at 18:00 (6 PM). Configure it to execute daily at a specific time.
````

---

### Node 2: Set Vars

**설명**
보고서를 수신할 이메일 주소(`RECIPIENT_EMAIL`)와 기준 시간대(`TIMEZONE`)를 설정합니다.

**예시 값**

* RECIPIENT_EMAIL: `your@email.com`
* TIMEZONE: `Asia/Seoul`

```text
Add a Set node named "Set Vars" that exposes two variables: RECIPIENT_EMAIL and TIMEZONE.
```

---

### Node 3: Google Calendar - List Events

**설명**
Google Calendar API를 통해 **오늘의 모든 일정**을 불러옵니다.

**중요 설정**

* Return All: true
* Time Min:

```js
{{ $now.setTimezone($json.TIMEZONE || $vars.TIMEZONE).startOf('day').toISO() }}
```

* Time Max:

```js
{{ $now.setTimezone($json.TIMEZONE || $vars.TIMEZONE).endOf('day').toISO() }}
```

---

### Node 4: Normalize Events

**설명**
Google Calendar의 원본 데이터를 GPT가 분석하기 좋은 JSON 구조로 정제합니다.

**출력 필드 예시**

```json
{
  "summary": "회의 제목",
  "description": "내용",
  "start": "2025-10-16T09:00:00+09:00",
  "end": "2025-10-16T10:00:00+09:00",
  "duration_minutes": 60,
  "attendee_count": 3,
  "html_link": "https://calendar.google.com/...",
  "location": "",
  "status": "confirmed",
  "organizer": "홍길동",
  "all_day": false
}
```

---

### Node 5: Analyze Calendar and Generate Report (OpenAI)

**설명**
GPT-4(o)를 활용하여 이벤트 데이터를 분석하고,
HTML 리포트 + 메트릭(JSON)을 생성합니다.

**모델 설정**

* Model: gpt-4o
* Temperature: 0.1
* JSON mode: 가능 시 활성화

**User Prompt 예시 (요약)**

```
You are a daily productivity report generator.
Use ONLY the provided events data from the input.
Access events: {{ $('Normalize Events').item.json.events }}
Access timezone: {{ $('Set Vars').item.json.TIMEZONE }}
Generate HTML + metrics + problems + wins + tomorrow_plan
```

**리포트 구조**

* 헤더: 날짜, 총 회의 시간, 집중 시간 등 요약
* 이벤트 리스트(시간순)
* 시간 분석(총 시간, 빈 시간, 가장 바쁜 시간 등)
* 문제점(Back-to-back 회의 등)
* Wins/성과
* 내일 일정 제안

---

### Node 6: Send Report via Gmail

**설명**
Node 5에서 생성한 HTML 보고서를 이메일로 발송합니다.

**설정**

* To:

```text
{{ $env.RECIPIENT_EMAIL || $json.RECIPIENT_EMAIL || $vars.RECIPIENT_EMAIL }}
```

* Subject:

```
Daily Calendar Analytics & Productivity Report – {{ $now.format('YYYY-MM-DD') }}
```

* Body:

```
{{ $json.html }}
```

* HTML: true

---

## 워크플로우 구조 요약

| Node | 이름                        | 역할                  |
| ---- | ------------------------- | ------------------- |
| 1    | Daily Trigger             | 매일 18시 자동 실행        |
| 2    | Set Vars                  | 환경 변수 설정            |
| 3    | Google Calendar           | 일정 불러오기             |
| 4    | Normalize Events          | 데이터 정제              |
| 5    | Analyze Calendar (OpenAI) | 일정 분석 + HTML 리포트 생성 |
| 6    | Gmail                     | 메일 발송 (최종)          |

---

## 적용 시나리오

* 🧭 **개인 일정 분석 자동화**
* 👥 **팀 회의/집중시간 밸런스 리포트**
* 📝 **리더/매니저의 일일 회고 자동화**
* 🧠 **Context Switching 패턴 파악**
* 🕒 **Back-to-back 회의 탐지 및 개선 제안**

---

## 결론

이 워크플로우는 “하루 일정 분석 → 리포트 작성 → 발송”의 전 과정을 완전히 자동화합니다.
매일 반복되는 일정 정리 작업을 자동화함으로써,
사용자는 **데이터 기반의 피드백 루프**를 쉽게 확보할 수 있습니다.

### 핵심 포인트

1. ✅ 일정 데이터 자동 수집 및 정규화
2. 🤖 GPT를 활용한 메트릭 계산 & 리포트 자동화
3. ✉️ Gmail 자동 발송으로 워크플로우 완결

---

## 참고 자료

* [n8n 공식 문서](https://docs.n8n.io/)
* [Google Calendar API](https://developers.google.com/calendar)
* [OpenAI GPT-4 API](https://platform.openai.com/docs/)
* [Gmail API](https://developers.google.com/gmail/api)

```

---

### ✅ 정리

이 문서는 예시로 주신 Perplexity Search API 가이드와 동일한 구조(목차, 시나리오, Step by Step, 코드 블록)를 따르며,  
GitHub의 Docs 또는 Notion에서도 깔끔하게 렌더링됩니다.

원하신다면 이 문서를 `.md` 파일로 정리해서 `docs/01-daily-calendar-workflow.md`로 내보내 드릴 수 있습니다. 그렇게 할까요? 📝📂
```
