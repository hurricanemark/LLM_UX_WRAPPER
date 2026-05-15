# A complete, drop‑in scaffold you can unzip on Ubuntu, open in VS Code, `npm install`, and run. It’s a simple monorepo: Node backend wrapping Ollama + React frontend.

---

## Project structure

```text
local-llm-wrapper/
  package.json
  README.md
  .gitignore
  server/
    package.json
    tsconfig.json
    src/
      index.ts
      ollamaClient.ts
  client/
    package.json
    tsconfig.json
    vite.config.ts
    index.html
    src/
      main.tsx
      App.tsx
      api.ts
      components/
        Layout.tsx
        Sidebar.tsx
        ChatPane.tsx
        MessageBubble.tsx
        InputBar.tsx
        ModelSelector.tsx
      styles.css
```

---

## Root files

### `package.json`

```json
{
  "name": "local-llm-wrapper",
  "private": true,
  "scripts": {
    "dev": "concurrently \"npm run dev --prefix server\" \"npm run dev --prefix client\"",
    "dev:server": "npm run dev --prefix server",
    "dev:client": "npm run dev --prefix client",
    "build": "npm run build --prefix server && npm run build --prefix client",
    "start": "npm run start --prefix server"
  },
  "devDependencies": {
    "concurrently": "^9.0.0"
  }
}
```

### `.gitignore`

```gitignore
node_modules
dist
.env
```

### `README.md`

```md
# Local LLM Wrapper (Ollama)

## Prereqs

- Node 20+
- Ollama running locally on Ubuntu
- `ollama serve` listening on `http://localhost:11434`

## Install

```bash
npm install
npm install --prefix server
npm install --prefix client
```

## Run in dev

```bash
npm run dev
```

- Backend: http://localhost:3001
- Frontend: http://localhost:5173

## Build

```bash
npm run build
```
```

---

## Backend (server)

### `server/package.json`

```json
{
  "name": "local-llm-wrapper-server",
  "main": "dist/index.js",
  "scripts": {
    "dev": "ts-node-dev --respawn src/index.ts",
    "build": "tsc",
    "start": "node dist/index.js"
  },
  "dependencies": {
    "axios": "^1.7.0",
    "cors": "^2.8.5",
    "express": "^4.19.0"
  },
  "devDependencies": {
    "@types/express": "^4.17.21",
    "@types/node": "^22.0.0",
    "ts-node-dev": "^2.0.0",
    "typescript": "^5.5.0"
  }
}
```

### `server/tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "CommonJS",
    "rootDir": "src",
    "outDir": "dist",
    "esModuleInterop": true,
    "strict": true,
    "skipLibCheck": true
  }
}
```

### `server/src/ollamaClient.ts`

```ts
import axios from "axios";

const OLLAMA_URL = process.env.OLLAMA_URL || "http://localhost:11434";

export interface GenerateRequest {
  model: string;
  prompt: string;
  temperature?: number;
  stream?: boolean;
}

export async function generate(req: GenerateRequest): Promise<string> {
  const response = await axios.post(
    `${OLLAMA_URL}/api/generate`,
    {
      model: req.model,
      prompt: req.prompt,
      temperature: req.temperature ?? 0.7,
      stream: false
    },
    { timeout: 600000 }
  );

  // Ollama returns { response: "..." }
  return response.data.response ?? "";
}

export async function listModels(): Promise<string[]> {
  const response = await axios.get(`${OLLAMA_URL}/api/tags`);
  return (response.data.models || []).map((m: any) => m.name);
}
```

### `server/src/index.ts`

```ts
import express from "express";
import cors from "cors";
import { generate, listModels } from "./ollamaClient";

const app = express();
const PORT = process.env.PORT || 3001;

app.use(cors());
app.use(express.json());

app.get("/api/health", (_req, res) => {
  res.json({ status: "ok" });
});

app.get("/api/models", async (_req, res) => {
  try {
    const models = await listModels();
    res.json({ models });
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: "Failed to list models" });
  }
});

app.post("/api/generate", async (req, res) => {
  const { model, prompt, temperature } = req.body || {};
  if (!model || !prompt) {
    return res.status(400).json({ error: "model and prompt are required" });
  }

  try {
    const text = await generate({ model, prompt, temperature, stream: false });
    res.json({ text });
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: "Generation failed" });
  }
});

app.listen(PORT, () => {
  console.log(`Server listening on http://localhost:${PORT}`);
});
```

---

## Frontend (client)

### `client/package.json`

```json
{
  "name": "local-llm-wrapper-client",
  "private": true,
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "react": "^18.3.0",
    "react-dom": "^18.3.0"
  },
  "devDependencies": {
    "@types/react": "^18.3.0",
    "@types/react-dom": "^18.3.0",
    "@vitejs/plugin-react-swc": "^3.7.0",
    "typescript": "^5.5.0",
    "vite": "^5.3.0"
  }
}
```

### `client/tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "jsx": "react-jsx",
    "moduleResolution": "bundler",
    "strict": true,
    "skipLibCheck": true
  },
  "include": ["src"]
}
```

### `client/vite.config.ts`

```ts
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react-swc";

export default defineConfig({
  plugins: [react()],
  server: {
    port: 5173,
    proxy: {
      "/api": "http://localhost:3001"
    }
  }
});
```

### `client/index.html`

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <title>Local LLM Wrapper</title>
    <meta name="viewport" content="width=device-width, initial-scale=1" />
  </head>
  <body class="dark">
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

### `client/src/styles.css`

```css
:root {
  color-scheme: dark;
  --bg: #050816;
  --bg-alt: #0b1020;
  --accent: #4f46e5;
  --border: #1f2937;
  --text: #e5e7eb;
  --muted: #9ca3af;
}

* {
  box-sizing: border-box;
}

body {
  margin: 0;
  font-family: system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
  background: var(--bg);
  color: var(--text);
}

.app-root {
  display: flex;
  height: 100vh;
}

.sidebar {
  width: 260px;
  background: #020617;
  border-right: 1px solid var(--border);
  display: flex;
  flex-direction: column;
  padding: 16px;
}

.sidebar-header {
  font-weight: 600;
  margin-bottom: 24px;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.sidebar-nav {
  flex: 1;
}

.sidebar-item {
  padding: 8px 10px;
  border-radius: 6px;
  cursor: pointer;
  color: var(--muted);
  margin-bottom: 4px;
}

.sidebar-item.active {
  background: rgba(79, 70, 229, 0.15);
  color: var(--text);
}

.sidebar-footer {
  font-size: 12px;
  color: var(--muted);
}

.main-pane {
  flex: 1;
  display: flex;
  flex-direction: column;
  background: radial-gradient(circle at top, #111827 0, #020617 55%);
}

.chat-header {
  padding: 12px 16px;
  border-bottom: 1px solid var(--border);
  display: flex;
  gap: 12px;
  align-items: center;
}

.chat-body {
  flex: 1;
  overflow-y: auto;
  padding: 16px;
}

.chat-footer {
  padding: 12px 16px;
  border-top: 1px solid var(--border);
}

.message {
  max-width: 70%;
  padding: 10px 12px;
  border-radius: 10px;
  margin-bottom: 8px;
  white-space: pre-wrap;
  font-size: 14px;
}

.message.user {
  margin-left: auto;
  background: #1d4ed8;
}

.message.assistant {
  margin-right: auto;
  background: #111827;
  border: 1px solid #1f2937;
}

.input-container {
  display: flex;
  gap: 8px;
}

.input-textarea {
  flex: 1;
  resize: none;
  border-radius: 8px;
  border: 1px solid var(--border);
  padding: 8px;
  background: #020617;
  color: var(--text);
  min-height: 60px;
}

.button-primary {
  background: var(--accent);
  border: none;
  color: white;
  padding: 0 16px;
  border-radius: 8px;
  cursor: pointer;
}

.select {
  background: #020617;
  border-radius: 6px;
  border: 1px solid var(--border);
  color: var(--text);
  padding: 4px 8px;
  font-size: 14px;
}
```

### `client/src/api.ts`

```ts
export interface ChatMessage {
  role: "user" | "assistant";
  content: string;
}

export async function fetchModels(): Promise<string[]> {
  const res = await fetch("/api/models");
  if (!res.ok) throw new Error("Failed to fetch models");
  const data = await res.json();
  return data.models || [];
}

export async function generate(model: string, prompt: string): Promise<string> {
  const res = await fetch("/api/generate", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ model, prompt })
  });
  if (!res.ok) throw new Error("Generation failed");
  const data = await res.json();
  return data.text || "";
}
```

### `client/src/components/Layout.tsx`

```tsx
import React from "react";
import { Sidebar } from "./Sidebar";
import { ChatPane } from "./ChatPane";

export const Layout: React.FC = () => {
  return (
    <div className="app-root">
      <Sidebar />
      <ChatPane />
    </div>
  );
};
```

### `client/src/components/Sidebar.tsx`

```tsx
import React from "react";

export const Sidebar: React.FC = () => {
  return (
    <aside className="sidebar">
      <div className="sidebar-header">
        <span>Local LLM</span>
        <span style={{ fontSize: 10, color: "#22c55e" }}>● online</span>
      </div>
      <div className="sidebar-nav">
        <div className="sidebar-item active">Chat</div>
        <div className="sidebar-item">Models</div>
        <div className="sidebar-item">Prompts</div>
        <div className="sidebar-item">Settings</div>
      </div>
      <div className="sidebar-footer">
        <div>Mode: Local only</div>
        <div>v0.1.0</div>
      </div>
    </aside>
  );
};
```

### `client/src/components/ModelSelector.tsx`

```tsx
import React, { useEffect, useState } from "react";
import { fetchModels } from "../api";

interface Props {
  value: string;
  onChange: (model: string) => void;
}

export const ModelSelector: React.FC<Props> = ({ value, onChange }) => {
  const [models, setModels] = useState<string[]>([]);
  const [loading, setLoading] = useState(false);

  useEffect(() => {
    setLoading(true);
    fetchModels()
      .then(setModels)
      .catch(console.error)
      .finally(() => setLoading(false));
  }, []);

  return (
    <select
      className="select"
      value={value}
      onChange={(e) => onChange(e.target.value)}
    >
      {loading && <option>Loading...</option>}
      {!loading && models.length === 0 && <option>No models</option>}
      {!loading &&
        models.map((m) => (
          <option key={m} value={m}>
            {m}
          </option>
        ))}
    </select>
  );
};
```

### `client/src/components/MessageBubble.tsx`

```tsx
import React from "react";
import { ChatMessage } from "../api";

export const MessageBubble: React.FC<{ message: ChatMessage }> = ({
  message
}) => {
  const cls =
    "message " + (message.role === "user" ? "user" : "assistant");
  return <div className={cls}>{message.content}</div>;
};
```

### `client/src/components/InputBar.tsx`

```tsx
import React, { useState } from "react";

interface Props {
  onSend: (text: string) => void;
  disabled?: boolean;
}

export const InputBar: React.FC<Props> = ({ onSend, disabled }) => {
  const [value, setValue] = useState("");

  const handleSend = () => {
    const trimmed = value.trim();
    if (!trimmed) return;
    onSend(trimmed);
    setValue("");
  };

  return (
    <div className="input-container">
      <textarea
        className="input-textarea"
        placeholder="Type a prompt and press Enter..."
        value={value}
        disabled={disabled}
        onChange={(e) => setValue(e.target.value)}
        onKeyDown={(e) => {
          if (e.key === "Enter" && !e.shiftKey) {
            e.preventDefault();
            handleSend();
          }
        }}
      />
      <button
        className="button-primary"
        onClick={handleSend}
        disabled={disabled}
      >
        Send
      </button>
    </div>
  );
};
```

### `client/src/components/ChatPane.tsx`

```tsx
import React, { useState } from "react";
import { ChatMessage, generate } from "../api";
import { MessageBubble } from "./MessageBubble";
import { InputBar } from "./InputBar";
import { ModelSelector } from "./ModelSelector";

export const ChatPane: React.FC = () => {
  const [messages, setMessages] = useState<ChatMessage[]>([]);
  const [model, setModel] = useState<string>("llama3");
  const [loading, setLoading] = useState(false);

  const handleSend = async (content: string) => {
    const userMsg: ChatMessage = { role: "user", content };
    setMessages((prev) => [...prev, userMsg]);
    setLoading(true);
    try {
      const reply = await generate(model, content);
      const assistantMsg: ChatMessage = {
        role: "assistant",
        content: reply
      };
      setMessages((prev) => [...prev, assistantMsg]);
    } catch (err) {
      console.error(err);
      setMessages((prev) => [
        ...prev,
        { role: "assistant", content: "Error: generation failed." }
      ]);
    } finally {
      setLoading(false);
    }
  };

  return (
    <main className="main-pane">
      <div className="chat-header">
        <span style={{ fontWeight: 500 }}>Chat</span>
        <ModelSelector value={model} onChange={setModel} />
      </div>
      <div className="chat-body">
        {messages.map((m, i) => (
          <MessageBubble key={i} message={m} />
        ))}
        {loading && (
          <div className="message assistant">Thinking…</div>
        )}
      </div>
      <div className="chat-footer">
        <InputBar onSend={handleSend} disabled={loading} />
      </div>
    </main>
  );
};
```

### `client/src/App.tsx`

```tsx
import React from "react";
import { Layout } from "./components/Layout";
import "./styles.css";

export const App: React.FC = () => {
  return <Layout />;
};
```

### `client/src/main.tsx`

```tsx
import React from "react";
import ReactDOM from "react-dom/client";
import { App } from "./App";

ReactDOM.createRoot(document.getElementById("root") as HTMLElement).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);
```

---

## How to run on Ubuntu

```bash
# from project root
npm install
npm install --prefix server
npm install --prefix client

# ensure Ollama is running:
# ollama serve

npm run dev
```

Then open `http://localhost:5173` in your browser—vertical navbar on the left, interactive prompt I/O on the right, all local to your Ubuntu box.

If you want next: we can add streaming, system prompt editor, or a Models page wired to `/api/models`.
