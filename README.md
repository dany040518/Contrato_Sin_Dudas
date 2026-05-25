# Contrato Sin Dudas

> Sistema inteligente para analizar contratos del sector agrícola. Detecta ambigüedades, inconsistencias y cláusulas confusas usando IA generativa.

https://contrato-sin-dudas.vercel.app/

![n8n](https://img.shields.io/badge/n8n-workflow-orange?logo=n8n)
![Supabase](https://img.shields.io/badge/Supabase-pgvector-3ECF8E?logo=supabase)
![OpenRouter](https://img.shields.io/badge/OpenRouter-LLM-blue)
![Vercel](https://img.shields.io/badge/Vercel-frontend-black?logo=vercel)
![Railway](https://img.shields.io/badge/Railway-n8n-purple?logo=railway)

---

## ¿Qué hace?

1. **Sube un contrato** en PDF — el sistema extrae el texto, identifica partes, fechas y cláusulas clave con GPT-4o-mini
2. **Pregunta en lenguaje natural** — el sistema busca los fragmentos más relevantes con búsqueda vectorial y responde usando un LLM
3. **Alertas automáticas** — notificaciones por email cuando un contrato está próximo a vencer o iniciar

---

## Arquitectura

```
Frontend (Vercel)
    │  POST /subir-contrato      POST /consultar
    ▼                                ▼
n8n (Railway)──────────────────────────────────
    │                                │
    │  Ingesta de Documentos         │  RAG Final
    │  ├─ Extrae texto (PDF)         │  ├─ Embedding pregunta
    │  ├─ GPT-4o-mini → metadatos   │  ├─ Búsqueda vectorial
    │  ├─ Chunking + Embeddings      │  └─ llama-3-8b → respuesta
    │  └─ Guarda en Supabase         │
    ▼                                ▼
Supabase (PostgreSQL + pgvector + Storage)
    ├─ contracts          (metadatos del contrato)
    ├─ contract_files     (referencia al PDF en storage)
    ├─ contract_texts     (texto completo extraído)
    └─ contract_chunks    (fragmentos + vectores 1536d + entidad/tipo)
```

---

## Stack

| Capa | Tecnología | Rol |
|---|---|---|
| Frontend | HTML + Vanilla JS | UI de carga y consulta |
| Hosting frontend | **Vercel** | Deploy automático desde GitHub |
| Orquestación | **n8n** en Railway | Núcleo de todos los workflows |
| Base de datos | **Supabase** | PostgreSQL + pgvector + Storage |
| LLM / Embeddings | **OpenRouter** | GPT-4o-mini, text-embedding-3-small, llama-3-8b |
| Alertas | Google Gemini + Gmail | Detección de vencimientos |

---

## Estructura del repositorio

```
/
├── docs/
│   └── GUIA_USUARIO_DUMMIES.md   ← guía paso a paso para usuarios finales
├── workflows/
│   ├── Ingesta de Documentos.json
│   ├── RAG Final.json
│   ├── Embeddings OpenRouter.json
│   └── Reminder.json
├── supabase/                     ← config Supabase CLI (local dev)
├── index.html                    ← frontend (debe estar en raíz para Vercel)
├── vercel.json
├── .env.example
└── README.md
```

---

## Workflows n8n

| Archivo | Disparador | Función |
|---|---|---|
| `workflows/Ingesta de Documentos.json` | Webhook POST | Recibe PDF → extrae → guarda → vectoriza |
| `workflows/RAG Final.json` | Webhook POST | Pregunta → búsqueda vectorial → respuesta LLM |
| `workflows/Embeddings OpenRouter.json` | Manual | Backfill de embeddings para chunks existentes |
| `workflows/Reminder.json` | Schedule 8am | Alerta por contratos que vencen en 14 días |

---

## Setup

### Requisitos
- n8n (local o Railway)
- Supabase (cuenta gratuita)
- OpenRouter API key
- Google AI Studio API key (para Reminder)
- Gmail OAuth2 (para Reminder)

### 1. Variables de entorno

```bash
cp .env.example .env
# Edita .env con tus claves reales
```

### 2. Supabase — crear tablas

Ejecutar en **Supabase Dashboard → SQL Editor**:

```sql
-- Extensión vectorial
create extension if not exists vector;

-- Contratos
create table contracts (
  id bigint generated always as identity primary key,
  entidad text,
  proveedor text,
  tipo text,
  valor numeric,
  fecha_inicio date,
  fecha_fin date,
  created_at timestamptz default now()
);

-- Archivos PDF
create table contract_files (
  id bigint generated always as identity primary key,
  contract_id bigint references contracts(id),
  file_name text,
  file_url text,
  created_at timestamptz default now()
);

-- Texto completo
create table contract_texts (
  id bigint generated always as identity primary key,
  contract_id bigint references contracts(id),
  full_text text,
  created_at timestamptz default now()
);

-- Chunks con vectores
create table contract_chunks (
  id bigint generated always as identity primary key,
  contract_id text,
  chunk text,
  embeddings vector(1536),
  entidad text,
  tipo text,
  created_at timestamptz default now()
);

-- Función de búsqueda semántica
create or replace function match_contract_chunks(
  query_embedding vector(1536),
  match_count int,
  filter_contract_id text default null
)
returns table (
  id bigint,
  contract_id text,
  chunk text,
  entidad text,
  tipo text,
  similarity float
)
language sql stable as $$
  select id, contract_id, chunk, entidad, tipo,
    1 - (embeddings <=> query_embedding) as similarity
  from contract_chunks
  where
    (filter_contract_id is null or contract_id = filter_contract_id)
    and embeddings is not null
  order by embeddings <=> query_embedding
  limit match_count;
$$;
```

### 3. Supabase Storage

Crear bucket llamado `contracts` en **Storage → New bucket**.

### 4. Credenciales en n8n

Importar los 4 archivos de `workflows/` en n8n y crear las siguientes credenciales en **Settings → Credentials**:

| Nombre exacto | Tipo | Dónde obtenerla |
|---|---|---|
| `Supabase account` | Supabase API | Dashboard → Settings → API |
| `Supabase account 2` | Supabase API | Mismos datos |
| `Google Gemini(PaLM) Api account` | Google PaLM API | [aistudio.google.com](https://aistudio.google.com/app/apikey) |
| `Gmail OAuth2 API` | Gmail OAuth2 | Google Cloud Console |

En los nodos **HTTP Request** de OpenRouter, reemplazar `YOUR_OPENROUTER_API_KEY` con tu clave de [openrouter.ai/keys](https://openrouter.ai/keys).

En los nodos de **Upload to bucket**, reemplazar `YOUR_SUPABASE_STORAGE_KEY` con la `service_role` key de Supabase (Dashboard → Settings → API).

---

## Despliegue

### n8n → Railway

```bash
# En Railway: New Project → Deploy from template → n8n
# Variables de entorno obligatorias:
WEBHOOK_URL=https://[tu-proyecto].railway.app
N8N_ENCRYPTION_KEY=[clave-aleatoria-32-chars]
```

Importar los workflows desde `workflows/*.json` en la UI de n8n.

### Frontend → Vercel

El frontend se despliega automáticamente desde GitHub (rama `main`) cada vez que hay un push.

Para el primer deploy o para forzar uno manual:

```bash
# Opción 1: importar desde GitHub en vercel.com
# Opción 2: CLI
npm i -g vercel
vercel login
vercel --prod
```

Actualizar las URLs en `index.html` antes del deploy:
```js
const WEBHOOK_UPLOAD = "https://[tu-n8n].railway.app/webhook/subir-contrato";
const WEBHOOK_QUERY  = "https://[tu-n8n].railway.app/webhook/consultar";
```

---

## Seguridad

- Las API keys viven **solo en n8n** — nunca en el frontend
- `.env` está en `.gitignore`
- Los workflows en este repo usan placeholders: `YOUR_OPENROUTER_API_KEY`, `YOUR_SUPABASE_STORAGE_KEY`, `YOUR_SUPABASE_SERVICE_ROLE_KEY`
- La `service_role` key de Supabase solo se usa server-side (n8n)

---

## Documentación para usuarios

Ver [docs/GUIA_USUARIO_DUMMIES.md](docs/GUIA_USUARIO_DUMMIES.md) — guía paso a paso en español para usuarios sin conocimientos técnicos.
