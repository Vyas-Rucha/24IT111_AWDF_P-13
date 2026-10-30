# Practical 13: AI API Integration into a Web Application
**Course:** Advanced Web Development Frameworks (ITUE301)  
**Student ID:** 24IT111  
**Course Outcomes / Program Outcomes:** CO6 / PO3, PO5, PO12  

---

## 📌 Objective
To securely integrate an external AI service (Google Gemini API & OpenAI API) into an existing full-stack Task Management application (React + Node.js / Express), handle third-party service failures gracefully via **Graceful Degradation**, and adhere to **Responsible AI** governance practices.

---

## 🏛️ System Architecture & Workflow

```text
+-----------------------------------------------------------------------------------+
|                              REACT FRONTEND (Vite)                                 |
|                                                                                   |
|  [ Task Title Input ] ---> Click "✨ Generate with AI"                              |
|           |                                                                       |
|           | (Shows loading spinner + shimmer animation)                           |
+-----------|-----------------------------------------------------------------------+
            |
            | POST /api/ai/generate-description { title: "..." }
            | (Internal API call: NO API KEY EXPOSED IN BROWSER)
            v
+-----------------------------------------------------------------------------------+
|                             EXPRESS BACKEND (Node.js)                              |
|                                                                                   |
|  1. Express Rate Limiter: Protects against quota exhaustion                       |
|  2. Read API Key from server-side .env (OPENAI_API_KEY / GEMINI_API_KEY)          |
|  3. Try / Catch Execution Block                                                   |
+-----------|-----------------------------------------------------------------------+
            |
            +------------> [ External AI Service: Gemini / OpenAI ]
            |                        |
            |                        | (HTTP 200 / Success)
            |                        v
            |             Parse JSON Output: { description, priority, suggestedDeadline }
            |
            | (On Error / Timeout / Invalid Key / Network Failure)
            +------------> [ Graceful Degradation Fallback Handler ]
                                     |
                                     v
                          Generate Intelligent Local Fallback Template
                                     |
                                     v
+-----------------------------------------------------------------------------------+
|                        RESPONSE RETURNED TO REACT UI                              |
|                                                                                   |
|  - Populates editable textarea (User maintains final editorial control)            |
|  - Displays source banner (Live AI vs Graceful Fallback)                          |
|  - Renders Responsible AI Disclaimer: "AI-generated suggestion. Please review."   |
+-----------------------------------------------------------------------------------+
```

---

## 🛡️ Key Analysis & Viva Questions

### 1. Why must the AI API key be called from the backend rather than directly from the React frontend?
* **Exposure Vulnerability:** Any code or variable present in client-side React code (including `.env` variables prefixed with `VITE_` or `REACT_APP_`) is compiled directly into the public JavaScript bundle downloaded to the client's browser.
* **Network Interception:** If the browser calls OpenAI or Gemini directly, anyone opening the browser **DevTools > Network Tab** will see the raw `Authorization: Bearer <API_KEY>` or URL query string with the secret key in plaintext.
* **Security & Financial Risk:** Malicious actors can extract the key, burn your API quotas, trigger astronomical billing, or access sensitive models. 
* **The Solution:** By maintaining the API key exclusively in the server-side `.env` file, the Express backend serves as a secure proxy. The client only interacts with your backend endpoint (`/api/ai/generate-description`).

### 2. What does 'graceful degradation' mean in this context, and why does it matter?
* **Definition:** *Graceful degradation* is an architectural design philosophy where a system continues to fulfill its core mission without crashing or displaying fatal errors when external, non-critical sub-components or dependencies fail.
* **Relevance:** Third-party AI services are outside our direct operational control—they can experience outages, latency spikes, rate-limit 429 errors, or invalid key rejections.
* **Implementation:** In `server/services/aiService.js`, the AI request is wrapped in an asynchronous `try/catch` block. If the external API fails or the key is absent:
  1. The server catches the error cleanly without crashing.
  2. It generates an intelligent, contextual fallback description using deterministic rule templates.
  3. It returns `{ description: "...", fallback: true, message: "..." }` with status 200.
  4. The user can still add, edit, and organize tasks seamlessly.

### 3. What are the privacy and governance implications of sending task data to an external AI API, and what kind of data should never be sent this way?
* **Data Transmission:** When task data is dispatched to external LLMs, it travels over the public internet to third-party data centers, where it may be logged, monitored, or ingested for model fine-tuning unless enterprise Zero Data Retention (ZDR) agreements exist.
* **Data that must NEVER be sent:**
  1. **Personally Identifiable Information (PII):** Employee/student names, government IDs, phone numbers, home addresses, personal email IDs.
  2. **Security Credentials:** API keys, database credentials, passwords, cryptographic private keys, internal IP addresses.
  3. **Confidential Business Data:** Proprietary source code, financial spreadsheets, trade secrets, unreleased patent designs.
  4. **Regulated Healthcare / Financial Data:** Patient health records (HIPAA violation) or credit card details (PCI-DSS violation).

---

## 🚀 Setup & Execution Guide

### Prerequisites
* Node.js v18+ and npm installed.

### 1. Clone & Install Dependencies
```bash
# Navigate to project folder
cd "d:/AWDF PRACS/Prac-13_24IT111"

# Install backend dependencies
cd server
npm install

# Install frontend dependencies
cd ../client
npm install
```

### 2. Configure Environment Variables
Inside the `server/` directory, open `.env`:
```env
PORT=5000

# Provide at least one key, or leave blank to test Graceful Fallback mode!
OPENAI_API_KEY=your_openai_api_key_here
GEMINI_API_KEY=your_gemini_api_key_here

# Provider selection: 'auto', 'gemini', or 'openai'
AI_PROVIDER=auto
```
> **Security Reminder:** Notice that `.env` is listed in `.gitignore` and is **never** committed to version control.

### 3. Start Backend & Frontend
Open two terminal windows:

**Terminal 1 (Backend Server):**
```bash
cd server
npm run dev
# Server runs on http://localhost:5000
```

**Terminal 2 (React Frontend):**
```bash
cd client
npm run dev
# Frontend runs on http://localhost:3000
```

---

## 🧪 Testing Graceful Failure Live (Lab Requirement)
To verify failure handling live during evaluation:
1. In `server/.env`, enter an invalid key (e.g. `GEMINI_API_KEY=invalid_dummy_key_12345`).
2. In the React web UI, type a title like `"Debug Production Memory Leak"` and click **Generate with AI**.
3. **Observation:**
   - The server catches the 401/403 authorization error without crashing.
   - The UI immediately populates an intelligent fallback description.
   - A clear status banner notifies: *"Graceful fallback: Local template applied because external AI API was unreachable or unconfigured."*
   - The core Task Management application remains fully operational.

---

## 📡 API Endpoints

| Method | Endpoint | Description |
| :--- | :--- | :--- |
| `GET` | `/api/health` | Backend health check |
| `GET` | `/api/ai/status` | Current AI provider and security diagnostics |
| `POST` | `/api/ai/generate-description` | Generates AI task description, priority, and deadline |
| `GET` | `/api/tasks` | Retrieves all tasks |
| `POST` | `/api/tasks` | Creates a new task |
| `PUT` | `/api/tasks/:id` | Updates an existing task |
| `PATCH` | `/api/tasks/:id/toggle` | Toggles completion status |
| `DELETE` | `/api/tasks/:id` | Deletes a task |
#   2 4 I T 1 1 1 _ A W D F _ P - 1 3  
 