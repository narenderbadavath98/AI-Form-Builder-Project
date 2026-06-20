# AI Form Builder — Project Documentation

## Overview

AI Form Builder is a full-stack web application that lets you:
1. **Design custom forms** with a drag-and-drop builder
2. **Upload documents** (PDF or image) and automatically extract field values
3. **Review & correct** the extracted data with confidence scores, then save it

---

## Technology Stack

| Layer | Technology |
|---|---|
| Frontend | React 18 + Vite, TypeScript, Tailwind CSS, shadcn/ui |
| Drag & Drop | dnd-kit |
| API Client | Orval (auto-generated from OpenAPI spec) + React Query |
| Backend | Node.js 24, Express 5, TypeScript |
| Database | PostgreSQL + Drizzle ORM |
| PDF Parsing | pdf-parse v1.1.1 |
| OCR (Images) | Tesseract.js |
| Validation | Zod |

---

## How to Use the App

### Step 1 — Build Form (Form Builder)

1. Open the app — you land on the **Form Builder** page (Step 1).
2. Enter a **Form Title** (e.g. "Job Application Form").
3. Add an optional **Description**.
4. Click **+ Add Field** to add fields. Each field has:
   - **Label** — what the field is called (e.g. "Full Name", "Email", "Date of Birth")
   - **Type** — Text, Textarea, Number, Date, Dropdown, or Checkbox
   - **Required** toggle — marks the field as mandatory
   - **Placeholder** — hint text shown inside the field
   - **Help Text** — small description shown below the field
   - **Options** — for Dropdown fields, add the choices
5. **Drag fields** up or down using the ≡ handle to reorder them.
6. Click a field to select it and edit its properties in the **Field Editor** panel.
7. The **Live Preview** on the right shows exactly how your form will look.
8. Optionally click **Save Template** to store this form for future reuse.
9. Click **Continue →** to proceed to Step 2.

---

### Step 2 — Upload & Extract

1. You see the **Current Form Schema** — a summary of your form fields.
2. Click the upload area or drag a file onto it:
   - Supported formats: **PDF**, **PNG**, **JPG / JPEG**
   - Max file size: **20 MB**
3. Click **Upload Document**. You'll see a green "Uploaded successfully" confirmation.
4. Click **Run Extraction** (or **Extract & Continue**).
5. The server will:
   - For PDFs: extract text using `pdf-parse`
   - For images: run OCR using `Tesseract.js`
   - Match extracted text against each field label using keyword matching and regex
6. On success you're automatically moved to Step 3.

---

### Step 3 — Review & Save

1. Each field is shown with its auto-filled value.
2. A colored badge shows the **confidence level** of each extraction:

   | Badge | Meaning |
   |---|---|
   | 🟢 High | Value found directly after a matching label (e.g. `Email: john@example.com`) |
   | 🟡 Medium | Value found on the line below a matching label |
   | 🟠 Low | Value found nearby in the document but with less certainty |

3. Fields marked **"Needs manual input"** were not found — fill them in yourself.
4. You can edit **any** field value freely, regardless of confidence.
5. Required fields that are empty will show a validation error if you try to save.
6. Click **Save Submission** to store the final data.
7. A success screen appears — click **View Submissions** to see all saved records, or **Start New Form** to begin again.

---

### Templates Page

- Browse all forms you've saved with **Save Template**.
- Click **Load** to bring a template back into the Form Builder (all fields restored).
- Click the **↓ Export** icon to download the form schema as a `.json` file.
- Click **🗑 Delete** to remove a template permanently.

---

### Submissions Page

- Browse all saved submissions in reverse chronological order.
- Each card shows the form name, submission time, and which document was uploaded.
- Click a card to expand it and see every field value along with its confidence badge.

---

## Project Structure

```
artifacts-monorepo/
├── artifacts/
│   ├── api-server/                 # Express backend
│   │   ├── src/
│   │   │   ├── routes/
│   │   │   │   ├── forms.ts        # GET/POST/PUT/DELETE /api/forms
│   │   │   │   ├── upload.ts       # POST /api/upload  (multer)
│   │   │   │   ├── extract.ts      # POST /api/extract
│   │   │   │   └── submissions.ts  # GET/POST /api/submissions
│   │   │   ├── services/
│   │   │   │   ├── fileParser.service.ts   # PDF text + OCR
│   │   │   │   └── extraction.service.ts   # Keyword/regex field matching
│   │   │   └── utils/
│   │   │       ├── regex.ts                # Email, phone, date, number extractors
│   │   │       └── textNormalization.ts    # Text cleanup helpers
│   │   └── uploads/                # Uploaded files stored here
│   └── form-builder/               # React frontend
│       └── src/
│           ├── App.tsx             # Router + navbar
│           ├── pages/
│           │   ├── BuilderPage.tsx
│           │   ├── FormsPage.tsx
│           │   └── SubmissionsPage.tsx
│           ├── components/
│           │   ├── builder/        # FormBuilderStep, FieldList, FieldEditor, StepIndicator
│           │   ├── preview/        # FormPreview
│           │   ├── upload/         # UploadExtractStep
│           │   └── review/         # ReviewSaveStep, ConfidenceBadge
│           ├── hooks/
│           │   └── use-form-builder.tsx    # Shared state across all 3 steps
│           └── types/form.ts       # TypeScript types
├── lib/
│   ├── api-spec/openapi.yaml       # Source-of-truth API contract
│   ├── api-client-react/           # Auto-generated React Query hooks
│   ├── api-zod/                    # Auto-generated Zod schemas
│   └── db/                         # Drizzle schema + DB client
│       └── src/schema/
│           ├── forms.ts            # "forms" table
│           └── submissions.ts      # "submissions" table
└── scripts/                        # Utility scripts
```

---

## Database Schema

### `forms` table
| Column | Type | Description |
|---|---|---|
| id | UUID (PK) | Unique form identifier |
| title | text | Form name |
| description | text (nullable) | Optional description |
| fields | jsonb | Array of field definitions |
| created_at | timestamp | Creation time |

### `submissions` table
| Column | Type | Description |
|---|---|---|
| id | UUID (PK) | Unique submission identifier |
| schema_id | UUID | Which form schema this belongs to |
| form_title | text | Form name at time of submission |
| values | jsonb | Key-value map of field answers |
| extracted_meta | jsonb (nullable) | Confidence + extraction info per field |
| uploaded_file_name | text (nullable) | Original filename of uploaded document |
| submitted_at | timestamp | Submission time |

---

## API Endpoints

| Method | Path | Description |
|---|---|---|
| GET | `/api/healthz` | Health check |
| GET | `/api/forms` | List all form templates |
| POST | `/api/forms` | Create a new form template |
| GET | `/api/forms/:id` | Get a specific form |
| PUT | `/api/forms/:id` | Update a form |
| DELETE | `/api/forms/:id` | Delete a form |
| POST | `/api/upload` | Upload a file (multipart/form-data) |
| POST | `/api/extract` | Extract field values from uploaded file |
| GET | `/api/submissions` | List all submissions |
| POST | `/api/submissions` | Save a new submission |

---

## Extraction Engine

The extraction engine works without any AI API key — it uses deterministic text analysis:

1. **Text extraction**: Raw text is pulled from the document (PDF text layer or OCR)
2. **Keyword generation**: For each field label (e.g. "Email Address"), synonyms and related terms are generated (e.g. "e-mail", "electronic mail", "mail")
3. **Pattern matching** (tried in order):
   - **Regex patterns** — email addresses, phone numbers, dates, numbers
   - **Colon-delimited labels** — `Email: value` → High confidence
   - **Next-line labels** — label on one line, value on the next → Medium confidence
   - **Substring proximity** — label and value found nearby in text → Low confidence
4. **Field-type specific logic**:
   - **Dropdown** — scans for option keywords in the document
   - **Checkbox** — looks for "yes", "agreed", "✓" near the label
   - **Number** — uses number/experience pattern regex
   - **Date** — tries ISO, DD/MM/YYYY, and written formats

---

## Development Commands

```bash
# Start the API server
pnpm --filter @workspace/api-server run dev

# Start the frontend (Vite dev server)
pnpm --filter @workspace/form-builder run dev

# Run full typecheck
pnpm run typecheck

# Regenerate API client after changing openapi.yaml
pnpm --filter @workspace/api-spec run codegen

# Push DB schema changes
pnpm --filter @workspace/db run push
```

---

## Environment Variables

| Variable | Description |
|---|---|
| `DATABASE_URL` | PostgreSQL connection string |
| `SESSION_SECRET` | Session signing secret |
| `PORT` | Port for each service (set by workflow config) |
