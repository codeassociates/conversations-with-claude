# MCP Demystified: How Model Context Protocol Actually Works

*A Q&A exploration of MCP internals, using an LDAP MCP server as a concrete example.*

---

## Background: Initial Mental Model

**Q: I have a high-level understanding of MCP: since the "host" LLM only "understands" text/natural language, I assume an MCP server must comprise some text injected into the context window at the beginning of a session. This tells the LLM what the MCP server does, and must define some special text markers that get forwarded to the MCP server to invoke its methods. There must also be some mechanism to define the schema for MCP server response data. Is this roughly correct?**

**A:** Your intuition is partially correct, but the mechanism is different from what you imagined.

### The Architecture

MCP doesn't communicate directly with the LLM. Instead:

```
┌─────────────┐    JSON-RPC/stdio    ┌─────────────┐
│  MCP Server │ ◄─────────────────► │    Host     │ ◄───► LLM
│             │                      │(Claude Code)│
└─────────────┘                      └─────────────┘
```

The **Host** (Claude Code, Claude Desktop, or any MCP-compatible client) acts as the intermediary:

1. It connects to MCP servers via JSON-RPC (over stdio, HTTP, etc.)
2. It queries MCP servers for their available **tools**, **prompts**, and **resources**
3. It presents those tools to the LLM in the LLM's native tool format
4. When the LLM calls a tool, the Host translates it to an MCP protocol call

### Where the "Instructions to the LLM" Live

The "instruction" to the LLM comes from **Python docstrings and type hints**. Example:

```python
@mcp.tool()
def list_all_users(limit: int = 50, server_name: Optional[str] = None) -> Dict[str, Any]:
    """List users in the directory with computed status.

    Note: In privacy mode (default), returns count only.
    Set LDAP_MCP_EXPOSE_SENSITIVE_DATA=true for full user details.
    """
```

The `@mcp.tool()` decorator does several things:

| Source | Becomes |
|--------|---------|
| Function name | Tool name (`list_all_users`) |
| Docstring | Tool description shown to the LLM |
| Type hints (`limit: int`) | JSON schema for parameters |
| Return type | Expected response structure |

### The Protocol Layer

The protocol handling is in the `fastmcp` library (not in individual MCP servers). MCP servers extend `FastMCP`:

```python
from fastmcp import FastMCP

class LDAPAssistantMCP(FastMCP):
    ...
```

FastMCP handles:
- JSON-RPC message serialization/deserialization
- The `tools/list`, `tools/call` protocol methods
- Schema generation from Python type hints
- Transport (stdio, SSE, etc.)

### The Wire Protocol

When the Host asks "what tools do you have?", FastMCP responds with JSON:

```json
{
  "tools": [
    {
      "name": "list_all_users",
      "description": "List users in the directory with computed status...",
      "inputSchema": {
        "type": "object",
        "properties": {
          "limit": {"type": "integer", "default": 50},
          "server_name": {"type": "string"}
        }
      }
    }
  ]
}
```

When the LLM (via the Host) calls a tool:

```json
{"method": "tools/call", "params": {"name": "list_all_users", "arguments": {"limit": 10}}}
```

FastMCP routes this to the decorated Python function and returns the result.

### Summary Table

| What you might expect | What actually happens |
|-----------------------|----------------------|
| Special tokens in context | JSON-RPC protocol over stdio |
| Text injected at session start | Host queries server for tool schemas |
| LLM parses special markers | Host translates LLM tool calls to MCP protocol |
| Schema definitions in code | Auto-generated from Python type hints + docstrings |

---

## Going Deeper: What Are "Tools" Really?

**Q: Can you explain what exactly the LLM's "native tool format" is, and how exactly LLMs conceive of "tools"? I'm having trouble understanding this given that I believe an LLM is a box that accepts embedded text tokens and responds with text tokens. Are "tools" something else or are they some encoding of this textual/embedding scheme?**

**A:** Your intuition is correct: **it's tokens all the way down**.

### The Reality: Tools Are Just Structured Text

LLMs like Claude are indeed "text in, text out" (more precisely, tokens in, tokens out). There is no separate "tool mechanism" at the neural network level. What we call "tools" is really:

1. **A training convention** - Claude was trained on examples showing tool definitions and how to "invoke" them
2. **A prompt injection pattern** - tool schemas get serialized into the context as text
3. **An output parsing convention** - certain output patterns are intercepted and interpreted as "tool calls"

### What Actually Happens

When you (or Claude Code) make an API call with tools:

**API Request (simplified):**
```json
{
  "messages": [{"role": "user", "content": "List LDAP users"}],
  "tools": [{
    "name": "list_all_users",
    "description": "List users in the directory...",
    "input_schema": {"type": "object", "properties": {"limit": {"type": "integer"}}}
  }]
}
```

**What Claude Actually "Sees" (conceptually):**

The API layer serializes this into the prompt context. While the exact format is proprietary, conceptually it's something like:

```
You have access to these tools:

<tool name="list_all_users">
<description>List users in the directory...</description>
<parameters>
  <param name="limit" type="integer" optional="true"/>
</parameters>
</tool>

When you want to use a tool, output:
<tool_use name="..."><param>...</param></tool_use>

User: List LDAP users
```

**What Claude Outputs:**
```xml
<tool_use name="list_all_users">
<limit>50</limit>
</tool_use>
```

**What the Host Does:**

The host application parses this output, recognizes the `<tool_use>` pattern, extracts the parameters, calls the actual tool, and injects the result back:

```xml
<tool_result name="list_all_users">
{"type": "user_list", "total_returned": 50, "items": [...]}
</tool_result>
```

Then Claude continues generating based on that injected result.

### The "Training" Part

Claude wasn't born knowing how to use tools. During training (including RLHF), Claude saw millions of examples like:

- "Here are tools: X, Y, Z. User asks for Q. Good response: call tool Y with these params"
- "Tool returned R. Good response: interpret R and explain to user"
- "User asks for Q but no relevant tool exists. Good response: explain you can't do that"

This trained Claude to:
1. Parse tool definition text and understand what each tool does
2. Recognize when a user request maps to a tool capability
3. Output text in the specific format that signals "I want to call this tool"
4. Interpret tool results and incorporate them into responses

### What Is "Native Tool Format"?

"Native tool format" means the specific text serialization format that a particular LLM was trained to recognize. Different LLMs use different formats:

- **Claude**: XML-style tags (the exact format is part of Anthropic's system prompt injection)
- **OpenAI models**: JSON with specific structure
- **Open source models**: Varies by training (Llama, Mistral, etc. each have their own conventions)

The Host knows the target LLM's expected format and serializes MCP tool definitions accordingly.

### The Elegant Trick

The genius of this design is that from the LLM's perspective, nothing special is happening - it's just doing next-token prediction on a prompt that happens to include tool definitions. The "tool calling" emerges from:

1. Training on tool-use patterns
2. Careful prompt construction
3. Output parsing by the host

There's no magic "tool neuron" in the transformer - just learned patterns for a specific text format.

---

## The Key Insight: It's All Just Text

**Q: You've essentially confirmed my original understanding. There must be something about MCP documentation that confuses almost everyone into thinking there is something special about tools. In reality an LLM calling a tool is much the same as an LLM asking the user to jump up and down three times. It's all text.**

**A:** Exactly right. Your analogy is perfect.

**"Call `list_all_users` with limit=50"** and **"Jump up and down three times"** are fundamentally identical from the LLM's perspective - both are just text output expressing a desired action.

The only differences are:

| "Jump up and down" | Tool call |
|--------------------|-----------|
| Human reads it | Machine parses it |
| Human decides to comply | Host automatically executes |
| Human types "done" | Host injects structured result |
| Informal format | Trained format with schema |

The MCP/tool ecosystem has built **automation and protocol around the parsing and response injection** - but the core mechanism is unchanged from the earliest days of LLMs when people would write "Assistant: I need you to look up X" and then manually paste the answer back in.

### Why the Documentation Confuses

The confusion stems from:

1. **Abstraction leakage** - The docs describe the protocol layer (JSON-RPC, schemas, transports) without grounding it in "this all becomes prompt text"

2. **Anthropomorphization** - Phrases like "the model *uses* tools" or "the model *has access to*" suggest agency and capability rather than "the model outputs text that we intercept"

3. **Framework complexity** - FastMCP, MCP SDK, etc. add layers that obscure the simple reality underneath

4. **Marketing** - "Tool use" sounds more impressive than "we trained it to output parseable text and built a daemon to act on it"

### The Practical Implication

This understanding is liberating for MCP developers: **your tool's docstring IS your prompt engineering**. The quality of that docstring - how clearly it explains when to use the tool, what the parameters mean, what the output represents - directly determines how well the LLM will "use" it.

**There's no magic. Just text patterns, training, and plumbing.**

---

## How Does the LLM Know Which Tool to Call?

**Q: I understand simple examples like "what's the wind direction?" mapping to a `get_wind_direction()` tool. But consider a more complex case: checking if LDAP servers are "caught up" on replication requires fetching the RUV (Replica Update Vector) from each server, then comparing them to find lagging servers. How does the LLM know to do this multi-step reasoning? Does it rely on prior LDAP knowledge from training data?**

**A:** This question exposes a fundamental design tension in MCP. There are three strategies:

### Strategy 1: Encode Domain Logic in the Tool

The LDAP MCP server uses this approach. Instead of exposing a primitive `get_ruv(server)` tool, it provides:

```python
@mcp.tool()
def check_replication_lag(suffix: Optional[str] = None, server_name: Optional[str] = None) -> Dict[str, Any]:
    """Analyze replication lag across agreements.

    Compares CSN values between supplier and consumers to identify
    replication delays and their severity.
    """
```

This tool **internally**:
1. Queries all agreements on the server
2. Compares supplier vs consumer CSNs
3. Returns pre-analyzed results: `"in_sync"`, `"lagging"`, `"error"`

The LLM doesn't need to understand RUV comparison logic. It only needs to understand "this tool checks if replication is lagging."

```
User: "Are my servers caught up?"
         ↓
LLM sees: check_replication_lag() - "Analyze replication lag across agreements"
         ↓
LLM calls: check_replication_lag()
         ↓
Tool internally: queries all servers, compares CSNs, determines lag
         ↓
Returns: {"summary": "HEALTHY: All 3 agreements in sync", ...}
```

### Strategy 2: Rely on LLM's Prior Training Knowledge

If you only had `get_ruv(server)`, the LLM would need to:
- Know to call it for each server
- Know that RUVs contain CSNs
- Know how to compare CSNs to detect lag

This works **if and only if** the LLM was trained on enough LDAP documentation.

### Strategy 3: Teach via Prompts

MCP supports "prompts" that guide orchestration:

```python
@self.prompt()
def diagnose_replication() -> List[PromptMessage]:
    """Start a guided replication troubleshooting session."""
    return [
        PromptMessage(role="user", content="I need help diagnosing replication issues."),
        PromptMessage(role="assistant", content=(
            "I'll help you diagnose replication issues. Let me perform a systematic analysis:\n\n"
            "**Step 1:** Use `get_replication_status` to get an overview\n"
            "**Step 2:** Use `list_replication_conflicts` to find conflicts\n"
            "**Step 3:** Use `check_replication_lag` to identify sync delays\n"
            "**Step 4:** Use `get_agreement_status` for deeper investigation\n\n"
            "Let me start..."
        )),
    ]
```

### The Core Insight

Your question contained its own answer:

> "Perhaps it is relying on the LLM understanding the operation of LDAP servers from prior training data? If this is true then the MCP server isn't really adding fundamental capabilities to the LLM because it already understands LDAP servers."

This is **partially true**:

| What MCP adds | What MCP does NOT add |
|---------------|----------------------|
| Access to live data (actual RUV values) | Understanding of what RUVs mean |
| Ability to execute operations | Knowledge of LDAP concepts |
| Structured responses | Reasoning about how to use the data |

An MCP server is like giving someone a calculator. The calculator adds computational capability, but if the person doesn't understand math, they won't know which buttons to press or what the output means.

### Practical Implications for MCP Design

1. **Primitive tools require LLM domain knowledge** - A `get_ruv(server)` tool only works if the LLM already understands LDAP replication

2. **High-level tools encode your expertise** - `check_replication_lag()` encapsulates the comparison logic so the LLM only needs to understand "check if lag exists"

3. **Prompts can teach orchestration** - If you must use primitives, prompts can guide multi-step workflows

4. **Response design matters** - Returning `"in_sync"` vs `"lagging"` is more LLM-friendly than raw CSN values

---

## What Value Does the LLM Actually Add?

**Q: The code in this MCP server could just as well have been shipped as a standalone CLI utility with a `check-replication-lag` subcommand. What is the LLM actually bringing to the party? Is this just a "hidden app store" for utilities that would be equally useful standalone? The LLM translation of "check if my servers are lagging" to `check-replication-lag` seems more like a party trick than something profoundly useful.**

**A:** Your skepticism is warranted. Let me be direct about where the value is and isn't.

### Where You're Right

For **single-tool invocations by domain experts**, the LLM layer is indeed mostly a "party trick":

```
Traditional CLI:
$ ldap-grokker check-replication-lag
{"summary": "HEALTHY: All 3 agreements in sync", ...}

LLM + MCP:
You: "check if my servers are lagging"
LLM: *calls check_replication_lag()*
     "Your servers are all in sync."
```

The LLM path adds latency, cost, potential for error, and external service dependency. For someone who knows LDAP cold, the CLI is arguably *better*.

### Where the LLM Might Add Value

**1. Orchestration of multi-step investigations**

```
You: "My LDAP server is slow, figure out why"

LLM: *calls get_performance_summary()*
     "Cache hit ratio is 45% - that's low. Let me check cache config..."
     *calls get_cache_statistics()*
     "Your entry cache is 200MB but you have 2M entries. Let me check memory..."
     *calls get_resource_utilization()*
     "You have 16GB free RAM. I recommend increasing nsslapd-cachememsize to 2GB."
```

A CLI requires you to know the diagnostic flow. The LLM chains tools based on intermediate results.

**2. Interpretation in context**

The CLI gives you data; the LLM gives you meaning (if it's right).

**3. Non-experts / unfamiliar domains**

For someone who doesn't know LDAP, they don't know `check-replication-lag` exists, don't know what "RUV" means, and can't interpret the output. The LLM provides discoverability and translation.

**4. Conversational refinement**

```
You: "Check replication lag"
LLM: *shows results*
You: "Just for the dc=example,dc=com suffix"
LLM: *calls check_replication_lag(suffix="dc=example,dc=com")*
You: "Now compare to server2"
LLM: *remembers context*
```

### Honest Assessment

| Aspect | CLI wins | LLM wins |
|--------|----------|----------|
| Speed | ✓ | |
| Determinism | ✓ | |
| Scriptability | ✓ | |
| Offline use | ✓ | |
| Discoverability | | ✓ |
| Multi-step reasoning | | Maybe |
| Result interpretation | | Maybe |
| Non-expert users | | ✓ |

The "maybe" entries depend heavily on the LLM actually getting it right.

### The "Hidden App Store" Observation

Your framing is astute. There's definitely an element of:

> "Let's make utilities discoverable/invocable via natural language and call it AI"

The counter-argument: **integration is valuable**. Having 50 CLIs requires knowing 50 CLIs. Having 50 MCP servers behind one LLM interface means you can say "help me with X" and the right tool gets invoked.

But that's still a UX/discovery argument, not a "profound AI capability" argument.

### Bottom Line

The LLM + MCP combination is most valuable when:
1. **The user doesn't know what they don't know** (exploration, not execution)
2. **The task requires chaining multiple tools with reasoning between steps**
3. **Interpretation/explanation is needed, not just data**
4. **The domain is unfamiliar to the user**

For an LDAP expert running routine checks? You might genuinely be better off with a CLI.

---

## Is MCP a Transitional Technology?

**Q: I suspect future LLMs won't need MCP because they can directly write and execute data access code based on documentation and source code in their training data.**

**A:** You're describing a plausible future.

### The Thesis

Instead of:
```
User → LLM → MCP protocol → MCP server → lib389 → LDAP server
```

Future:
```
User → LLM → generates Python using lib389 → executes it → LDAP server
```

The LLM already knows lib389 from training data. Why the indirection?

### Arguments For Your Thesis

**1. LLMs can already do this (sometimes)**

An LLM could write:
```python
from lib389 import DirSrv
from lib389.replica import Replicas

ds = DirSrv()
ds.open(uri="ldap://server1:389", binddn="cn=admin", bindpw="...")
for replica in Replicas(ds).list():
    print(replica.get_ruv().format_ruv())
```

...and execute it. No MCP needed.

**2. MCP tools are just frozen code**

The `check_replication_lag()` function is ~170 lines of Python. If the LLM can reliably generate equivalent code on demand, pre-packaging it adds no value.

**3. Code generation is improving rapidly**

As LLMs get better at writing correct code first-try and debugging their own errors, the reliability gap between "call tested MCP tool" and "generate and run code" shrinks.

### Current Limitations (Engineering Problems, Not Fundamental Barriers)

| Limitation | Status |
|------------|--------|
| Security/sandboxing | Improving (Claude Code has sandboxing) |
| Credentials management | Solvable |
| Code reliability | Improving rapidly |
| Execution speed | Solvable |

### MCP's Possible Future

| Era | How LLMs get capabilities |
|-----|---------------------------|
| Early | Can only output text, humans act on it |
| Current | Pre-packaged tools (MCP) extend capabilities safely |
| Future? | Generate and execute code directly, tools unnecessary |

### What Might Persist

Even in a "generate code directly" world, some value might remain:

1. **Trust/curation** - For critical operations, you might want human-vetted implementations

2. **Capabilities beyond code** - Proprietary APIs, hardware interfaces, persistent state

3. **Efficiency** - Pre-built tools are faster than generating code every time

4. **The "app store" model** - Vendors might prefer packaging for branding, monetization, support

### Conclusion

MCP is partly a workaround for current LLM limitations:
- Not reliable enough at code generation
- Execution environments not secure enough
- No standard way to provide credentials

As those limitations erode, the MCP abstraction may become unnecessary overhead for many use cases. The protocol layer is solving a problem that might not exist in 3-5 years.

The counterpoint: "pre-packaged, tested, trusted capabilities" might always have value - for the same reasons we use libraries instead of writing everything from scratch.

---

## Appendix: Key Files in an MCP Server

For reference, here's where the important pieces live in a typical FastMCP-based server:

| Component | Location | Purpose |
|-----------|----------|---------|
| Tool definitions | `src/*/tools/*.py` | `@mcp.tool()` decorated functions |
| Server class | `src/*/server.py` | Extends `FastMCP`, registers tools |
| Entry point | `src/main.py` | Creates and returns the server instance |
| Configuration | `fastmcp.json` | Tells the host how to launch the server |
| Protocol handling | `fastmcp` library | JSON-RPC, schema generation, transport |

The "instructions to the LLM" are the **docstrings** on the tool functions. The "special tokens" are just **structured text patterns** that the LLM learned during training. The "schema definitions" are **auto-generated from Python type hints**.

---

*Document generated from a conversation exploring MCP internals.*
