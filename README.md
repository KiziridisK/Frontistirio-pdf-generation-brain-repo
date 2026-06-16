# Frontistirio — PDF Generation Brain

> **Full-stack reference** for server-side PDF generation using pdfkit, S3 upload, and pre-signed URL delivery. Documents both PDF types with their exact content structure and the reusable patterns.

---

## Overview

The backend generates PDFs on-demand and serves them via AWS S3 pre-signed URLs (60 second expiry). The browser downloads directly from S3 — the backend is not involved in serving bytes.

```
Frontend POST request (entity IDs)
    ↓
Backend fetches data from MongoDB
    ↓
pdfkit renders PDF into buffer (with DejaVu fonts for Greek)
    ↓
PutObjectCommand → AWS S3
    ↓
GetObjectCommand + getSignedUrl (60s expiry)
    ↓
{ downloadUrl, filename } returned to frontend
    ↓
window.open(downloadUrl) → browser downloads from S3 directly
```

---

## Tech Stack

| Concern | Detail |
|---|---|
| PDF creation | `pdfkit` (Node.js) |
| Buffer conversion | `pdfToBuffer(doc)` — custom `Promise` wrapping `doc.on('data')` events |
| Date formatting | `dayjs` (format: `'DD-MM-YYYY HH:mm'`) |
| Greek font | DejaVu Sans TTF + Bold TTF (`/assets/fonts/`) |
| S3 upload | `@aws-sdk/client-s3` `PutObjectCommand` |
| Pre-signed URLs | `@aws-sdk/s3-request-presigner` `getSignedUrl` |

---

## PDF Type 1: Student Private Lessons PDF

**Backend trigger:** `POST /private-lessons/export-student-private-lessons`
**Helper:** `helpers/createStudentLessonsPdf.js`
**Frontend service:** `PrivateLessonsService.exportStudentLessonPrices(payload)`

### Helper signature
```javascript
async function createStudentLessonsPdf(student, lessons, rates_to_use, date_from, date_to, courses)
// Returns: Buffer
```

### PDF Structure (Greek text throughout)

```
[Header — student name, date range]
[Table header row]
  Column: Μάθημα (Course) | Από (From) | Έως (To) | Ώρες (Hours) | Κόστος (Cost)
  ─────────────────────────────────────────────────────
[Table rows — one per lesson]
  course_name | formatDateTime(date_from) | formatDateTime(date_to) | duration.toFixed(2) | cost.toFixed(2)€
  Row calculation: cost = lesson.duration × rates_to_use[course_id]
[Page break if needed — added automatically when nearing bottom]
[New table header on new page]
[Summary row — total hours, total cost]
```

### Column positions (pixel offsets)
```javascript
const COL = { COURSE: 50, FROM: 170, TO: 290, HOURS: 410, COST: 460 };
```

### Buffer conversion pattern (no external library)
```javascript
function pdfToBuffer(doc) {
  return new Promise((resolve, reject) => {
    const buffers = [];
    doc.on('data', (chunk) => buffers.push(chunk));
    doc.on('end', () => resolve(Buffer.concat(buffers)));
    doc.on('error', reject);
    doc.end();  // must call end() to flush
  });
}
```

### S3 upload details
```javascript
// Filename: "{FirstName}_{LastName}_{DD-MM-YYYY_HH:mm}_{DD-MM-YYYY_HH:mm}.pdf"
// Bucket: "logeion-private-lesson-costs" (different from educational material bucket)
// Key: "private-lesson-costs/{group_id}/{store_id}/{timestamp}_{filename}"
const uploadParams = {
  Bucket: 'logeion-private-lesson-costs',
  Key: s3Key, Body: fileContent, ContentType: 'application/pdf'
};
```

---

## PDF Type 2: Test Cycle Report PDF

**Backend trigger:** `POST /test-cycles/export-store-test-cycle-report`
**Helper:** `helpers/createTestCyclePdf.js`
**Frontend service:** `TestCycleService.exportTestCycleReport(test_cycle_id)`

### Helper signature
```javascript
async function createTestCyclePdf({ testCycle, tests })
// tests is array of: { test, grades: [{ grade, courses: [{ course, students[] }] }] }
// Returns: Buffer
```

### PDF Structure (Greek text — A4 format)
```
[Title] "Αναφορά Εξεταστικού Κύκλου" (centered, 18pt bold)
[Cycle description] "Κύκλος: {testCycle.description}"

For each test:
  [Test date header] "Τεστ – DD-MM-YYYY" (14pt bold)
  For each grade in the test:
    [Grade] "Τάξη: {grade.name}" (12pt bold)
    For each course in the grade:
      "• {course.name}"
      For each student:
        "   - {firstName} {lastName}"
  [Page break between tests — except after last]
```

### Font usage
```javascript
const doc = new PDFDocument({ margin: 50, size: 'A4' });
const fontRegular = path.join(__dirname, '../assets/fonts/DejaVuSans.ttf');
const fontBold = path.join(__dirname, '../assets/fonts/DejaVuSans-Bold.ttf');
doc.font(fontRegular);         // default
doc.font(fontBold).fontSize(18).text('...');   // headings
doc.font(fontRegular).fontSize(10).text('...'); // body
```

---

## Greek Font Support

**Both PDFs use DejaVu Sans** because pdfkit's built-in fonts (Helvetica, Times, Courier) don't support the Greek Unicode block (U+0370–U+03FF). Without DejaVu, Greek characters render as boxes or are omitted.

Font files:
- `/assets/fonts/DejaVuSans.ttf` (~757KB) — regular weight
- `/assets/fonts/DejaVuSans-Bold.ttf` (~706KB) — bold weight

These are bundled in the backend repository and loaded from disk at PDF generation time using `path.join(__dirname, '../assets/fonts/...')`.

---

## S3 Pre-Signed URL Pattern (Reusable)

```javascript
// Upload
const command = new PutObjectCommand({
  Bucket: BUCKET_NAME,
  Key: s3Key,
  Body: pdfBuffer,
  ContentType: 'application/pdf'
});
const uploadResult = await s3Client.send(command);
if (uploadResult.$metadata.httpStatusCode !== 200) throw new Error('s3_upload_failed');

// Pre-signed download URL
const get_command = new GetObjectCommand({
  Bucket: BUCKET_NAME,
  Key: s3Key,
  ResponseContentDisposition: `attachment; filename="${encodeURIComponent(filename)}"`,
  ResponseContentType: 'application/pdf'
});
const signedUrl = await getSignedUrl(s3Client, get_command, { expiresIn: 60 });

res.json({ success: true, downloadUrl: signedUrl, filename });
```

`ResponseContentDisposition: attachment` forces browser download instead of inline preview.
`ResponseContentType: 'application/pdf'` ensures correct MIME type from S3.
`encodeURIComponent(filename)` handles special characters (spaces, Greek, etc.) in filenames.

---

## S3 Bucket Names (Hardcoded)

| PDF Type | Bucket | Path prefix |
|---|---|---|
| Private lesson billing | `logeion-private-lesson-costs` | `private-lesson-costs/{group_id}/{store_id}/` |
| Test cycle report | `logeion-educational-material` | `test-cycle-reports/{group_id}/{store_id}/` |
| Educational files | `logeion-educational-material` | `educational-materials/{group_id}/{store_id}/` |

Note: `logeion-educational-material` is shared between educational files and test cycle reports. They are differentiated by path prefix.

---

## Frontend: Download Trigger Pattern

```typescript
// Generic pattern used in both ExportStudentLessonPricesComponent and TestCycleDetailsComponent:
service.exportXxx(entityId).subscribe(res => {
  if (res.success && res.downloadUrl) {
    window.open(res.downloadUrl, '_blank');
    // URL expires in 60 seconds — user must not delay
  }
});
```

---

## Adding a New PDF Type

1. **Create helper** `helpers/createXxxPdf.js`:
   ```javascript
   const PDFDocument = require('pdfkit');
   const path = require('path');
   const fontRegular = path.join(__dirname, '../assets/fonts/DejaVuSans.ttf');
   const fontBold = path.join(__dirname, '../assets/fonts/DejaVuSans-Bold.ttf');
   
   async function createXxxPdf(data) {
     const doc = new PDFDocument({ margin: 50, size: 'A4' });
     doc.font(fontRegular);
     // ... build content
     return await pdfToBuffer(doc);  // use same pdfToBuffer function
   }
   module.exports = createXxxPdf;
   ```

2. **Add controller function** in the relevant controller, following the exact S3 upload + pre-signed URL pattern above

3. **Add route** with `authMiddleware` and appropriate role middleware

4. **Add service method** in Angular (`Observable<{ success, downloadUrl, filename }>`)

5. **Add UI trigger** — button click → subscribe → `window.open(res.downloadUrl)`

---

## Pre-Signed URL Behavior & Gotchas

- **60 second expiry** — strictly enforced by AWS. The user must click/trigger the download immediately after the frontend receives the URL.
- **No automatic S3 cleanup** — PDFs accumulate in S3 indefinitely. They are regenerated on each export request rather than cached.
- **Filename encoding** — uses `encodeURIComponent()` for the `Content-Disposition` filename to handle Greek characters and spaces in filenames correctly across browsers.
- **Transaction wrapping** — both PDF generation controllers use MongoDB transactions even though no DB writes occur during the export itself. This is a defensive pattern.
