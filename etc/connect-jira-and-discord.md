# Connect Jira and Discord

> 🔔 **주의사항** <br/><br/>
> 해당 기능은 무료로 제공되는 기능이라 월 100회 밖에 사용할 수 없음 <br/>

## 사용 목적

- 지라에서 생성된 티켓의 등록, 수정, 삭제 및 담당자 배정을 완료했을 때 디스코드에 메시지가 전송되게하여 티켓의 변동사항을 빠르게 확인함을 목적으로 함
- 유료 플러그인을 사용하지 않고 지라에서 제공해주는 웹훅 기능을 사용하여 인프라 비용을 절감함을 목적으로 함

## 사용 방법

### 1. 디스코드에서 웹훅 URL 받기

- 해당 기능을 추가하고 싶은 서버에서 Webhook 주소를 등록합니다.
  - 해당 기능 추가 페이지는 권한 부여된 계정만 접근가능

### 2. 지라에서 해당 웹훅 등록하기

- 좌측 프로젝트 설정 클릭
- 좌측에 Automation 클릭
- 규칙 만들기 등록
- 특정한 규칙을 실행할 수 있는 트리거를 등록
  - ex: 이슈가 만들었을 때 메시지를 전달하고 싶으면 트리거에서 **이슈 만들어짐** 을 찾은 뒤 등록
  - 이슈의 수정과 이슈의 상태변경은 서로 상관이 없음
- 이후 작업 추가를 클릭하여 진행시키고 싶은 작업을 등록
- **웹 요청 전송**을 클릭하여 해당 섹션으로 이동
  - **웹 요청 URL**에 디스코드에서 받은 Webhook 주소를 등록
  - HTTP 메서드는 POST 요청으로 등록
  - 웹 요청 본문은 사용자 정의 데이터를 이용하여 원하는 메시지가 전달 될 수 있게 커스텀
    - 자세한 내용은 [디스코드 웹 훅 가이드](https://birdie0.github.io/discord-webhooks-guide/index.html)를 참조
    - 아래는 임의로 작성한 사용자 정의 데이터 값
    ```JSON
    {
    "embeds": [
        {
            "title": "{{issue.key}} 이슈에 {{issue.assignee.displayName}}님이 담당자로 할당됐습니다.",
            "fields": [
                {
                    "name": "Assignee",
                    "value": "{{issue.assignee.displayName}}",
                    "inline": true
                },
                {
                    "name": "Number",
                    "value": "{{issue.key}}",
                    "inline": true
                },
                {
                    "name": "Title",
                    "value": "{{issue.fields.summary}}",
                    "inline": false
                },
                {
                    "name": "Status",
                    "value": "{{issue.fields.status.name}}",
                    "inline": true
                },
                {
                    "name": "URL",
                    "value": "[{{issue.key}}](https://jiraprojectname.atlassian.net/browse/{{issue.key}})",
                    "inline": false
                }
            ]
        }
    ],
    "username": "JIRA"
    }
    ```
- 등록이 완료되면 자동으로 사용 상태로 변경 사용을 원치 않으면 Automation 페이지에서 사용 토글을 클릭하여 사용을 중단상태로 변경

### 3. 주의 사항

- 현재 이슈의 상태변경이 됐을 때 메시지가 전송되는 기능은 시작 상태를 기점으로 각 3개의 Automation이 생성된 상태
