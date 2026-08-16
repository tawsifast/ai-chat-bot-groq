backend-এ (এই index.ts ফাইলেই) ম্যানুয়ালি চ্যাট এন্ডপয়েন্ট যোগ করার ধাপগুলো দিচ্ছি:

ধাপ ১ — .env ও .env.example-এ key যোগ করুন

GROQ_API_KEY=your_groq_api_key_here

(আসল key শুধু .env-এ বসান, .env.example-এ placeholder রাখুন)

ধাপ ২ — একটা নতুন rate limiter বানান

checkoutLimiter এর ঠিক নিচে এটা যোগ করুন:

typescript
const chatLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  limit: 20,
  message: {
    message: "Too many requests. Please try again in a few minutes.",
  },
  standardHeaders: true,
  legacyHeaders: false,
});

ধাপ ৩ — চ্যাট রুট যোগ করুন

/api/reviews/latest রুটের নিচে (Reviews সেকশনের শেষে) এটা বসান:

typescript
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

কেন এভাবে লিখলাম:

আলাদা chatLimiter — যাতে চ্যাট স্প্যাম হলেও checkout/global limiter প্রভাবিত না হয়
Groq SDK ইনস্টল না করে সরাসরি fetch ব্যবহার করলাম — এতে আপনার existing dependencies-এ নতুন প্যাকেজ যোগ করা লাগছে না, আর Node 18+ এ fetch built-in থাকে
message validation প্যাটার্নটা আপনার checkout রুটের মতোই রাখলাম, যাতে কোডবেসের স্টাইলের সাথে মেলে


  Run this code in powershall terminal for testing
  "Invoke-RestMethod -Uri "http://localhost:5000/api/chat" -Method Post -ContentType "application/json" -Body '{"messages":[{"role":"user","content":"hello"}]}'"