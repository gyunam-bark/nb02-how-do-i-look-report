## HOW DO I LOOK 프로젝트 개발 보고서

**작성자**: 박규남

### 구현 기능 \/ 요소 목록

| 기능/요소 이름                          |
| :-------------------------------------- |
| GITHUB ISSUE - GOOGLE SPREAD SHEET 연동 |
| GITHUB ISSUE 탬플릿 구현                |
| GITHUB PR - DISCORD 연동                |
| DTO 미들웨어 구현                       |
| TAG API 구현                            |
| LOG API 구현 / LOG 스키마 작성          |
| 프론트엔드 - 백엔드 연동 및 배포        |

### GITHUB ISSUE - GOOGLE SPREAD SHEET 연동

#### 기능 개요

```mermaid
flowchart TD
    GITHUB[GITHUB]
    GOOGLE_RUN_FUNCTION[GOOGLE RUN FUNCTION]
    GOOGLE_APPS_SCRIPT[GOOGLE APPS SCRIPT]
    GOOGLE_SPREAD_SHEET[GOOGLE SPREAD SHEET]

    GITHUB -->|WEBHOOK| GOOGLE_RUN_FUNCTION
    GOOGLE_RUN_FUNCTION -->|HTTP REQUEST| GOOGLE_APPS_SCRIPT
    GOOGLE_APPS_SCRIPT --> GOOGLE_SPREAD_SHEET
```

#### 구현 목표

1. Github Issue가 등록되면, Webhook 으로 Google Run Function 으로 이슈 목록 정보를 보낸다.

2. Google Run Function 기능과 Node.js 와 Axios 를 활용하여 Google Apps Script 로 이슈 목록 정보를 보낸다.

3. Google Apps Script 는 받은 이슈 목록 정보를 처리한 뒤, 소속된 Google Spread Sheet 에 업데이트 한다.

#### 기술 스택 및 도구

| 기술 스택                          | 서비스 제공자 |
| :--------------------------------- | ------------- |
| Github Webhook                     | 깃허브        |
| Google Run Function(Node.js/Axios) | 구글          |
| Google Apps Script(js)             | 구글          |

#### 구현 방식 / 로직

1. Github Issue -> Google Runs Function

![1](/images/1.png)

Github 설정에서 Webhook 에 어디로 보낼 지 URL 을 지정한다.

2번에서 만드는 Google Run Function 의 App URL 을 넣어주면 된다.

2. Google Run Function -> Google Apps Script

**Google Run Function**

구글에서 만든 컨테이너화 된 애플리케이션을 서버리스 방식으로 배포하고 실행할 수 있는 완전 관리형 플랫폼이다.

즉, 컨테이너 이미지로 만들어서 업로드만 하면, 구글이 알아서 자동 확장, 로드 밸런싱 들 운영 관련된 작업을 알아서 처리해준다.

매월 요청 200만개가 무료로 제공된다.

이 무료 요청 때문에 이 플랫폼을 사용하기로 결정했다.

팀 프로젝트인데, 돈 쓰기는 부담되었기 때문이다.

**새 프로젝트 생성**

프로젝트를 생성할 때, 일반적인 Run 과 Function 이 있는데, 나는 Function 을 선택했다.

Function 은 함수 하나만 동작할 때 편리하다. 이번에는 단순히 Webhook 을 받아서 Google Apps Script 의 특정 배포 URL POST 로 Issue 목록 정보를 전달 할 수 있으면 기 때문에, Function 을 사용한다.

반대로 Run 은 진짜 API 서버와 같이 서비스를 올릴 때 편리하다.

**함수 진입점**

> githubWebhook

**INDEX.JS**

반드시 함수 진입점이 메인으로 모듈화 되어야 한다.

```js
// INDEX.JS
const axios = require('axios');

exports.githubWebhook = async (req, res) => {
  console.log('GitHub Webhook 수신:', req.body);

  const scriptUrl = 'https://script.google.com/macros/s/{Apps Script URL}';

  try {
    const response = await axios.post(scriptUrl);
    console.log('Apps Script 호출 성공:', response.status);
    res.status(200).send('OK');
  } catch (err) {
    console.error('호출 실패:', err.message);
    res.status(500).send('실패');
  }
};
```

**App URL**
App URL 은 아래와 같은 형태로 생성된다.

> https://{project-id}.{region}.run.app

이 URL 을 아까 1번의 Github Webhook 대상 URL 에 적용해주면 된다.

3. Google Apps Script -> Google Spread Sheet
   Google Apps Script 는 js 거의 유사한 스크립트 언어를 사용한다.

세부적으로 차이가 있는지는 모르겠지만, 일단 이 경우를 기준으로 보았을 때는 그냥 js 를 사용하듯이 사용해도 문제가 없었다.

```js
const GITHUB_TOKEN = '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';
const GITHUB_OWNER = 'gyunam-bark';
const GITHUB_REPO = 'nb02-how-do-i-look-team1';
const GITHUB_NICKNAMES = {
  'gyunam-bark': '박규남',
  kwonnahyun0125: '권나현',
  stella62420: '김슬비',
  JINSOLdev: '김진솔',
  hippo8427: '하상준',
};

function mapGithubUser(login) {
  return GITHUB_NICKNAMES[login] || login;
}

function extractDatesFromBody(body) {
  const startMatch = body.match(/### 시작날짜\s+([0-9]{4}-[0-9]{2}-[0-9]{2})/);
  const endMatch = body.match(/### 종료날짜\s+([0-9]{4}-[0-9]{2}-[0-9]{2})/);
  return {
    startDate: startMatch ? startMatch[1] : '',
    endDate: endMatch ? endMatch[1] : '',
  };
}

function fillIssuesInSheet() {
  const sheet = SpreadsheetApp.getActiveSpreadsheet().getActiveSheet();
  const maxRows = 997;
  const allIssues = [];
  let page = 1;

  // 기존 데이터 삭제
  sheet.getRange('A6:H999').clearContent();

  while (allIssues.length < maxRows) {
    const url = `https://api.github.com/repos/${GITHUB_OWNER}/${GITHUB_REPO}/issues?state=all&per_page=100&page=${page}`;
    const options = {
      method: 'get',
      headers: {
        Authorization: `token ${GITHUB_TOKEN}`,
        Accept: 'application/vnd.github+json',
      },
      muteHttpExceptions: true,
    };

    const response = UrlFetchApp.fetch(url, options);
    const result = JSON.parse(response.getContentText());

    if (!Array.isArray(result) || result.length === 0) break;

    for (const issue of result) {
      if (!issue.pull_request) {
        const allLabels = issue.labels || [];

        // '스크럼' 라벨이 있으면 제외
        const hasScrumLabel = allLabels.some((label) => label.name === '스크럼');
        if (hasScrumLabel) continue;

        let progress = 0;
        const otherLabels = [];

        for (const label of allLabels) {
          const match = label.name.match(/^(\d{1,3})%$/);
          if (match) {
            progress = parseInt(match[1], 10);
          } else {
            otherLabels.push(label.name);
          }
        }

        const { startDate, endDate } = extractDatesFromBody(issue.body || '');

        allIssues.push({
          number: issue.number,
          author: issue.user ? mapGithubUser(issue.user.login) : '알 수 없음',
          labels: otherLabels,
          labelsStr: otherLabels.join(', '),
          title: issue.title,
          progress: progress,
          startDate: startDate,
          endDate: endDate,
        });

        if (allIssues.length >= maxRows) break;
      }
    }

    page++;
  }

  // 정렬 순서: '일정' → 일반 → '완료'
  const scheduleIssues = allIssues.filter((issue) => issue.labels.includes('일정'));
  const doneIssues = allIssues.filter((issue) => issue.labels.includes('완료') && !issue.labels.includes('일정'));
  const otherIssues = allIssues.filter((issue) => !issue.labels.includes('완료') && !issue.labels.includes('일정'));
  const sortedIssues = [...scheduleIssues, ...otherIssues, ...doneIssues];

  for (let i = 0; i < sortedIssues.length; i++) {
    const issue = sortedIssues[i];
    const row = 6 + i;

    const duration =
      issue.startDate && issue.endDate
        ? (new Date(issue.endDate) - new Date(issue.startDate)) / (1000 * 60 * 60 * 24) + 1
        : '';

    sheet.getRange(row, 1).setValue(issue.number); // A: 번호
    sheet.getRange(row, 2).setValue(issue.author); // B: 작성자
    sheet.getRange(row, 3).setValue(issue.labelsStr); // C: 라벨
    sheet.getRange(row, 4).setValue(issue.title); // D: 제목
    sheet.getRange(row, 5).setValue(issue.startDate); // E: 시작날짜
    sheet.getRange(row, 6).setValue(issue.endDate); // F: 종료날짜
    sheet.getRange(row, 7).setValue(duration); // G: 진행일수
    sheet.getRange(row, 8).setValue(issue.progress); // H: 진행률
  }
}

// GCF → WebApp용 진입점
function doPost(e) {
  fillIssuesInSheet();
  return ContentService.createTextOutput('OK');
}
```

#### 문제 및 해결 과정

1. Google Apps Script 는 무겁다.

물론, 내가 작성한 코드가 모두 지운 뒤에 다시 전부 새로 입력하기 때문에 느린 것도 있다.

근데 테스트해봤더니, 의외로 전부 지운 뒤에 다시 새로 입력하나, 변동된 부분만 수정해서 행을 변동하나 큰 속도 차이가 없었다.

오히려 이 Google Apps Script 가 동작하는 도중에 또 새로운 요청이 들어오면 꼬이는 경우가 발생해서, 아예 리셋 후 새로 그리는 걸로 꼬이는 경우를 방지했다.

속도가 느린 것에는 조건부 서식이 적용되는 것도 한 몫하는 것으로 추정된다.

조건부 서식이 없을 경우에는 2초 정도 더 빨리 완료 되었다.

#### 성과 및 결과

1. Github Wwbhook 사용법을 학습했다.

2. Google Run Function 사용법을 학습했다.

3. Google AppScript 사용법을 학습했다.

4. 팀원들은 Github Issue 만 탬플릿에 따라 작성하면 된다.

### GITHUB ISSUE 템플릿 구현

#### 기능 개요

#### 구현 목표

1. 정해진 규칙에 따라 반복되는 Issue 항목들을 템플릿화 하여 제공한다.

#### 기술 스택 및 도구

| 기술 스택             | 서비스 제공자 |
| :-------------------- | ------------- |
| Github ISSUE_TEMPLATE | 깃허브        |

#### 구현 방식 / 로직

**/.github/ISSUE_TEMPLATE/**

ISSUE_TEMPLATE 디렉토리를 만들고 내부에 .md 파일을 생성하면, Github 에서 Issue 를 생성할 때 선택지가 표시된다.

![2](/images/2.png)

**todo-template.md**

처음 설계했을 때는 Commit 컨벤션에 따른 키워드를 \[\] 에 넣을 예정이었다.

근데, 나중에 코드잇 협업 규칙들 중에 \'\[이름\]\' 으로 적는 규칙을 발견해서 TODO 대신 이름을 적는 것으로 통일했다.

코드는 따로 수정할 필요는 없어 보여서 TODO 그대로 두기로 했다.

```md
---
name: 기본 TODO
about: 시작/종료 날짜가 포함된 업무를 등록합니다.
title: '[TODO] '
labels: []
assignees: []
---

### 시작날짜

2025-06-02

### 종료날짜

2025-06-20

---

### 할 일 내용

- [ ] 할 일 1
- [ ] 할 일 2
```

#### 문제 및 해결 과정

1. default 브랜치에 올라가야 동작한다.

1팀은 기본적으로 dev 브랜치에 작업하고, 안정화 버전만 main 브랜치에 올리는 것이 규칙이었다.

그래서 todo-template.md 을 dev 까지만 올리고 테스트를 진행했는데, Issue 를 만들 때 템플릿이 표시가 되지 않았다.

이리저리 찾아보니, 이런 템플릿은 default 브랜치, 즉, 1팀 기준 main 브랜치에 올라가야 동작한다는 것을 알 수 있었다.

main 브랜치에 올리니 문제없이 동작했다.

#### 성과 및 결과

1. 모두가 동일한 템플릿으로 Issue 를 작성할 수 있다.

2. Gantt 시트가 문제없이 동기화 될 수 있다.

### GITHUB PR - DISCORD 연동

#### 기능 개요

#### 구현 목표

#### 기술 스택 및 도구

#### 구현 방식 / 로직

#### 문제 및 해결 과정

#### 성과 및 결과

### DTO 미들웨어 구현

#### 기능 개요

#### 구현 목표

#### 기술 스택 및 도구

#### 구현 방식 / 로직

#### 문제 및 해결 과정

#### 성과 및 결과

### TAG API 구현

#### 기능 개요

#### 구현 목표

#### 기술 스택 및 도구

#### 구현 방식 / 로직

#### 문제 및 해결 과정

#### 성과 및 결과

### LOG API 구현

#### 기능 개요

#### 구현 목표

#### 기술 스택 및 도구

#### 구현 방식 / 로직

#### 문제 및 해결 과정

#### 성과 및 결과

### 프론트엔드 - 백엔드 연동 및 배포

#### 기능 개요

#### 구현 목표

#### 기술 스택 및 도구

#### 구현 방식 / 로직

#### 문제 및 해결 과정

#### 성과 및 결과
