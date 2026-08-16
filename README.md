# ai-chat-bot-groq

A Groq-powered AI chat widget for a shopping site — a rate-limited `POST /api/chat` endpoint on an Express backend, plus a floating chat widget on a Next.js (shadcn/ui) frontend. The assistant answers questions about products, orders, shipping, and returns.

## Features

- Groq chat completions via plain `fetch` — no extra npm packages required (Node 18+ has built-in `fetch`)
- Dedicated rate limiter (`chatLimiter`) so chat spam never affects your checkout or global limiters
- Input validation matching your existing route style (`messages` array, role/content checks)
- Floating chat widget with open/close toggle, auto-scroll, loading state, and error fallbacks

## Tech Stack

- **Backend:** Express + `express-rate-limit`
- **Frontend:** Next.js (App Router) + shadcn/ui + lucide-react
- **AI:** Groq API (`llama-3.3-70b-versatile`)

## Prerequisites

- Node.js 18+
- A Groq API key from <https://console.groq.com>
- An existing Express backend and Next.js project (this guide adds the feature to them)

## Backend Setup

### 1. Add your API key

Create/edit `.env` (and `.env.example` with a placeholder) in the backend:

```env
GROQ_API_KEY=your_groq_api_key_here
```

Put the real key only in `.env` — keep the placeholder in `.env.example`.

### 2. Add a dedicated rate limiter

Right below your existing `checkoutLimiter`, add:

```typescript
const chatLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  limit: 20,
  message: {
    message: "Too many requests. Please try again in a few minutes.",
  },
  standardHeaders: true,
  legacyHeaders: false,
});
```

### 3. Add the chat route

<!-- Place this after the `/api/reviews/latest` route (at the end of the Reviews section): -->

```typescript
app.post(
  "/api/chat",
  chatLimiter,
  async (req: Request, res: Response, next: NextFunction) => {
    try {
      const GROQ_API_KEY = process.env.GROQ_API_KEY;
      if (!GROQ_API_KEY) {
        return res.status(503).json({ message: "Chat is not configured" });
      }

      const requestBody = req.body ?? {};
      const rawMessages = requestBody.messages;
      if (!Array.isArray(rawMessages) || rawMessages.length === 0) {
        return res.status(400).json({ message: "messages is required" });
      }
      if (rawMessages.length > 30) {
        return res.status(400).json({ message: "Too many messages" });
      }

      const messages: { role: string; content: string }[] = [];
      for (const m of rawMessages) {
        if (typeof m !== "object" || m === null) {
          return res.status(400).json({ message: "Invalid message" });
        }
        if (m.role !== "user" && m.role !== "assistant") {
          return res.status(400).json({ message: "Invalid message role" });
        }
        if (typeof m.content !== "string" || m.content.length > 2000) {
          return res.status(400).json({ message: "Invalid message content" });
        }
        messages.push({ role: m.role, content: m.content });
      }

      const groqRes = await fetch("https://api.groq.com/openai/v1/chat/completions", {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          Authorization: `Bearer ${GROQ_API_KEY}`,
        },
        body: JSON.stringify({
          model: "llama-3.3-70b-versatile",
          messages: [
            {
              role: "system",
              content:
                "You are a helpful shopping assistant for an online store. Answer questions about products, orders, shipping, and returns concisely and politely.",
            },
            ...messages,
          ],
          temperature: 0.5,
        }),
      });

      if (!groqRes.ok) {
        const errText = await groqRes.text();
        console.error("Groq error:", errText);
        return res.status(502).json({ message: "Chat service unavailable" });
      }

      const data = await groqRes.json();
      const reply = data?.choices?.[0]?.message?.content ?? "";
      res.json({ reply });
    } catch (err) {
      next(err);
    }
  },
);
```

### Why this approach

- **Separate `chatLimiter`** — chat spam won't affect your checkout or global limiters
- **Direct `fetch` instead of the Groq SDK** — no new dependencies, and `fetch` is built into Node 18+
- **Validation mirrors your checkout route** — keeps the codebase style consistent

### 4. Test the endpoint

Run this in a PowerShell terminal (backend must be running):

```powershell
Invoke-RestMethod -Uri "http://localhost:5000/api/chat" -Method Post -ContentType "application/json" -Body '{"messages":[{"role":"user","content":"hello"}]}'
```

## Frontend Setup

### 1. Install required shadcn components

Run in the frontend project terminal:

```bash
npx shadcn@latest add button input scroll-area card
```

### 2. Add the backend URL

Create/edit `.env.local` in the frontend project:

```env
NEXT_PUBLIC_API_URL=http://localhost:5000
```

This is your Express backend address — after deploying, replace it with the real backend URL.

### 3. Create the chat widget component

Create a new file `components/chat-widget.tsx`:

```tsx
"use client";

import { useState, useRef, useEffect } from "react";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { MessageCircle, X, Send } from "lucide-react";

type ChatMessage = {
  role: "user" | "assistant";
  content: string;
};

export default function ChatWidget() {
  const [open, setOpen] = useState(false);
  const [messages, setMessages] = useState<ChatMessage[]>([]);
  const [input, setInput] = useState("");
  const [loading, setLoading] = useState(false);
  const scrollRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    scrollRef.current?.scrollTo({ top: scrollRef.current.scrollHeight });
  }, [messages, open]);

  async function sendMessage() {
    const text = input.trim();
    if (!text || loading) return;

    const nextMessages: ChatMessage[] = [...messages, { role: "user", content: text }];
    setMessages(nextMessages);
    setInput("");
    setLoading(true);

    try {
      const res = await fetch(`${process.env.NEXT_PUBLIC_API_URL}/api/chat`, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ messages: nextMessages }),
      });
      const data = await res.json();
      if (!res.ok) {
        setMessages((prev) => [
          ...prev,
          { role: "assistant", content: "Sorry, I am unable to reply at the moment." },
        ]);
        return;
      }
      setMessages((prev) => [...prev, { role: "assistant", content: data.reply }]);
    } catch {
      setMessages((prev) => [
        ...prev,
        { role: "assistant", content: "A network issue occurred, please try again." },
      ]);
    } finally {
      setLoading(false);
    }
  }

  return (
    <div className="fixed bottom-6 right-6 z-50 flex flex-col items-end">
      {open && (
        <div className="mb-3 flex h-[420px] w-[320px] flex-col overflow-hidden rounded-xl border bg-background shadow-xl">
          <div className="flex items-center justify-between border-b px-4 py-3">
            <span className="font-medium">Shop Assistant</span>
            <button onClick={() => setOpen(false)} aria-label="Close chat">
              <X className="h-4 w-4" />
            </button>
          </div>

          <div ref={scrollRef} className="flex-1 space-y-3 overflow-y-auto px-4 py-3">
            {messages.length === 0 && (
              <p className="text-sm text-muted-foreground">
                Ask something — I can help with products, orders, and shipping.
              </p>
            )}
            {messages.map((m, i) => (
              <div
                key={i}
                className={`max-w-[85%] rounded-lg px-3 py-2 text-sm ${
                  m.role === "user"
                    ? "ml-auto bg-primary text-primary-foreground"
                    : "bg-muted"
                }`}
              >
                {m.content}
              </div>
            ))}
            {loading && (
              <div className="max-w-[85%] rounded-lg bg-muted px-3 py-2 text-sm text-muted-foreground">
                লিখছে...
              </div>
            )}
          </div>

          <div className="flex items-center gap-2 border-t p-2">
            <Input
              value={input}
              onChange={(e) => setInput(e.target.value)}
              onKeyDown={(e) => e.key === "Enter" && sendMessage()}
              placeholder="Message..."
              disabled={loading}
            />
            <Button size="icon" onClick={sendMessage} disabled={loading}>
              <Send className="h-4 w-4" />
            </Button>
          </div>
        </div>
      )}

      <Button
        size="icon"
        className="h-14 w-14 rounded-full bg-orange-500 shadow-lg hover:bg-orange-600"
        onClick={() => setOpen((v) => !v)}
      >
        <MessageCircle className="h-6 w-6" />
      </Button>
    </div>
  );
}
```

### 4. Add the widget to the root layout

Open `app/layout.tsx`, add the import and render `<ChatWidget />` inside `<body>`, after `children`:

```tsx
import ChatWidget from "@/components/chat-widget";

// ...inside <body>, after children:
<ChatWidget />
```

## Run & Verify

1. Start the backend: `npm run dev` (Express)
2. Start the frontend: `npm run dev` (Next.js)
3. Open the site in the browser — the floating chat button should appear in the bottom-right corner
4. If you hit any errors, share them and I'll help you fix them

## API Reference

### `POST /api/chat`

**Request body:**

```json
{
  "messages": [{ "role": "user", "content": "hello" }]
}
```

- `messages` — required array of `{ role: "user" | "assistant", content: string }`
- Max 30 messages per request; each `content` max 2000 characters

**Success response (200):**

```json
{ "reply": "Hello! How can I help you today?" }
```

**Error responses:**

| Status | Meaning |
| ------ | ------- |
| `400` | Invalid `messages` (missing, too many, bad role/content) |
| `502` | Groq API call failed |
| `503` | `GROQ_API_KEY` not configured |
| `429` | Rate limited (20 requests per 15 minutes) |

## Notes

- Model: `llama-3.3-70b-versatile`, temperature `0.5`
- Rate limit: 20 requests per 15 minutes per client
- Swap `NEXT_PUBLIC_API_URL` for the real backend URL when deploying
- The widget's placeholder and fallback messages are in Bengali — translate them if needed

## নতুন প্রজেক্টে বসানোর আগে মনে রাখুন (Notes / Limitations)

- **প্রোডাকশনে rate limit ভুল কাজ করতে পারে** — এই rate limiter (`chatLimiter`) ইউজারের IP address দেখে হিসাব করে। Vercel বা অন্য কোনো প্রক্সির পেছনে ডিপ্লয় করলে Express-কে বলে দিতে হয় `app.set('trust proxy', 1)` — নাহলে সব ইউজারকে একই IP মনে করে rate limit ভুলভাবে কাজ করতে পারে।

```typescript
const app = express();
app.set('trust proxy', 1);
```

- **এন্ডপয়েন্টে কোনো authentication নেই** — `/api/chat`-এ লগইন ছাড়াই যে কেউ হিট করতে পারবে। Rate limiter কিছুটা প্রোটেকশন দেয়, কিন্তু পুরোপুরি না — কেউ ইচ্ছা করলে স্প্যাম করে Groq quota শেষ করে দিতে পারে। নতুন প্রজেক্টে দরকার হলে লগইন-বাধ্যতামূলক করে নেওয়ার কথা ভাবুন।

- **AI মডেলের নাম hardcoded** — কোডে সরাসরি `llama-3.3-70b-versatile` বসানো আছে। Groq মাঝেমধ্যে পুরনো মডেল বন্ধ করে নতুন মডেল আনে। নতুন প্রজেক্টে বসানোর আগে [Groq-এর মডেল লিস্ট](https://console.groq.com/docs/models) চেক করে নিশ্চিত হয়ে নিন মডেলটা এখনো available কিনা।

- **secret key নিয়ে সতর্কতা** — এই README-তে কোনো আসল পাসওয়ার্ড/key নেই, শুধু placeholder আছে, তাই এটা পাবলিক রিপোতে রাখলেও সমস্যা নেই। তবে ভবিষ্যতে কোনো প্রজেক্টে টেস্ট করার সময় ভুলে কোনো আসল key/URL এই ফাইলে কপি-পেস্ট হয়ে গেলে সেটা commit করার আগে অবশ্যই চেক করে সরিয়ে নেবেন।