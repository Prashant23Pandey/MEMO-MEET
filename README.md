**Deal Intelligence Agent Project: "PitchMind" — The AI Sales Deal Intelligence Agent**

### Why This Wins
- Immediately answerable: "Would someone pay $50/month?" → **Yes. Sales reps pay $100+/month for Gong alone.**
- Memory is *obviously* the value: without it, you have a chatbot; with it, you have institutional deal knowledge.
- 60-second demo is trivial: show interaction 1 vs interaction 20, judges immediately get it.

### User Persona
**Marcus**, a B2B SaaS Account Executive at a 50-person startup. He manages 30+ active deals, has calls daily, and loses 45 minutes before each call re-reading CRM notes, Slack threads, and email chains to remember what the prospect said 3 weeks ago.

### The 3-Interaction Story

| Interaction | What Marcus asks | Without memory (bad) | With memory (PitchMind) |
|---|---|---|---|
| **#1** | "Brief me on the Acme Corp deal" | "I don't have any info on that deal." | Stores first mention of Acme: industry, contact name, budget hint |
| **#5** | "They pushed back on pricing again" | "Here are some generic objection-handling tips." | "Acme raised pricing concerns twice before. Last time you offered a quarterly payment plan — they went quiet after that. Try ROI framing instead." |
| **#20** | "I have a call with Acme in 10 mins" | Generic agenda template | Full pre-call brief: decision-maker names, every objection ever raised, what worked, competitor (Salesforce) mentioned in week 2, promises made on the last call |

---

## 2. COMPLETE ARCHITECTURAL STRUCTURE

### Frontend: Next.js 14 + Tailwind CSS
- **Chat Interface** — standard message input/output, streaming responses
- **Memory Monitor Panel** (the secret weapon for judges) — a live sidebar that lights up whenever Hindsight is queried, showing:
  - Which memories were retrieved
  - The relevance score of each memory
  - How many past interactions informed this response
- **Deal Sidebar** — shows active deal context (prospect name, stage, last touch)

### Backend: Python + FastAPI
- Handles chat POST requests from frontend
- Orchestrates the memory query → LLM call → memory save lifecycle
- Returns streaming SSE (Server-Sent Events) to the frontend for real-time typing effect

### Protocols
```
Browser ←──SSE streaming──→ FastAPI Backend ←──REST──→ Groq LLM
                                    ↕
                              Hindsight Cloud (REST)
```

---

## 3. AGENT & MEMORY LIFECYCLE PIPELINE

```
User types message
       ↓
POST /api/chat  (FastAPI)
       ↓
① Extract deal name + user intent from message
       ↓
② Query Hindsight: "fetch memories for deal=Acme, user=marcus"
   → Returns: [past objections, promises made, competitor mentions]
       ↓
③ Build enriched prompt:
   SYSTEM: "You are a deal intelligence assistant..."
   MEMORY CONTEXT: [retrieved memories injected here]
   USER: "They pushed back on pricing again"
       ↓
④ Send to Groq (qwen/qwen3-32b) → stream response back
       ↓
⑤ Save new interaction to Hindsight:
   { deal: "Acme", content: "pricing objection raised again", 
     metadata: { type: "objection", date: today } }
       ↓
Stream SSE chunks → Frontend renders response + Memory Panel updates
```

---

## 4. THE "I KNOW NOTHING" BEGINNER BLUEPRINT

### Step 1: Terminal Setup

```bash
# Create project
mkdir pitchmind && cd pitchmind

# Backend setup (Python)
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# Install backend deps
pip install fastapi uvicorn groq httpx python-dotenv sse-starlette

# Frontend setup
npx create-next-app@latest frontend --typescript --tailwind --app
cd frontend && npm install
```

### Step 2: Project Structure

```
pitchmind/
├── backend/
│   ├── main.py              # FastAPI app entry point
│   ├── agent.py             # Core agent logic
│   ├── memory.py            # Hindsight client wrapper
│   ├── prompts.py           # System prompt templates
│   └── .env                 # GROQ_API_KEY, HINDSIGHT_API_KEY
├── frontend/
│   ├── app/
│   │   ├── page.tsx         # Main chat page
│   │   └── api/             # (unused — using FastAPI instead)
│   ├── components/
│   │   ├── ChatWindow.tsx   # Message thread
│   │   ├── MemoryPanel.tsx  # 🔑 The live memory monitor
│   │   └── DealSidebar.tsx  # Active deal context
│   └── .env.local           # NEXT_PUBLIC_API_URL=http://localhost:8000
└── README.md
```

### Step 3: Starter Code

**`backend/.env`**
```
GROQ_API_KEY=your_groq_key_here
HINDSIGHT_API_KEY=your_hindsight_key_here
HINDSIGHT_BASE_URL=https://ui.hindsight.vectorize.io
```

**`backend/memory.py`** — Hindsight wrapper
```python
import httpx
import os

HINDSIGHT_BASE = os.getenv("HINDSIGHT_BASE_URL")
HINDSIGHT_KEY = os.getenv("HINDSIGHT_API_KEY")

headers = {
    "Authorization": f"Bearer {HINDSIGHT_KEY}",
    "Content-Type": "application/json"
}

async def fetch_memories(user_id: str, deal_name: str, query: str) -> list:
    """Query Hindsight for relevant past interactions."""
    async with httpx.AsyncClient() as client:
        try:
            resp = await client.post(
                f"{HINDSIGHT_BASE}/api/memory/search",
                headers=headers,
                json={
                    "query": query,
                    "user_id": user_id,
                    "filters": {"deal": deal_name},
                    "top_k": 5
                }
            )
            resp.raise_for_status()
            return resp.json().get("memories", [])
        except Exception as e:
            print(f"[Memory fetch error] {e}")
            return []  # Graceful fallback — agent still works, just without memory

async def save_memory(user_id: str, deal_name: str, content: str, metadata: dict):
    """Save a new interaction to Hindsight after the agent responds."""
    async with httpx.AsyncClient() as client:
        try:
            await client.post(
                f"{HINDSIGHT_BASE}/api/memory/store",
                headers=headers,
                json={
                    "user_id": user_id,
                    "content": content,
                    "metadata": {"deal": deal_name, **metadata}
                }
            )
        except Exception as e:
            print(f"[Memory save error] {e}")
```

**`backend/agent.py`** — Core agent with Groq + memory injection
```python
from groq import Groq
from memory import fetch_memories, save_memory
import os

client = Groq(api_key=os.getenv("GROQ_API_KEY"))

SYSTEM_PROMPT = """You are PitchMind, an elite sales deal intelligence assistant.
You have access to the full memory of every past interaction about this deal.
Use it. Be specific. Reference past objections, promises, and context by name.
Never give generic advice when you have deal-specific memory available.
If you have no memory yet, be honest and ask questions to start building it."""

async def run_agent(user_id: str, deal_name: str, user_message: str):
    # 1. Fetch relevant memories
    memories = await fetch_memories(user_id, deal_name, user_message)
    
    # 2. Build memory context string
    memory_context = ""
    if memories:
        memory_context = "\n\n[DEAL MEMORY — USE THIS]\n"
        for m in memories:
            memory_context += f"- {m['content']} (recorded: {m.get('metadata', {}).get('date', 'unknown')})\n"
    
    # 3. Call Groq with injected memory
    try:
        response = client.chat.completions.create(
            model="qwen/qwen3-32b",  # or openai/gpt-oss-120b
            messages=[
                {"role": "system", "content": SYSTEM_PROMPT + memory_context},
                {"role": "user", "content": user_message}
            ],
            max_tokens=800,
            stream=False
        )
        answer = response.choices[0].message.content
    except Exception as e:
        # Handle function calling errors gracefully
        print(f"[LLM error] {e}")
        answer = "I encountered an issue generating a response. Please try again."
    
    # 4. Save this interaction to memory AFTER responding
    await save_memory(user_id, deal_name, 
        content=f"User said: {user_message} | Agent replied: {answer[:200]}",
        metadata={"type": "interaction", "date": str(__import__('datetime').date.today())}
    )
    
    return answer, memories  # Return memories too so frontend can show them
```

**`backend/main.py`** — FastAPI entry point
```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from agent import run_agent
from dotenv import load_dotenv

load_dotenv()
app = FastAPI()

app.add_middleware(CORSMiddleware, allow_origins=["*"], allow_methods=["*"], allow_headers=["*"])

class ChatRequest(BaseModel):
    user_id: str
    deal_name: str
    message: str

@app.post("/api/chat")
async def chat(req: ChatRequest):
    answer, memories = await run_agent(req.user_id, req.deal_name, req.message)
    return {
        "response": answer,
        "memories_used": memories  # Sent to frontend Memory Monitor Panel
    }

# Run: uvicorn main:app --reload
```

**`frontend/components/MemoryPanel.tsx`** — The judge-impressing panel
```tsx
// Shows live which memories Hindsight retrieved for the current response
export function MemoryPanel({ memories }: { memories: any[] }) {
  return (
    <div className="border-l border-zinc-700 p-4 w-72 bg-zinc-900 text-sm">
      <h3 className="text-emerald-400 font-mono font-bold mb-3">
        ⚡ HINDSIGHT MEMORY
      </h3>
      {memories.length === 0 ? (
        <p className="text-zinc-500 italic">No past context retrieved yet.</p>
      ) : (
        <div className="space-y-2">
          <p className="text-zinc-400">{memories.length} memories retrieved:</p>
          {memories.map((m, i) => (
            <div key={i} className="bg-zinc-800 rounded p-2 border border-zinc-600">
              <p className="text-white text-xs">{m.content?.slice(0, 100)}...</p>
              <p className="text-emerald-500 text-xs mt-1">
                relevance: {(m.score * 100).toFixed(0)}%
              </p>
            </div>
          ))}
        </div>
      )}
    </div>
  )
}
```

---

## 5. FREE RESOURCE & TOOLING CHECKLIST

| Resource | Link | Action |
|---|---|---|
| **Hindsight Cloud** | https://ui.hindsight.vectorize.io | Register → Billing → Enter `MEMHACK6` for $50 credit |
| **Hindsight Docs** | https://hindsight.vectorize.io | Read the `/api/memory/search` and `/store` endpoints |
| **Hindsight GitHub** | https://github.com/vectorize-io/hindsight | Star it (looks good), check examples |
| **Hindsight Slack** | Join via docs page | Ask questions — organizers monitor this during hackathon |
| **Groq Console** | https://console.groq.com | Sign up → API Keys → copy key |
| **OpenClaw Plugin** | https://hindsight.vectorize.io/sdks/integrations/openclaw | Optional: use if you want agent framework abstractions |
| **Synthetic Data** | ChatGPT or Claude | Generate 20 fake deal interactions to pre-seed memory for demo |

---

## Demo Script (60 seconds)

> *"Meet Marcus, a sales rep with 30 active deals. Before PitchMind — [show blank generic response]. After 20 interactions — [show response citing Acme's pricing objections from 3 weeks ago, the competitor they mentioned, the promises made]. Watch the Memory Panel on the right — those green cards are Hindsight retrieving exact past context in real time. The agent doesn't just remember — it gets strategically smarter with every call."*

Now let me build you a visual architecture diagram you can use for the submission:The architecture diagram above is clickable — tap any box to dive deeper into that component.

---

## Quick-start command sequence (copy-paste ready)

```bash
# 1. Clone / init
mkdir pitchmind && cd pitchmind
git init

# 2. Backend
python -m venv venv && source venv/bin/activate
pip install fastapi uvicorn groq httpx python-dotenv sse-starlette

# 3. Frontend
npx create-next-app@latest frontend --typescript --tailwind --app --no-eslint
cd frontend && npm install && cd ..

# 4. Run backend
cd backend && uvicorn main:app --reload --port 8000

# 5. Run frontend (new terminal)
cd frontend && npm run dev
```

---

## The single most important thing for your score

**Pre-seed your Hindsight instance before the demo.** Run a script the night before that fires 15–20 synthetic interactions into Hindsight for a fake deal ("Acme Corp / Q3 Enterprise Contract"). When you demo live, interaction #1 will already "know" Acme's objections, competitors, and stakeholder names — making the memory benefit immediately visible to judges in under 10 seconds. This is the difference between a 20% and a 25% on the Hindsight memory criterion.
