# AI Form Builder

A full-stack AI-powered Form Builder & Document Autofill web application. Users design dynamic forms with drag-and-drop fields, upload PDF/image documents, and auto-extract field values using regex-based extraction with confidence scoring — then review, edit, and save the results.

## Run & Operate

- `pnpm --filter @workspace/api-server run dev` — run the API server (port 8080)
- `pnpm --filter @workspace/form-builder run dev` — run the frontend (Vite)
- `pnpm run typecheck` — full typecheck across all packages
- `pnpm run build` — typecheck + build all packages
- `pnpm --filter @workspace/api-spec run codegen` — regenerate API hooks and Zod schemas from the OpenAPI spec
- `pnpm --filter @workspace/db run push` — push DB schema changes (dev only)
- Required env: `DATABASE_URL` — Postgres connection string

## Stack

- pnpm workspaces, Node.js 24, TypeScript 5.9
- API: Express 5 on port 8080 at `/api`
- DB: PostgreSQL + Drizzle ORM
- Validation: Zod (`zod/v4`), `drizzle-zod`
- API codegen: Orval (from OpenAPI spec)
- Build: esbuild (CJS bundle)
- Frontend: React + Vite, Tailwind CSS, shadcn/ui, dnd-kit
- File parsing: pdf-parse (PDF text), tesseract.js (OCR for images)

## Where things live

- `lib/api-spec/openapi.yaml` — the source-of-truth OpenAPI spec (forms CRUD, extract, submissions)
- `lib/db/src/schema/forms.ts` + `submissions.ts` — Drizzle ORM table definitions
- `lib/api-client-react/` — Orval-generated React Query hooks
- `lib/api-zod/` — Orval-generated Zod request/response schemas
- `artifacts/api-server/src/routes/` — Express route handlers (forms, upload, extract, submissions)
- `artifacts/api-server/src/services/` — fileParser.service.ts (pdf-parse + tesseract OCR), extraction.service.ts (regex-based field matching)
- `artifacts/api-server/src/utils/` — regex.ts, textNormalization.ts
- `artifacts/api-server/uploads/` — uploaded file storage
- `artifacts/form-builder/src/` — React frontend
  - `App.tsx` — main router + navbar
  - `pages/` — BuilderPage, FormsPage, SubmissionsPage
  - `components/builder/` — FormBuilderStep, FieldList (dnd-kit), FieldEditor, StepIndicator
  - `components/preview/` — FormPreview
  - `components/upload/` — UploadExtractStep
  - `components/review/` — ReviewSaveStep, ConfidenceBadge
  - `hooks/use-form-builder.tsx` — shared context for all 3 steps

## Architecture decisions

- Upload endpoint (`POST /api/upload`) is NOT in the OpenAPI spec — multipart/FormData doesn't codegen cleanly; frontend calls it directly with `fetch + FormData`
- Extraction is regex/keyword-based (no LLM API key required) with confidence scoring: "high" = colon-delimited label match, "medium" = next-line match, "low" = substring proximity
- Body schema names use entity-shaped naming (`FormSchemaInput`, `ExtractInput`) to avoid TS2308 collision with Orval's auto-generated `<OperationId>Body` names
- `use-form-builder.tsx` uses a React context to share state across the 3 wizard steps — must be `.tsx` not `.ts` for JSX support
- DB tables: `forms` (id, title, description, fields jsonb, created_at) and `submissions` (id, schema_id, form_title, values jsonb, extracted_meta jsonb, uploaded_file_name, submitted_at)

## Product

- **Step 1 — Build Form**: Drag-and-drop field builder with live preview. Field types: text, textarea, number, date, dropdown, checkbox. Save as reusable template.
- **Step 2 — Upload & Extract**: Upload PDF or image; server parses text (pdf-parse for PDFs, tesseract.js OCR for images) then runs extraction engine to auto-fill fields.
- **Step 3 — Review & Save**: Review extracted values with confidence badges (High/Medium/Low), edit any field manually, validate required fields, save submission.
- **Templates page**: Browse, load, export (JSON), or delete saved form schemas.
- **Submissions page**: Browse all saved submissions with extracted metadata, expand to see per-field values and confidence.

## User preferences

_Populate as you build — explicit user instructions worth remembering across sessions._

## Gotchas

- Run `pnpm run typecheck:libs` before leaf typechecks when changing `lib/db` or `lib/api-spec`
- `pdf-parse` v2 ESM needs `(module as any).default ?? module` pattern for dynamic import
- `tesseract.js` must be approved via `pnpm approve-builds` after install
- Upload route uses `multer.diskStorage` → uploads land in `artifacts/api-server/uploads/`

## Pointers

- See the `pnpm-workspace` skill for workspace structure, TypeScript setup, and package details
