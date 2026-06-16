# ReceiptAI - 영수증 OCR 기반 스마트 회계 관리

## 소개
영수증 한 장 찍으면 끝 — 단체 회계 입력·정리·보관을 
자동화하는 OCR 기반 스마트 회계 관리 서비스.
영수증을 촬영/업로드하면 Gemini AI가 상호명·날짜·금액·품목을 
자동 추출하여 회계 장부에 즉시 입력합니다.

🔗 **라이브 데모**: https://darling-donut-fec852.netlify.app

---

## 주요 기능

- 영수증 촬영/업로드로 Gemini AI 자동 분석 (상호명/날짜/금액/품목)
- Gemini API fallback 체인 (2.5-flash → 2.0-flash → 3.1-flash-lite → Tesseract.js)
- 영수증 일괄 업로드 (다중 파일 순차 분석)
- 영수증 재분석 기능 (OCR 결과 보정)
- 커스텀 카테고리 추가/삭제
- 작성자(담당자) 필드 및 작성자별 통계
- 통합 검색/필터 (상호명, 날짜 범위, 금액 범위, 카테고리, 작성자)
- 지출 통계 시각화 (카테고리별/월별/예산 대비)
- 예산 설정 및 카테고리별 예산 경고
- 월별 정산 리포트 (인쇄/PDF 출력 지원)
- 사용자별 Google Sheets 독립 연동
- 영수증 이미지 다운로드
- CSV 내보내기

---

## 페이지 구성

| 페이지 | 설명 |
|---|---|
| index.html | 서비스 소개 + 진단 퀴즈 + 랜딩 |
| receipts.html | 영수증 등록, 일괄 업로드, 재분석, 통합 검색/필터 |
| analytics-receipts.html | 회계 데이터 시각화 (카테고리/기간/예산) |
| report.html | 월별 정산 리포트 (인쇄/PDF 출력) |
| budget.html | 예산 설정 및 카테고리별 집행 현황 |
| settings.html | 카테고리 관리, 사용자 정보, Google Sheets 연동 |

---

## 시작하기

### 1. 저장소 클론

```bash
git clone https://github.com/Se0jin-Kim/receiptai.git
cd receiptai
```

### 2. config.js 설정 (필수)

config.example.js를 복사해서 config.js를 생성합니다:

```bash
cp config.example.js config.js
```

config.js를 열어서 아래 값을 입력합니다:

```javascript
const GEMINI_API_KEY = '여기에_Gemini_API_키_입력';
const APPS_SCRIPT_URL = '여기에_본인_Apps_Script_URL_입력';
// APPS_SCRIPT_URL은 선택사항입니다.
// 비워두면 데이터가 localStorage에만 저장됩니다.
```

> ⚠️ config.js는 .gitignore에 포함되어 있어 
> GitHub에 업로드되지 않습니다. 절대 커밋하지 마세요.

---

### 3. Gemini API 키 발급

1. [Google AI Studio](https://aistudio.google.com) 접속
2. 좌측 **"Get API key"** 클릭
3. **"Create API key"** 클릭
4. 생성된 키를 복사하여 config.js의 
   `GEMINI_API_KEY` 값에 붙여넣기

> 💡 무료 티어 제한:
> - Gemini 2.5 Flash: 분당 5회, 일 20회
> - Gemini 3.1 Flash Lite: 분당 15회, 일 500회
> - API 과부하 시 자동으로 다음 모델로 전환됩니다

---

### 4. 로컬 실행

HTML 파일은 로컬 서버로 실행해야 정상 작동합니다.

**VS Code Live Server 사용 (권장)**:
1. VS Code에서 프로젝트 폴더 열기
2. Live Server 확장 설치
3. index.html 우클릭 → "Open with Live Server"

**Python으로 실행**:
```bash
# Python 3
python -m http.server 5500
# 브라우저에서 http://localhost:5500 접속
```

---

### 5. Netlify 배포

1. [netlify.com](https://netlify.com) 접속 후 로그인
2. **"Add new site" → "Deploy manually"**
3. 프로젝트 폴더 전체를 드래그&드롭
4. 배포 완료 후 생성된 URL 확인

> ⚠️ config.js는 .gitignore에 포함되어 있으므로
> Netlify 배포 시 직접 포함시켜야 합니다.
> 드래그&드롭 방식으로 배포하면 config.js도 함께 업로드됩니다.

---

### 6. Google Sheets 연동 (선택사항)

연동하지 않으면 데이터가 브라우저 localStorage에만 저장됩니다.
연동하면 데이터가 본인의 Google Sheets에도 함께 저장됩니다.

#### 6-1. Google Sheets 준비

1. [sheets.google.com](https://sheets.google.com)에서 
   새 스프레드시트 생성
2. 하단 탭 이름을 **`receipts`** 로 변경
3. 1행에 아래 컬럼명 입력:

```
id  date  store_name  amount  category  payment_method  memo  created_at  items  author
```

> ⚠️ 개인 Gmail 계정을 사용하세요.
> 학교/회사 Google Workspace 계정은 보안 정책으로 
> 연동이 제한될 수 있습니다.

#### 6-2. Apps Script 설정

1. 스프레드시트 상단 메뉴 
   **"확장 프로그램" → "Apps Script"** 클릭
2. 기존 내용을 모두 지우고 아래 코드 붙여넣기:

```javascript
function doGet(e) {
  var p = e.parameter;
  var ss = SpreadsheetApp.getActiveSpreadsheet();
  var sheet = ss.getSheetByName(p.table);
  var result;

  if (!sheet) {
    result = { status: 'error', message: 'Sheet not found: ' + p.table };
  } else if (p.action === 'insert') {
    var data = JSON.parse(p.data);
    var headers = sheet.getRange(1, 1, 1, sheet.getLastColumn()).getValues()[0];
    sheet.appendRow(headers.map(function(h) {
      return data[h] !== undefined ? data[h] : '';
    }));
    result = { status: 'ok' };
  } else if (p.action === 'read') {
    var values = sheet.getDataRange().getValues();
    var headers = values[0];
    result = {
      status: 'ok',
      data: values.slice(1).map(function(row) {
        var obj = {};
        headers.forEach(function(h, i) { obj[h] = row[i]; });
        return obj;
      })
    };
  } else if (p.action === 'delete') {
    var values = sheet.getDataRange().getValues();
    var targetId = p.id;
    var deleted = false;
    for (var i = 1; i < values.length; i++) {
      if (String(values[i][0]) === String(targetId)) {
        sheet.deleteRow(i + 1);
        deleted = true;
        break;
      }
    }
    result = deleted
      ? { status: 'ok', message: 'deleted' }
      : { status: 'error', message: 'id not found' };
  } else {
    result = { status: 'error', message: 'unknown action' };
  }

  return ContentService
    .createTextOutput(p.callback + '(' + JSON.stringify(result) + ')')
    .setMimeType(ContentService.MimeType.JAVASCRIPT);
}
```

3. **Ctrl+S**로 저장
4. 우측 상단 **"배포" → "새 배포"** 클릭
5. 설정:
   - 유형: **웹 앱**
   - 실행: **나** (본인 Gmail 계정)
   - 액세스 권한: **모든 사용자**
6. **"배포"** 클릭 → 권한 승인 팝업에서 모두 허용
7. 생성된 URL 복사

> ⚠️ Apps Script 코드 수정 후에는 반드시
> "배포 관리 → 연필 아이콘 → 새 버전"으로 재배포해야 합니다.

#### 6-3. URL 입력

1. 서비스의 **Settings** 페이지 접속
2. "내 Google Sheets 연결" 섹션에서 
   복사한 URL 붙여넣기
3. **"연결 테스트"** 버튼으로 정상 작동 확인
4. **"저장"** 버튼 클릭

---

## 사용 흐름

1. index.html에서 30초 진단 퀴즈 완료
2. 사용자 정보(이름/소속) 입력 (또는 나중에 하기)
3. receipts.html에서 영수증 촬영/업로드
4. AI 자동 분석 결과 확인 및 수정
5. 카테고리/작성자 지정 후 저장
6. analytics-receipts.html에서 전체 통계 확인
7. budget.html에서 예산 설정 및 집행 현황 확인
8. report.html에서 월별 리포트 생성 및 인쇄

---

## 기술 스택

프론트엔드는 순수 HTML/CSS/JavaScript로 구현했으며
오픈소스 템플릿인 TemplateMo DayNight Admin을 
UI 기반으로 활용했습니다. 데이터는 브라우저의 
localStorage 및 사용자 Google Sheets에 저장합니다.

AI/OCR 핵심 기능은 Gemini Vision API를 사용합니다.
영수증 이미지를 직접 AI에 전송해 상호명, 날짜, 금액, 
품목을 추출하며, API 과부하 시 자동 전환되는 
fallback 체인을 구현했습니다.
모든 모델 실패 시 오픈소스 OCR 라이브러리인 
Tesseract.js가 최후 수단으로 동작합니다.

백엔드는 Google Apps Script를 서버리스 API로,
Google Sheets를 데이터베이스로 활용합니다.
CORS 우회를 위해 JSONP 방식으로 통신합니다.

배포는 Netlify를 사용했으며,
UTM 파라미터로 배포처별 유입을 추적합니다.

---

## 주의사항

- Gemini API 무료 티어는 일일/분당 호출 제한이 있습니다
- 영수증 이미지는 base64로 localStorage에 저장되므로
  많은 영수증 등록 시 브라우저 저장 공간 한도에 
  도달할 수 있습니다
- 일괄 업로드 시 Gemini API RPM 제한으로 인해
  파일당 약 1.5초의 처리 간격이 있습니다
- 학교/회사 Google Workspace 계정은 Apps Script 
  배포 시 제한될 수 있으므로 개인 Gmail 계정을 
  사용하세요

---

## 라이선스

MIT License
