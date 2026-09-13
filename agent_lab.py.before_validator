import os, json, re, urllib.request, urllib.error, math
from pathlib import Path
from http.server import ThreadingHTTPServer, BaseHTTPRequestHandler

ROOT = Path.cwd() / "case"
API_URL = "https://openrouter.ai/api/v1/chat/completions"
MODEL = "openrouter/free"

ROOT.mkdir(exist_ok=True)

# ---------------- DEMO CASE ----------------

(ROOT / "incident.txt").write_text("""INCIDENT REPORT

Case: TX-1042
Question: Was transaction TX-1042 processed twice?

Initial report:
At approximately 10:14:32, transaction TX-1042 was submitted.
The operator reported seeing two successful processing messages.

Important:
This report is an initial observation, not proof of duplicate settlement.
""", encoding="utf-8")

(ROOT / "transactions.csv").write_text("""timestamp,transaction_id,status,amount,reference
10:14:32.100,TX-1042,SUBMITTED,125.00,R-7781
10:14:32.880,TX-1042,APPROVED,125.00,R-7781
10:14:33.020,TX-1042,SETTLEMENT,125.00,R-7781
10:14:33.021,TX-1042,RETRY_REJECTED,125.00,R-7781
10:15:01.100,TX-1043,APPROVED,80.00,R-7782
""", encoding="utf-8")

(ROOT / "operator_notes.txt").write_text("""OPERATOR NOTES

TX-1042:
The terminal displayed a timeout after the first request.
The operator pressed retry once.

The retry received a duplicate/retry rejection.
No second successful settlement was observed at the operator terminal.

Note:
A second message does NOT necessarily mean a second financial settlement.
""", encoding="utf-8")

(ROOT / "system.log").write_text("""10:14:32.100 INFO request received tx=TX-1042 ref=R-7781
10:14:32.880 INFO authorization approved tx=TX-1042 ref=R-7781
10:14:33.020 INFO settlement completed tx=TX-1042 ref=R-7781 amount=125.00
10:14:33.021 WARN retry rejected duplicate tx=TX-1042 ref=R-7781
10:14:33.022 INFO duplicate prevention rule matched tx=TX-1042
10:14:40.000 INFO unrelated transaction tx=TX-1043
""", encoding="utf-8")

# ---------------- TOOLS ----------------

def safe_path(name):
    p = (ROOT / name).resolve()
    if not str(p).startswith(str(ROOT.resolve())):
        raise ValueError("Path outside case directory")
    return p

def list_files():
    return {
        "files": [
            str(p.relative_to(ROOT))
            for p in ROOT.rglob("*")
            if p.is_file()
        ]
    }

def read_file(path):
    p = safe_path(path)
    if not p.exists() or not p.is_file():
        return {"error": "File not found"}
    text = p.read_text(encoding="utf-8", errors="replace")
    return {"path": path, "content": text[:12000]}

def search_files(term):
    hits = []
    term_l = term.lower()
    for p in ROOT.rglob("*"):
        if p.is_file():
            text = p.read_text(encoding="utf-8", errors="replace")
            for i, line in enumerate(text.splitlines(), 1):
                if term_l in line.lower():
                    hits.append({
                        "file": str(p.relative_to(ROOT)),
                        "line": i,
                        "text": line[:500]
                    })
    return {"term": term, "matches": hits[:100]}

def calculate(expression):
    # Deliberately restricted calculator.
    if not re.fullmatch(r"[0-9+*/().,%\\s-]+", expression):
        return {"error": "Only basic arithmetic is allowed"}
    try:
        expr = expression.replace("%", "/100")
        return {"expression": expression, "result": eval(expr, {"__builtins__": {}}, {})}
    except Exception as e:
        return {"error": str(e)}

TOOLS = {
    "list_files": list_files,
    "read_file": read_file,
    "search_files": search_files,
    "calculate": calculate,
}

TOOL_SCHEMAS = [
    {
        "type": "function",
        "function": {
            "name": "list_files",
            "description": "List all files available in the evidence case.",
            "parameters": {"type": "object", "properties": {}}
        }
    },
    {
        "type": "function",
        "function": {
            "name": "read_file",
            "description": "Read an evidence file. Use only after discovering the file.",
            "parameters": {
                "type": "object",
                "properties": {
                    "path": {"type": "string"}
                },
                "required": ["path"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "search_files",
            "description": "Search every evidence file for a term and return matching lines.",
            "parameters": {
                "type": "object",
                "properties": {
                    "term": {"type": "string"}
                },
                "required": ["term"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "calculate",
            "description": "Perform basic arithmetic when numerical verification is needed.",
            "parameters": {
                "type": "object",
                "properties": {
                    "expression": {"type": "string"}
                },
                "required": ["expression"]
            }
        }
    }
]

# ---------------- MODEL ----------------

def call_model(messages, force_tool=False):
    key = os.environ.get("OPENROUTER_API_KEY")
    if not key:
        raise RuntimeError("OPENROUTER_API_KEY is missing in this Terminal.")

    payload = {
        "model": MODEL,
        "messages": messages,
        "tools": TOOL_SCHEMAS,
        "temperature": 0.1,
    }

    if force_tool:
        payload["tool_choice"] = {
            "type": "function",
            "function": {"name": "list_files"}
        }

    req = urllib.request.Request(
        API_URL,
        data=json.dumps(payload).encode(),
        headers={
            "Authorization": "Bearer " + key,
            "Content-Type": "application/json",
            "HTTP-Referer": "http://localhost:8000",
            "X-Title": "Evidence Detective"
        },
        method="POST"
    )

    try:
        with urllib.request.urlopen(req, timeout=90) as r:
            return json.loads(r.read().decode())
    except urllib.error.HTTPError as e:
        body = e.read().decode(errors="replace")
        raise RuntimeError(f"OpenRouter HTTP {e.code}: {body[:1000]}")

# ---------------- AGENT ----------------

SYSTEM = """You are Evidence Detective, a rigorous evidence-analysis agent.

Your job is NOT to guess.
You must inspect the available evidence using tools before reaching a conclusion.

Rules:
1. Start by discovering available files.
2. Gather evidence relevant to the user's claim.
3. Search for important identifiers when useful.
4. Look for BOTH supporting and contradicting evidence.
5. If numerical reasoning is needed, use the calculator tool.
6. Never claim you inspected a file unless a tool returned its contents.
7. Distinguish an observation from proof.
8. End with a structured verdict:
   VERDICT: SUPPORTED / PARTIALLY SUPPORTED / NOT SUPPORTED / INSUFFICIENT EVIDENCE
   CONFIDENCE: 0-100%
   SUPPORTING EVIDENCE:
   CONTRADICTING EVIDENCE:
   REASONING:
   FILES INSPECTED:
9. Do not reveal hidden chain-of-thought. Give concise evidence-based reasoning only.
"""

def run_agent(task):
    messages = [
        {"role": "system", "content": SYSTEM},
        {"role": "user", "content": task}
    ]

    trace = []
    ledger = []
    max_steps = 10

    for step in range(1, max_steps + 1):
        force = (step == 1)
        data = call_model(messages, force_tool=force)
        msg = data["choices"][0]["message"]

        tool_calls = msg.get("tool_calls") or []

        if not tool_calls:
            answer = msg.get("content", "")
            trace.append({"step": step, "type": "final", "text": answer})
            return {
                "trace": trace,
                "answer": answer,
                "ledger": ledger
            }

        messages.append(msg)

        for tc in tool_calls:
            name = tc["function"]["name"]
            try:
                args = json.loads(tc["function"].get("arguments", "{}"))
            except:
                args = {}

            trace.append({
                "step": step,
                "type": "tool",
                "tool": name,
                "args": args
            })

            fn = TOOLS.get(name)
            if not fn:
                result = {"error": "Unknown tool"}
            else:
                try:
                    result = fn(**args)
                except Exception as e:
                    result = {"error": str(e)}

            # Build a lightweight evidence ledger from tool results.
            if name == "read_file" and "content" in result:
                ledger.append({
                    "source": result.get("path"),
                    "type": "inspected",
                    "status": "evidence available"
                })

            if name == "search_files":
                for hit in result.get("matches", [])[:20]:
                    ledger.append({
                        "source": f'{hit["file"]}:L{hit["line"]}',
                        "type": "search_hit",
                        "status": "evidence available"
                    })

            messages.append({
                "role": "tool",
                "tool_call_id": tc["id"],
                "content": json.dumps(result, ensure_ascii=False)
            })

    return {
        "trace": trace,
        "answer": "Agent stopped after reaching the safety step limit.",
        "ledger": ledger
    }

# ---------------- WEB UI ----------------

HTML = r"""<!doctype html>
<html>
<head>
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>Evidence Detective</title>
<style>
body{font-family:system-ui,sans-serif;background:#10141c;color:#eee;margin:0;padding:16px}
.card{background:#191f2b;border-radius:16px;padding:16px;margin-bottom:14px}
h1{margin:0 0 5px;font-size:25px}
.sub{color:#9da8b8;font-size:14px}
textarea{width:100%;box-sizing:border-box;background:#0d1118;color:#fff;border:1px solid #374151;border-radius:12px;padding:12px;font-size:16px;min-height:105px}
button{width:100%;border:0;border-radius:12px;padding:14px;font-size:17px;font-weight:700;margin-top:10px}
.step{padding:9px;border-left:3px solid #64748b;margin:8px 0;background:#111722;border-radius:7px}
.tool{font-family:monospace}
.answer{white-space:pre-wrap;line-height:1.5}
.badge{display:inline-block;padding:4px 8px;border-radius:8px;background:#293548;margin:3px;font-size:12px}
</style>
</head>
<body>
<div class="card">
<h1>🔎 Evidence Detective</h1>
<div class="sub">A real Tool-Calling Agent investigating a sandbox evidence case.</div>
</div>

<div class="card">
<textarea id="task">Investigate whether transaction TX-1042 was actually processed twice. Inspect the available evidence, look for supporting and contradicting evidence, and give a final verdict based only on the files.</textarea>
<button id="run" onclick="runAgent()">Run Investigation</button>
</div>

<div class="card">
<h3>🧭 Execution Trace</h3>
<div id="trace">Waiting for investigation...</div>
</div>

<div class="card">
<h3>⚖️ Evidence Ledger</h3>
<div id="ledger">No evidence collected yet.</div>
</div>

<div class="card">
<h3>📋 Verdict</h3>
<div id="answer" class="answer">No verdict yet.</div>
</div>

<script>
async function runAgent(){
 const b=document.getElementById('run');
 b.disabled=true;b.textContent='Investigating...';
 document.getElementById('trace').innerHTML='Agent is working...';
 document.getElementById('answer').textContent='';
 document.getElementById('ledger').innerHTML='';
 try{
   const r=await fetch('/run',{
     method:'POST',
     headers:{'Content-Type':'application/json'},
     body:JSON.stringify({task:document.getElementById('task').value})
   });
   const d=await r.json();

   document.getElementById('trace').innerHTML=d.trace.map(x=>{
     if(x.type==='tool')
       return `<div class="step"><b>Step ${x.step}</b> → <span class="tool">${x.tool}</span><br><span class="sub">${JSON.stringify(x.args)}</span></div>`;
     return `<div class="step"><b>Step ${x.step}</b> → FINAL</div>`;
   }).join('');

   document.getElementById('ledger').innerHTML=d.ledger.length
     ? d.ledger.map(x=>`<span class="badge">${x.source} — ${x.status}</span>`).join('')
     : 'No ledger entries.';

   document.getElementById('answer').textContent=d.answer;
 }catch(e){
   document.getElementById('answer').textContent='ERROR: '+e;
 }
 b.disabled=false;b.textContent='Run Investigation';
}
</script>
</body>
</html>
"""

class Handler(BaseHTTPRequestHandler):
    def send_json(self, obj):
        raw=json.dumps(obj,ensure_ascii=False).encode()
        self.send_response(200)
        self.send_header("Content-Type","application/json; charset=utf-8")
        self.send_header("Content-Length",str(len(raw)))
        self.end_headers()
        self.wfile.write(raw)

    def do_GET(self):
        if self.path == "/":
            raw=HTML.encode()
            self.send_response(200)
            self.send_header("Content-Type","text/html; charset=utf-8")
            self.send_header("Content-Length",str(len(raw)))
            self.end_headers()
            self.wfile.write(raw)
        else:
            self.send_response(404)
            self.end_headers()

    def do_POST(self):
        if self.path != "/run":
            self.send_response(404);self.end_headers();return
        try:
            n=int(self.headers.get("Content-Length","0"))
            body=json.loads(self.rfile.read(n).decode())
            result=run_agent(body.get("task",""))
            self.send_json(result)
        except Exception as e:
            self.send_json({
                "trace":[{"step":1,"type":"error","text":str(e)}],
                "answer":"ERROR: "+str(e),
                "ledger":[]
            })

    def log_message(self,*args):
        pass

print("")
print("🔎 Evidence Detective is ready")
print("📁 Evidence case:", ROOT)
print("🌐 http://127.0.0.1:8000")
print("")

ThreadingHTTPServer(("0.0.0.0",8000),Handler).serve_forever()
