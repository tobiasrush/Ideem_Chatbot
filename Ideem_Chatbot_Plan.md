# Ideem Support Chatbot - Implementation Plan

## Project Overview

An AI-powered support chatbot for the Ideem website that helps customers with installation, troubleshooting, and product usage. The chatbot uses:
- **n8n Cloud** for workflow orchestration
- **Google Drive** as the knowledge base source
- **Supabase** for vector storage and chat memory
- **Google Gemini** for embeddings and chat responses
- **@n8n/chat** widget embedded on the website

---

## Phase 0: Configuration Setup

> **ACTION REQUIRED:** Fill in these values before proceeding

### Supabase Configuration
```
SUPABASE_URL=https://____________.supabase.co
SUPABASE_ANON_KEY=eyJ____________
SUPABASE_SERVICE_KEY=eyJ____________  (for n8n server-side access)
```

**To get these values:**
1. Go to https://supabase.com/dashboard
2. Select your project (or create new one)
3. Go to Settings → API
4. Copy the Project URL and anon/service keys

### n8n Cloud Configuration
```
N8N_INSTANCE_URL=https://____________.app.n8n.cloud
N8N_WEBHOOK_BASE=https://____________.app.n8n.cloud/webhook
```

**To get these values:**
1. Log into https://app.n8n.cloud
2. Your instance URL is shown in the browser address bar
3. Webhook URLs are generated when you create a Chat Trigger node

### Google Drive Configuration
```
GOOGLE_DRIVE_FOLDER_ID=____________
```

**To get this value:**
1. Open Google Drive
2. Navigate to your knowledge base folder
3. Copy the folder ID from the URL: `drive.google.com/drive/folders/[FOLDER_ID]`

### Gemini API Configuration
```
GEMINI_API_KEY=____________
```

**To get this value:**
1. Go to https://makersuite.google.com/app/apikey
2. Create a new API key

---

## Phase 1: Supabase Database Setup

### Step 1.1: Enable pgvector Extension

In Supabase SQL Editor, run:

```sql
-- Enable the pgvector extension
CREATE EXTENSION IF NOT EXISTS vector;
```

### Step 1.2: Create Vector Store Table

```sql
-- Table for storing document embeddings
CREATE TABLE documents (
  id BIGSERIAL PRIMARY KEY,
  content TEXT NOT NULL,
  metadata JSONB DEFAULT '{}',
  embedding VECTOR(768)  -- Gemini text-embedding-004 dimension
);

-- Create index for fast similarity search
CREATE INDEX documents_embedding_idx ON documents
  USING ivfflat (embedding vector_cosine_ops)
  WITH (lists = 100);

-- Create index on metadata for filtering
CREATE INDEX documents_metadata_idx ON documents USING GIN (metadata);
```

### Step 1.3: Create Chat Memory Table

```sql
-- Table for storing conversation history
CREATE TABLE n8n_chat_histories (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  session_id TEXT NOT NULL,
  message JSONB NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Index for fast session lookups
CREATE INDEX n8n_chat_histories_session_idx ON n8n_chat_histories(session_id);

-- Index for time-based queries
CREATE INDEX n8n_chat_histories_created_idx ON n8n_chat_histories(created_at);
```

### Step 1.4: Create Helper Functions

```sql
-- Function to search documents by similarity
CREATE OR REPLACE FUNCTION match_documents(
  query_embedding VECTOR(768),
  match_threshold FLOAT DEFAULT 0.7,
  match_count INT DEFAULT 5
)
RETURNS TABLE (
  id BIGINT,
  content TEXT,
  metadata JSONB,
  similarity FLOAT
)
LANGUAGE plpgsql
AS $$
BEGIN
  RETURN QUERY
  SELECT
    documents.id,
    documents.content,
    documents.metadata,
    1 - (documents.embedding <=> query_embedding) AS similarity
  FROM documents
  WHERE 1 - (documents.embedding <=> query_embedding) > match_threshold
  ORDER BY documents.embedding <=> query_embedding
  LIMIT match_count;
END;
$$;
```

### Step 1.5: Verify Setup

```sql
-- Check extension is enabled
SELECT * FROM pg_extension WHERE extname = 'vector';

-- Check tables exist
SELECT table_name FROM information_schema.tables
WHERE table_schema = 'public'
AND table_name IN ('documents', 'n8n_chat_histories');
```

---

## Phase 2: Google Drive Setup

### Step 2.1: Create Folder Structure

In Google Drive, create:
```
Ideem_Knowledge_Base/
├── help-docs/           # General help documentation
│   ├── getting-started.md
│   ├── faq.md
│   └── ...
├── installation/        # Installation and setup guides
│   ├── quick-start.md
│   ├── system-requirements.md
│   └── ...
├── troubleshooting/     # Error resolution guides
│   ├── common-errors.md
│   ├── error-codes.md
│   └── ...
└── examples/            # Code examples and templates
    ├── basic-setup.js
    ├── advanced-config.json
    └── ...
```

### Step 2.2: Share Folder with Service Account

1. In Google Cloud Console, create a service account
2. Download the JSON key file
3. In Google Drive, share the folder with the service account email
4. Store the service account credentials in n8n

---

## Phase 3: n8n Credentials Setup

### Step 3.1: Google Drive OAuth Credential

1. In n8n, go to **Credentials** → **Add Credential**
2. Select **Google Drive OAuth2 API**
3. Follow the OAuth flow to authorize

**Alternative: Service Account**
1. Select **Google Service Account**
2. Upload the JSON key file
3. Ensure the shared folder is accessible

### Step 3.2: Supabase Credential

1. In n8n, go to **Credentials** → **Add Credential**
2. Select **Supabase API**
3. Enter:
   - Host: `https://[PROJECT_ID].supabase.co`
   - Service Role Key: (from Supabase dashboard)

### Step 3.3: Google Gemini Credential

1. In n8n, go to **Credentials** → **Add Credential**
2. Select **Google Gemini API** (or Google AI)
3. Enter your Gemini API key

---

## Phase 4: Document Indexing Workflow

### Workflow: `Ideem_Document_Indexer`

**Purpose:** Sync documents from Google Drive to Supabase vector store

### Workflow Nodes:

```
[Schedule Trigger] → [Google Drive: List Files] → [Loop Over Items]
    → [Google Drive: Download] → [Switch: File Type]
    → [Text Splitter] → [Embeddings: Gemini] → [Supabase: Insert]
```

### Node Configuration:

#### 1. Schedule Trigger
- **Mode:** Cron
- **Expression:** `0 2 * * *` (daily at 2 AM)
- Also add: Manual Trigger for on-demand runs

#### 2. Google Drive: List Files
- **Operation:** List
- **Folder ID:** `{{ $env.GOOGLE_DRIVE_FOLDER_ID }}`
- **Options:**
  - Include subfolders: Yes
  - Filter: `mimeType != 'application/vnd.google-apps.folder'`

#### 3. Switch: File Type
Route based on MIME type:
- **PDF:** `application/pdf`
- **Markdown:** `text/markdown`, `text/plain`
- **Code:** `text/javascript`, `application/json`, etc.
- **Google Docs:** `application/vnd.google-apps.document`

#### 4. Text Splitter (Recursive Character)
- **Chunk Size:** 1000
- **Chunk Overlap:** 200
- **Separators:** `["\n\n", "\n", " ", ""]`

#### 5. Embeddings: Google Gemini
- **Model:** `text-embedding-004`
- **Operation:** Embed Documents

#### 6. Supabase Vector Store
- **Operation:** Insert Documents
- **Table:** `documents`
- **Metadata Fields:**
  - `filename`: `{{ $json.name }}`
  - `filepath`: `{{ $json.path }}`
  - `filetype`: `{{ $json.mimeType }}`
  - `lastModified`: `{{ $json.modifiedTime }}`
  - `category`: (extracted from folder path)

### Incremental Updates Logic:

Add a **Code Node** before processing to check if document has changed:

```javascript
// Check if document needs re-indexing
const currentModified = new Date($input.first().json.modifiedTime);
const storedModified = $input.first().json.storedModifiedTime; // From Supabase lookup

if (storedModified && currentModified <= new Date(storedModified)) {
  return []; // Skip - already indexed
}

return $input.all();
```

---

## Phase 5: Chat Handler Workflow

### Workflow: `Ideem_Chat_Handler`

**Purpose:** Process chat messages and generate AI responses

### Workflow Nodes:

```
[Chat Trigger] → [Set: Extract Session] → [IF: Has Image?]
    ├─ Yes → [Gemini Vision] → [Set: Add Context] ─┐
    └─ No ─────────────────────────────────────────┴→ [AI Agent] → [Respond]
```

### Node Configuration:

#### 1. Chat Trigger
- **Allowed Origins (CORS):** `https://yourdomain.com` (your website)
- **Authentication:** None (public access)
- **Webhook Path:** `/webhook/ideem-chat`

**Full Webhook URL:** `https://[YOUR_INSTANCE].app.n8n.cloud/webhook/ideem-chat`

#### 2. AI Agent
- **Model:** Google Gemini 1.5 Pro (or 2.0 Flash)
- **System Prompt:** (see below)
- **Memory:** Supabase Chat Memory
  - Table: `n8n_chat_histories`
  - Session ID: `{{ $json.sessionId }}`
  - Context Window: 10

**Tools to Connect:**
1. **Vector Store Tool (Supabase)**
   - Name: `search_documentation`
   - Description: "Search the Ideem knowledge base for relevant documentation"
   - Top K: 4

#### 3. System Prompt

```
You are the Ideem Support Assistant, an AI helper for customers using Ideem products.

## YOUR KNOWLEDGE SOURCE
You have access to the official Ideem documentation through your search_documentation tool. ALWAYS search before answering technical questions.

## CAPABILITIES
- Answer questions about Ideem installation, configuration, and usage
- Analyze error messages, logs, and JSON configurations
- Interpret screenshots of error dialogs or UI issues
- Provide code examples from the documentation
- Guide users through troubleshooting steps

## RESPONSE GUIDELINES
1. **Search First:** Always use search_documentation before answering technical questions
2. **Cite Sources:** When you find relevant documentation, mention the source (e.g., "According to the installation guide...")
3. **Be Specific:** For errors, identify the exact error type and provide targeted solutions
4. **Format Code:** Use markdown code blocks with appropriate language tags
5. **Be Concise:** Provide complete but focused answers
6. **Acknowledge Limits:** If you cannot find an answer, clearly say so and suggest:
   - Checking the documentation directly
   - Contacting support at support@ideem.com

## FOR ERROR LOGS/JSON
When a user shares error logs or JSON:
1. Identify the error type/code
2. Search for that specific error in documentation
3. Explain what the error means
4. Provide step-by-step resolution

## FOR SCREENSHOTS
When a user shares a screenshot:
1. Describe what you see in the image
2. Identify any error messages or issues
3. Search documentation for relevant solutions
4. Provide guidance based on the visual context

## LIMITATIONS
- You cannot access live systems or make changes
- You only know what's in the documentation
- For account-specific issues, direct users to human support

## TONE
Be helpful, professional, and friendly. Use clear, simple language.
```

#### 4. Image Handling Branch

**IF Node Condition:**
```javascript
{{ $json.files && $json.files.length > 0 }}
```

**Gemini Vision Node:**
- **Model:** Gemini 1.5 Pro Vision
- **Prompt:** "Analyze this screenshot. Describe any error messages, UI elements, or issues visible. Extract any text content."

---

## Phase 6: Chat Widget Integration

### Basic HTML Integration

Add to your website's HTML:

```html
<!-- n8n Chat Widget -->
<link href="https://cdn.jsdelivr.net/npm/@n8n/chat/dist/style.css" rel="stylesheet" />
<script type="module">
  import { createChat } from 'https://cdn.jsdelivr.net/npm/@n8n/chat/dist/chat.bundle.es.js';

  createChat({
    webhookUrl: 'https://[YOUR_N8N_INSTANCE].app.n8n.cloud/webhook/ideem-chat',
    mode: 'window',
    showWelcomeScreen: true,
    initialMessages: [
      'Hi! I\'m the Ideem Support Assistant.',
      'I can help you with installation, troubleshooting, and using our products.',
      'Feel free to paste screenshots, error logs, or JSON configs!'
    ],
    i18n: {
      en: {
        title: 'Ideem Support',
        subtitle: 'AI-powered help',
        footer: '',
        getStarted: 'Start Chat',
        inputPlaceholder: 'Type your question...',
        closeButtonTooltip: 'Close chat'
      }
    },
    theme: {
      // Customize to match your brand
      // See: https://github.com/n8n-io/n8n/tree/master/packages/%40n8n/chat
    }
  });
</script>
```

### Custom Styling

Create `chat-widget/custom-styles.css`:

```css
/* Override n8n chat default styles */
:root {
  --chat--color-primary: #your-brand-color;
  --chat--color-primary-shade-50: #lighter-shade;
  --chat--color-primary-shade-100: #darker-shade;
  --chat--color-secondary: #secondary-color;
  --chat--color-white: #ffffff;
  --chat--color-light: #f5f5f5;
  --chat--color-medium: #cccccc;
  --chat--color-dark: #333333;
  --chat--spacing: 1rem;
  --chat--border-radius: 0.5rem;
  --chat--window-width: 400px;
  --chat--window-height: 600px;
}

/* Position the chat button */
.n8n-chat-button {
  bottom: 20px;
  right: 20px;
}
```

---

## Phase 7: Testing Checklist

### Document Indexing Tests

- [ ] Run indexing workflow manually
- [ ] Verify documents appear in Supabase `documents` table
- [ ] Check embeddings are generated (768 dimensions)
- [ ] Verify metadata is stored correctly
- [ ] Add new file to Drive → Re-run → Verify it's indexed
- [ ] Modify existing file → Re-run → Verify it's updated

### Chat Functionality Tests

- [ ] Send basic question → Get relevant response
- [ ] Ask about specific topic → Verify RAG retrieval
- [ ] Check response cites documentation source
- [ ] Test multi-turn conversation (context retention)
- [ ] Paste JSON error log → Verify parsing and analysis
- [ ] Upload screenshot → Verify image analysis
- [ ] Ask unknown question → Verify graceful "I don't know"

### Widget Integration Tests

- [ ] Widget loads on website
- [ ] Chat window opens/closes correctly
- [ ] Messages send and receive
- [ ] File upload works
- [ ] Session persists during page navigation
- [ ] CORS is configured correctly
- [ ] Mobile responsive

---

## Phase 8: Deployment Checklist

### Security

- [ ] CORS configured to allow only your domain
- [ ] API keys stored securely in n8n credentials
- [ ] Supabase RLS policies configured (if needed)
- [ ] Rate limiting considered for webhook

### Performance

- [ ] Test with expected document volume
- [ ] Monitor n8n execution times
- [ ] Monitor Supabase query performance
- [ ] Consider caching for frequent queries

### Monitoring

- [ ] Enable n8n execution logging
- [ ] Set up Supabase monitoring
- [ ] Create alerts for workflow failures
- [ ] Track chat usage metrics

---

## Troubleshooting Guide

### Common Issues

**1. Chat widget not connecting**
- Check CORS settings in Chat Trigger node
- Verify webhook URL is correct
- Ensure workflow is active

**2. No results from knowledge base**
- Verify documents are indexed (check Supabase)
- Test similarity search directly in Supabase
- Adjust similarity threshold if needed

**3. Slow responses**
- Consider using Gemini 2.0 Flash instead of Pro
- Reduce context window size
- Check n8n execution logs for bottlenecks

**4. Images not processing**
- Verify Gemini Vision model is configured
- Check file size limits
- Ensure base64 encoding is working

---

## Cost Estimation

| Component | Usage | Est. Monthly Cost |
|-----------|-------|------------------|
| n8n Cloud | Starter plan | ~$20-50 |
| Supabase | Free tier (500MB) | $0 |
| Gemini API | ~100K queries | ~$10-30 |
| **Total** | | **$30-80** |

---

## Next Steps After Implementation

1. **Monitor & Iterate:** Review chat logs to identify common questions and improve documentation
2. **Add Feedback:** Implement thumbs up/down for response quality
3. **Escalation Path:** Add option to connect with human support
4. **Analytics:** Track metrics (response time, satisfaction, common topics)
5. **Expand Knowledge:** Continuously add documentation based on user questions
