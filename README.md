# 자산 QR 스캐너

회사 자산 QR을 모바일로 스캔해 Google Sheets에 자동 기록하는 PWA.

## 구조

- `index.html` — 단일 HTML (html5-qrcode 기반 스캐너)
- Apps Script 웹앱 (별도 배포) — 시트 append + 중복 체크

## 기능

- 📷 모바일 카메라로 QR 연속 스캔
- 🔢 QR에서 숫자만 추출 (URL이어도 자동 파싱)
- ✅ 신규 등록: 초록 알림 + 비프음 + 짧은 진동
- ⚠️ 중복 감지: 빨간 알림 + 저음 비프 + 긴 진동
- 📊 실시간 카운터 (총/신규/중복)
- 📋 CSV 로그 다운로드 (오프라인 백업용)
- 🔁 클라이언트 측 2초 debounce (같은 QR 연속 스캔 방지)

## 배포 순서

### 1. Google Sheet 생성

- 새 스프레드시트 생성 → 시트명 `자산` 으로 변경
- 1행 헤더: `A1=스캔시각`, `B1=자산번호`, `C1=결과`
- 시트ID 복사 (URL `/d/{ID}/edit` 부분)

### 2. Apps Script 웹앱 배포 (츄릭핑 형 담당)

```javascript
const SHEET_ID = "여기에_시트ID";

function doPost(e) {
  try {
    const { id } = JSON.parse(e.postData.contents);
    if (!id) return _json({ status: 'error', msg: '자산ID 없슴' });

    const sheet = SpreadsheetApp.openById(SHEET_ID).getSheetByName('자산');
    const lastRow = sheet.getLastRow();
    const ids = lastRow < 2 ? []
      : sheet.getRange(`B2:B${lastRow}`).getValues().flat().filter(String).map(String);

    if (ids.includes(String(id))) {
      return _json({ status: 'duplicate', id });
    }
    sheet.appendRow([new Date(), id, '신규']);
    return _json({ status: 'ok', id });
  } catch (err) {
    return _json({ status: 'error', msg: err.message });
  }
}

function _json(obj) {
  return ContentService.createTextOutput(JSON.stringify(obj))
    .setMimeType(ContentService.MimeType.JSON);
}
```

- Deploy → New deployment → Type: Web app
- Execute as: Me
- Who has access: Anyone
- Deploy → 웹앱 URL 복사

### 3. 스캐너 페이지 연결

`index.html` 상단 `WEBAPP_URL` 값을 복사한 URL로 교체.

### 4. 배포 (GitHub Pages 기준)

```bash
cd asset-qr-scanner
git init
git add .
git commit -m "Initial: asset QR scanner"
gh repo create treenod-patrick/asset-qr-scanner --public --source=. --push
gh api -X POST repos/treenod-patrick/asset-qr-scanner/pages \
  -f build_type=legacy \
  -f 'source[branch]=main' \
  -f 'source[path]=/'
```

HTTPS 필수 (카메라 권한 정책). GitHub Pages는 자동 HTTPS.

## 사용 방법

1. 모바일에서 배포된 URL 열기 (홈 화면에 추가하면 편함)
2. 카메라 권한 허용
3. 자산 QR을 화면 가운데 사각형에 맞추면 자동 인식
4. 신규(초록)/중복(빨강) 알림 확인 후 다음 QR로 이동
5. 세션 끝나면 CSV 다운로드로 로컬 백업

## 주의사항

- Content-Type을 `text/plain`으로 POST → CORS preflight 회피 (Apps Script 웹앱 표준 패턴)
- 숫자 추출 정규식: `/(\d{4,})/` — 4자리 이상 숫자 블록. QR이 URL이어도 ID만 뽑아냄
- 중복 체크는 서버(시트) 기준. 클라이언트 debounce는 단순히 같은 QR 연타 방지용
