# CAO: Technical Service Order Automation Pipeline

Production-grade, event-driven data pipeline that automates the full lifecycle of hardware technical service orders from field capture to client delivery for **Compuservicios Alfa y Omega S.A.S.** (Bogotá, Colombia).

**8+ hours/week of manual administrative work eliminated. Zero order loss since deployment.**

---

## Architecture

```
[Field Technician]
       │
       ▼
[clientes.html SPA] ════ Google Sheets CSV (real-time) ════► Client lookup + branch selection
       │
       ▼ URL params prefill
[Fillout Form]
       │
       ▼ POST /webhook
┌─────────────────────────────────────────────────────────────┐
│                    MAIN WORKFLOW (n8n)                       │
│                                                             │
│  Webhook Trigger                                            │
│       │                                                     │
│       ├══► Extract date → Generate monthly folder path      │
│       ├══► Serialize raw payload → Save JSON to Drive       │
│       │                                                     │
│       ▼                                                     │
│  OpenAI GPT-4o  (LangChain Agent)                          │
│       Field extraction from unstructured questions[] array  │
│       Spelling correction on free-text fields               │
│       Schema normalization → clean structured JSON          │
│       │                                                     │
│       ▼                                                     │
│  Read Servicios Sheet → Generate SRV-XXX ID (sequential)   │
│  Read Clientes Sheet  → NIT lookup                         │
│       │                                                     │
│       ├══ Client exists? ══► Resolve CLI-XXX from sheet     │
│       └══ New client?   ══► Generate CLI-XXX → Append row  │
│                                                             │
│       ▼                                                     │
│  Build Final JSON (canonical schema)                        │
│       │                                                     │
│       ├══► Append row to Servicios Sheet                    │
│       ├══► Append row to Raw audit Sheet                    │
│       ├══► Search/Create PDF folder (YYYY-MM partitioning)  │
│       ├══► Generate HTML → Convert to PDF via REST API      │
│       ├══► Save PDF to Drive                                │
│       │                                                     │
│       ▼                                                     │
│  Send approval email + PDF to Manager (Outlook API)        │
│       │                                                     │
│       ▼                                                     │
│  Wait/Resume node (async gate, 7-day timeout)              │
│       │                                                     │
│       ├══ APPROVED ══► Download PDF → Send to client        │
│       └══ CORRECTION ══► Resume URL → Correction workflow   │
│                                                             │
│  Escalation: Day 3 reminder → Day 5 reminder → Day 7 close │
│  Error routing: 16 critical nodes with auto-notifications   │
└─────────────────────────────────────────────────────────────┘
       │
       ▼ (correction path)
┌─────────────────────────────────────────────────────────────┐
│                 CORRECTION WORKFLOW (n8n)                    │
│                                                             │
│  Webhook POST /correccion-servicio                          │
│       │                                                     │
│       ├══► Cancel main Wait via HTTP resume URL             │
│       ├══► Update Servicios Sheet (upsert by id_servicio)   │
│       ├══► Regenerate HTML → PDF                            │
│       ├══► Delete original PDF from Drive                   │
│       ├══► Upload corrected PDF (same folder, same name)    │
│       ├══► Send re-approval email to Manager                │
│       ▼                                                     │
│  Wait/Resume (72h timeout)                                  │
│       │                                                     │
│       └══ APPROVED ══► Download PDF → Send to client        │
└─────────────────────────────────────────────────────────────┘
       │
[correccion.html SPA] ════ URL params prefill ════► Editable form → POST corrections
```

---

## Data Schema

### `Servicios` sheet (append + upsert)
```
id_servicio | fecha | hora_inicio | hora_fin | tecnico | id_cliente
nombre_cliente | nit_cedula | telefono | correo_cliente | direccion
sucursal | contacto | equipos_intervenidos (JSON) | solicitud_servicio (JSON)
trabajos_realizados | repuestos_suministros | observaciones | correo_envio
```

### `Clientes` sheet (append on new client)
```
id_cliente | nombre | nit_cedula | telefono | correo | direccion | sucursal
```

### `Raw` sheet (audit log)
```
timestamp | id_servicio | datos_raw (full JSON snapshot)
```

### Google Drive structure
```
/CAO-Servicios/
  ├── RAW/
  │   ├── RAW 2026-04/
  │   │   └── 2026-04-HH-mm-ss.json
  │   └── RAW 2026-05/
  └── PDF/
      ├── PDF 2026-04/
      │   └── SRV-001-ClientName.pdf
      └── PDF 2026-05/
```

---

## Key Engineering Decisions

**GPT-4o as ETL transformation layer**

Raw Fillout form submissions arrive as unstructured `questions[]` arrays with inconsistent casing, spelling errors, and mixed date and time formats. Instead of brittle field-by-field parsing logic, a LangChain agent powered by GPT-4o receives the full raw JSON and returns a validated canonical schema every time, including proper capitalization of equipment names, date formatting to `DD/MM/YYYY`, time formatting to `HH:MM AM/PM`, and spelling correction across all free-text fields.

**Async approval gate with Wait/Resume**

The main workflow pauses execution after sending the approval email and resumes only when the manager clicks the approval link, which triggers a webhook resume call tied to that specific workflow instance. This achieves a stateful human-in-the-loop checkpoint without polling, cron jobs, or external state management. Automatic escalation notifications fire at Day 3 and Day 5 if no action is taken.

**CORS proxy via intermediary webhook**

n8n Wait/Resume URLs cannot be called directly from the browser due to cross-origin restrictions. The correction SPA routes its resume call through a dedicated n8n webhook (`/iniciar-correccion`) that acts as a server-side proxy, forwarding the resume signal to the paused main workflow instance while simultaneously redirecting the user to the correction form with the order data encoded in the URL.

**Sequential ID generation with sheet as source of truth**

`SRV-XXX` and `CLI-XXX` IDs are generated at runtime by reading the full Servicios and Clientes sheets, finding the current maximum, and incrementing. No external sequence generator or database required. Google Sheets is the single source of truth for both ID sequences.

**Stateless SPAs with URL parameter prefill**

Both frontend applications are fully stateless. All context including client data, order data, and resume URLs is passed via URL parameters. No backend session management, no cookies, and no server-side rendering. Both apps are deployed instantly on GitHub Pages with zero infrastructure cost.

**Document lifecycle management**

On correction, the original PDF is located in Drive by querying the `id_servicio` value, deleted, and replaced with the regenerated version under the same filename and folder path. Full record traceability is maintained through the Raw audit sheet, which stores a complete JSON snapshot of every record at write time.

---

## Integrations

| API | Auth | Operations |
|---|---|---|
| Google Sheets API | OAuth2 | read, append, upsert |
| Google Drive API | OAuth2 | create, upload, download, delete, search folders |
| OpenAI API (GPT-4o) | API Key | LangChain agent, structured JSON extraction |
| Microsoft Outlook API | OAuth2 | send email with PDF attachment, CC routing |
| html2pdfrocket API | API Key | HTML to PDF conversion |
| Fillout Forms | Webhook | form submission trigger |

---

## Error Handling

Every critical node in the pipeline has a dedicated error route. 16 Stop-and-Error nodes halt execution on unrecoverable failures. 16 Outlook notification nodes fire automatically on any node failure, each including the node name, error message, and timestamp. Error coverage spans the AI processor, all Sheets read and write operations, Drive operations, PDF generation, and email delivery.

---

## Frontend SPAs

### `clientes.html` — Client Lookup

Fetches the client database in real time via Google Sheets CSV public export. Implements fuzzy search by name or NIT with regex-highlighted matches. Handles multi-branch clients by auto-selecting when a single branch exists or rendering a selector when multiple branches are found. Builds the Fillout form URL with prefilled query parameters on open. Built with Vanilla JS (ES6), CSS custom properties, DM Sans and Bebas Neue.

### `correccion.html` — Order Correction

Receives the full service order as URL-encoded JSON via query parameters. Renders a fully editable form covering header fields, client data, a dynamic equipment table, service request rows, work performed, parts and observations. Tracks field-level changes with visual diff indicators. On submit, POSTs corrections to the n8n correction webhook and fires the resume URL to unblock the paused main workflow. Built with Vanilla JS (ES6), IBM Plex Sans and IBM Plex Mono.

---

## Stack

```
Orchestration   n8n (cloud, event-driven)
AI / ETL        OpenAI GPT-4o · LangChain Agent
Data Store      Google Sheets API (append / upsert / read)
Storage         Google Drive API (partitioned by YYYY-MM)
Notifications   Microsoft Outlook API
PDF             html2pdfrocket REST API
Frontend        HTML5 · CSS3 · JavaScript ES6
Hosting         GitHub Pages · Cloudinary CDN
Auth            OAuth2 (Google Workspace · Microsoft 365)
```

---

## Impact

| Metric | Result |
|---|---|
| Manual work eliminated | 8+ hours/week |
| Order loss since deployment | 0 |
| Independent async workflows | 3 |
| API integrations | 6 |
| Error-handled critical nodes | 16 |
| Cloud migration | Zero downtime · Zero data loss |

---

## Author

**Santiago Mesa Pachón** — Data Engineer & AI Automation

[linkedin.com/in/santiagomesapachon](https://www.linkedin.com/in/santiagomesapachon) · Bogotá, Colombia
