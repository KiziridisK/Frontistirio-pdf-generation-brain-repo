# Frontistirio — PDF Generation Brain

> **Purpose:** Documents how PDFs are generated server-side and delivered to the frontend via AWS S3 pre-signed URLs. Reference this when working on report generation, adding new PDF types, or debugging download issues.

---

## Overview

The backend generates PDFs for two main use cases:
1. **Student Lesson Summary** — a period/date-range summary of a student's private lessons
2. **Test Cycle Summary** — a summary of tests in a cycle with enrolled students

Both use the same flow:
```
Generate PDF (pdfkit) → Upload to S3 → Return pre-signed URL → Frontend downloads directly from S3
```

---

## Tech Stack

| Concern | Library |
|---|---|
| PDF creation | `pdfkit` |
| Custom fonts | DejaVu Sans (TTF, stored in `/assets/fonts/`) |
| S3 upload | `@aws-sdk/client-s3` (`PutObjectCommand`) |
| S3 pre-signed URL | `@aws-sdk/s3-request-presigner` (`getSignedUrl`) |

---

## PDF Type 1: Student Lessons PDF

**Endpoint:** `GET /private-lessons/student/:studentId/pdf?start=<ISO>&end=<ISO>`

**Helper:** `helpers/createStudentLessonsPdf.js`

### Flow

```
1. Fetch private lessons for student in date range (from DB)
2. Call createStudentLessonsPdf(lessons, studentData)
   ├── Initialize pdfkit document
   ├── Load DejaVu font (supports Greek characters)
   ├── Add header: student name, date range
   ├── For each lesson: date, time, course, duration
   └── Return buffer
3. PutObjectCommand to S3
   Key: "student-lessons-<studentId>-<start>-<end>.pdf"
   Bucket: process.env.AWS_BUCKET_NAME
4. GetObjectCommand + getSignedUrl (expiresIn: 60s)
5. Return { downloadUrl, filename }
```

### PDF Content
- Student name and period
- Table/list of lessons: date, start time, end time, course name, duration (hours)
- Totals (total hours, potentially total cost if rate applied)

---

## PDF Type 2: Test Cycle PDF

**Endpoint:** `GET /test-cycles/:id/pdf`

**Helper:** `helpers/createTestCyclePdf.js`

### Flow

```
1. Fetch TestCycle with populated test_ids
2. For each Test: fetch grade, course, student details
3. Call createTestCyclePdf(cycle, tests)
   ├── Title: cycle description
   ├── Period date range
   ├── For each test:
   │   ├── Date, start_time, end_time
   │   ├── Courses tested
   │   ├── Grades involved
   │   └── Student list
   └── Return buffer
4. PutObjectCommand to S3
5. getSignedUrl (expiresIn: 60s)
6. Return { downloadUrl }
```

---

## Font Support (Greek Characters)

DejaVu Sans supports the Greek alphabet, which is critical for a Greek tutoring platform:

```javascript
// In pdfkit usage:
doc.font('/path/to/assets/fonts/DejaVuSans.ttf');
doc.font('/path/to/assets/fonts/DejaVuSans-Bold.ttf'); // for bold text
```

Font files are bundled in `/assets/fonts/` (~700KB each).

---

## S3 Upload Pattern

```javascript
const pdfBuffer = await createStudentLessonsPdf(lessons, studentData);

const key = `student-lessons-${studentId}-${start}-${end}.pdf`;

await s3Client.send(new PutObjectCommand({
  Bucket: process.env.AWS_BUCKET_NAME,
  Key: key,
  Body: pdfBuffer,
  ContentType: 'application/pdf'
}));

const signedUrl = await getSignedUrl(s3Client,
  new GetObjectCommand({
    Bucket: process.env.AWS_BUCKET_NAME,
    Key: key,
    ResponseContentDisposition: `attachment; filename="${filename}.pdf"`
  }),
  { expiresIn: 60 }
);

res.json({ downloadUrl: signedUrl });
```

---

## Pre-Signed URL Behavior

- URL expires in **60 seconds**
- Frontend must initiate the download immediately upon receiving the URL
- The file is not deleted from S3 after the URL expires — it remains until manually deleted
- `ResponseContentDisposition: attachment` forces browser download (not inline viewing)

---

## Adding a New PDF Type

To add a new PDF report:
1. Create `helpers/createXxxPdf.js` — accepts data, returns a `Buffer` using pdfkit
2. Add an endpoint in the relevant controller/route
3. Reuse the S3 upload + pre-signed URL pattern above
4. Use `DejaVuSans.ttf` for Greek text support
