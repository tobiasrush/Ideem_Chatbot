# Ideem Support Chatbot

AI-powered customer support chatbot for the Ideem website, built with n8n, Google Gemini, and Supabase.

## Overview

This chatbot helps customers with:
- Installation and setup questions
- Troubleshooting and error resolution
- Product usage guidance
- Screenshot and error log analysis

## Architecture

```
Website (Chat Widget) → n8n Cloud (Workflow) → Supabase (Vector DB + Memory)
                                    ↓
                             Google Drive (Knowledge Base)
                                    ↓
                             Google Gemini (AI)
```

## Project Structure

```
Ideem_Chatbot/
├── README.md                    # This file
├── Ideem_Chatbot_Plan.md        # Detailed implementation plan
├── chat-widget/                 # Frontend chat widget code
├── n8n-workflows/               # Exported n8n workflow JSON files
├── docs/                        # Additional documentation
└── scripts/                     # Utility scripts
```

## Quick Start

1. Read the full implementation plan: [Ideem_Chatbot_Plan.md](./Ideem_Chatbot_Plan.md)
2. Set up Supabase database (Phase 1)
3. Configure Google Drive folder (Phase 2)
4. Set up n8n credentials (Phase 3)
5. Create indexing workflow (Phase 4)
6. Create chat workflow (Phase 5)
7. Integrate widget on website (Phase 6)

## Technologies

- **n8n Cloud** - Workflow automation
- **Google Gemini** - LLM for chat and embeddings
- **Supabase** - PostgreSQL with pgvector for RAG
- **@n8n/chat** - Embeddable chat widget
- **Google Drive** - Knowledge base storage

## License

Private - Ideem internal use only
