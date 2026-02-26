# AgenticAI
Agentic AI Assignments 1

# 🧩 Word Search Puzzle Agent

> **User-System Interaction Flow Documentation**
> Built on Langflow · Powered by Groq Vision · Llama 4 Maverick

---

## 📋 Document Info

| Field | Value |
|---|---|
| Document Type | Interaction Flow + Architecture |
| Version | 1.0 — 2025 |
| Platform | Langflow + Groq API |
| Scope | End-to-End Pipeline |

---

## 1. Purpose & Scope

This document describes the complete user-to-system interaction flow of the **Word Search Puzzle Agent** built in Langflow. It covers every component in the pipeline, the data that passes between them, and how a user query is transformed into an interactive HTML solution.

---

## 2. System Component Overview

| # | Component | Role | Technology | Output Type |
|---|---|---|---|---|
| 1 | **Chat Input** | Entry point — receives user message or image URL | Langflow built-in | Message |
| 2 | **Groq (Text LLM)** | Reads the chat input; routes to grid extraction | Groq API / llama-3.1-8b-instant | Model Response |
| 3 | **Image URL to Base64** | Fetches the puzzle image from URL and encodes it | Python custom component | Base64 string |
| 4 | **Groq Vision (Base64)** | Reads the puzzle image; extracts letter grid & word list | Groq API / llama-4-maverick (vision) | JSON response |
| 5 | **Word Grid Finder** | Solves the grid (8-direction search); renders HTML | Python custom component | HTML string |
| 6 | **Chat Output** | Displays the interactive HTML solution to the user | Langflow built-in | Rendered HTML |

---

## 3. Step-by-Step Interaction Flow

| Step | Component | Action | Input | Output |
|---|---|---|---|---|
| 1 | **Chat Input** | User types a message (e.g. image URL or "Find TURKEY"). The Playground accepts text and optional file attachments. | User message / image URL | Message object |
| 2 | **Groq (Text LLM)** | Receives the chat message via the Input port. The System Message instructs it to identify word-search-related commands. | Message from Chat Input | Model Response (text) |
| 3 | **Image URL to Base64** | Fetches the puzzle image from the CDN URL. Converts binary image data to a base64 string with content type. | `https://...` image URL | base64 string + MIME type |
| 4 | **Groq Vision (Base64)** | Sends the base64 image + text prompt to Groq's multimodal model. The model reads every letter in the grid row by row, returning a 2D JSON array and a word list. | Base64 image + Text Prompt | `{"grid":[...],"words":[...]}` |
| 5 | **Word Grid Finder** | Parses the JSON grid. Searches for each word in all 8 directions (horizontal, vertical, 4 diagonals). Builds a colored HTML table with toggle interactions. | Grid JSON + Word List | Full HTML page (string) |
| 6 | **Chat Output** | Renders the HTML string in the Langflow Playground. The user sees the solved puzzle with highlighted words, word chips, and progress stats. | HTML string | Interactive rendered page |

---

## 4. Data Flow Diagram

```
  USER
    │
    │  (types message or image URL)
    ▼
  ┌─────────────┐
  │  Chat Input │
  └──────┬──────┘
         │ Message object
         ▼
  ┌──────────────────────────────────────────────┐
  │           Groq — Text LLM                    │
  │   System Message: "Find words in the input"  │
  └───────────────────┬──────────────────────────┘
                      │
          ┌───────────┴────────────┐
          │                        │
          ▼                        ▼
  ┌────────────────────┐   ┌──────────────────┐
  │ Image URL to Base64│   │  Word Grid Finder │◄──────────┐
  └────────┬───────────┘   └──────────────────┘           │
           │ base64 string                                 │
           ▼                                               │
  ┌────────────────────────┐                               │
  │  Groq Vision (Base64)  │  (Llama 4 Maverick)          │
  │  Reads image grid      │                               │
  │  Returns JSON grid ────┼───────────────────────────────┘
  └────────────────────────┘
  
  ┌──────────────────┐
  │  Word Grid Finder│
  │  · Parses 2D grid│
  │  · Solves 8-dir  │
  │  · Renders HTML  │
  └────────┬─────────┘
           │ HTML string
           ▼
  ┌─────────────┐
  │ Chat Output │
  └──────┬──────┘
         │
         ▼
  USER sees highlighted word search solution
```

---

## 5. Component Configurations

### 5.1 Chat Input

| Parameter | Value |
|---|---|
| Sender | User |
| Accepts Files | Yes (image upload in Playground) |
| Session ID | Auto-generated per session |
| Output Port | Message |

---

### 5.2 Groq (Text LLM)

| Parameter | Value |
|---|---|
| Model | `llama-3.1-8b-instant` |
| Groq API Key | Configured via `GROQ_API_KEY` environment variable |
| System Message | `"Find the words in the input. Give..."` |
| Enable Tool Models | Off |
| Output Port | Model Response |
| Connected To | Word Grid Finder (Letter Grid input) |

> ⚠️ **Note:** Change model to `meta-llama/llama-4-scout-17b-16e-instruct` for vision capability.

---

### 5.3 Image URL to Base64

| Parameter | Value |
|---|---|
| Type | Custom Python Component |
| Image URL | `https://as2.ftcdn.net/v2/jpg/05/48/...` (Countries puzzle) |
| Output | Base64 Message (string) |
| Connected To | Groq Vision (Base64) — Base64 Image Data port |

---

### 5.4 Groq Vision (Base64)

| Parameter | Value |
|---|---|
| Type | Custom Python Component |
| Model | `meta-llama/llama-4-maverick-17b-128e-instruct` |
| Image Type | `image/jpeg` |
| Text Prompt | `"Read the letter from the message..."` (extracts grid as JSON) |
| Output Port | Response |
| Connected To | Word Grid Finder — Letter Grid (JSON) port |

---

### 5.5 Word Grid Finder

| Parameter | Value |
|---|---|
| Type | Custom Python Component |
| Input: Letter Grid (JSON) | 2D array of letters from Groq Vision response |
| Input: Word List | Receiving input (from Groq Text or default value) |
| Algorithm | 8-directional search (horizontal, vertical, 4 diagonals) |
| Output | Full interactive HTML page with highlighted grid + word chips |
| Output Port | HTML |
| Connected To | Chat Output |

---

## 6. User Interaction Scenarios

| Scenario | User Action | System Response |
|---|---|---|
| **Solve entire puzzle** | User sends: `"Solve all words"` (default image pre-loaded) | Groq Vision reads grid image. Word Grid Finder finds all 30 countries, returns HTML with all highlighted. |
| **Find specific words** | User types: `"Find Australia and Mexico"` | Component parses natural language, locates the two words, renders HTML showing only those two highlighted. |
| **New puzzle image** | User provides a different image URL in the Image URL to Base64 field | Vision model reads new puzzle, extracts new grid and word list automatically. |
| **Interactive toggle** | User clicks a word chip in the HTML output | JavaScript toggles the highlight for that word on/off in the grid. |
| **Show All / Reset** | User clicks "Show All" or "Reset" button in HTML | All words highlighted simultaneously, or all highlights cleared. |

---

## 📁 Custom Components

| File | Description |
|---|---|
| `image_url_to_base64.py` | Fetches image from URL, returns base64 string |
| `word_grid_finder_filled.py` | Solves 8-direction word search, renders interactive HTML |
| `grid_to_html.py` | Converts raw grid JSON to styled HTML puzzle |
| `text_grid_to_image_highlighter.py` | Overlays highlights on original puzzle image via canvas |
| `grid_to_word_list.py` | Checks which known words exist in a grid |

---

*Word Search Puzzle Agent · Groq Vision Pipeline · Version 1.0*
