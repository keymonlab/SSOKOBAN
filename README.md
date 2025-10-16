# 🧠 SSOKOBAN — n8n & Low-code Automation Tutorials

![SSOKOBAN Banner](https://img.shields.io/badge/YouTube-Subscribe-red?logo=youtube&style=flat)  
**SSOKOBAN**은 n8n, GPT, Google API, Low-code 플랫폼을 활용한 자동화 워크플로우를 쉽게 따라 할 수 있도록 강의와 예시를 제공하는 채널입니다.

> 📺 YouTube: [SSOKOBAN 채널 바로가기](https://www.youtube.com/channel/UCtwhq7TGtoyywPUkqL393cA)  
> 🌐 목표: **“개발자도, 비개발자도, 진짜로 돌아가는 자동화를 만든다”**

---

## 📚 목차

- [소개](#-소개)
- [튜토리얼 시리즈](#-튜토리얼-시리즈)
- [예시 워크플로우: Daily Calendar Report](#-예시-워크플로우-daily-calendar-report)
  - [Node 구성](#node-구성)
  - [OpenAI Prompt 예시](#openai-prompt-예시-node-5)
- [필요한 설정](#-필요한-설정)
- [향후 업로드 예정](#-향후-업로드-예정)
- [라이선스](#-라이선스)

---

## 📝 소개

**SSOKOBAN**은 n8n, Make, Retool, Airtable, GPT API 등 **Low-code & AI 도구를 실전에서 쓰는 방법**을 다룹니다.  
단순한 기능 소개가 아닌, **“실제 돌아가는 자동화”**에 집중합니다.

특히 n8n을 중심으로:

- 🛠 Google API 연동 (Calendar, Gmail 등)  
- 🤖 OpenAI 모델과의 조합  
- 🧾 비즈니스용 보고서 자동 생성  
- 📤 Slack, Email, Notion 등 다양한 Output 연계  

를 다룹니다.

---

## ⚡ 튜토리얼 시리즈

| 시리즈 | 주제 | 주요 내용 |
|--------|------|-----------|
| 01 | Daily Calendar Automation | Google Calendar → GPT → HTML 리포트 → Gmail |
| 02 | Slack 자동 보고 | Slack 채널 요약 및 알림 자동화 |
| 03 | Notion & n8n | Notion DB 자동 업데이트 |
| 04 | ChatGPT Agent + n8n | 커스텀 에이전트와 n8n 연결 |
| 05 | Zapier 대체하기 | 유료 SaaS 자동화를 n8n으로 무료 구현 |

👉 각 시리즈는 YouTube 영상과 GitHub 예제 파일로 함께 제공합니다.

---

## 🧠 예시 워크플로우: Daily Calendar Report

이 워크플로우는 **매일 18:00**, Google Calendar에서 오늘의 이벤트를 수집하고  
OpenAI 모델로 분석해 **HTML 리포트를 생성**, Gmail로 자동 발송합니다.

### 🧭 Node 구성

| Node | 이름 | 기능 |
|------|------|------|
| 1 | Daily Trigger | 매일 18:00 실행 |
| 2 | Set Vars | 이메일 수신자 / 타임존 설정 |
| 3 | Google Calendar - List Events | 오늘 일정 가져오기 |
| 4 | Normalize Events (Function) | 이벤트 정규화(JSON 변환) |
| 5 | Analyze Calendar & Generate Report (OpenAI) | GPT-4로 분석 + HTML/JSON 생성 |
| 6 | Send Report via Gmail | 이메일 발송 (최종 단계) |

👉 자세한 구성 및 코드 예시는 `./workflows/daily-calendar-report.json` 참고

---

## 💬 OpenAI Prompt 예시 (Node 5)

아래는 OpenAI 노드(User Message) 예시입니다.  
LLM은 **이벤트 데이터만 기반으로**, HTML 리포트 + 메트릭 JSON을 반환합니다.

```text
You are a daily productivity report generator. Your task is to analyze calendar events and generate a comprehensive daily report.

IMPORTANT RULES:
1. Use ONLY the provided events data from the input - NO fabrication or hallucination allowed
2. Access the events array from the previous node: {{ $('Normalize Events').item.json.events }}
3. Access the timezone: {{ $('Set Vars').item.json.TIMEZONE }}

GENERATE A COMPLETE DAILY REPORT with the following structure:

Return a JSON object with these fields:
- html: Complete HTML report body with proper formatting
- metrics: { total_events, total_hours, meeting_count, all_day_events, busiest_hour, free_time_hours }
- problems: Array of issues (e.g., back-to-back meetings)
- wins: Array of accomplishments or highlights
- tomorrow_plan: Suggested action items for tomorrow

Ensure all data is derived from the actual events provided. If no events exist, reflect that accurately in the report.
