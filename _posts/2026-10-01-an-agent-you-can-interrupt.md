---
title: "An agent you can interrupt"
date: 2026-10-01
---

**Summary:** A plain event loop is enough to make a conversational LLM behave like a
ReAct agent, with no planner and no framework. Extending that loop to tools that have
not finished yet is what turns a static transcript into an agent you can keep talking
to. The hard part is not running the tool in the background; it is keeping the
conversation coherent while it runs. I tried this two ways — a message board that is
rewritten every turn, and Anthropic's Claude Agent SDK — and this is what each gets
right and wrong.

If you have ever typed a steering message to GitHub Copilot while it was waiting on a
long tool call, and watched your message sit there until the tool finished, you already
know this is not a solved problem — not even for the people building the leading agents.

I care about this less as a feature than as a **paradigm**. Frameworks come and go, but
the underlying mechanism survives them. Programmatic tool calling is the obvious
precedent: it is largely settled now, some frameworks do it better than others, and
knowing how it actually works lets you evaluate any new framework's version of it in
minutes. Asynchronous tool calls are at the stage programmatic tool calling was a few
years ago, and the same thing applies. What follows is an attempt to find the smallest
mechanisms that do the job, so they can be recognised in whatever ships next.

## What came out of it

- **An event loop is sufficient to produce ReAct.** Nothing has to be taught the
  observe-think-act cycle. If a tool result arrives as an event that grants another
  turn, the cycle falls out on its own.
- **A transcript is the wrong shape for pending work.** Once results can arrive late,
  a linear history cannot say what is still owed. Restructuring the history the agent
  reads into a **message board** — rewritten every turn, tool calls attached to the
  message that caused them — keeps the agent coherent for far longer.
- **Anthropic's SDK gets the mechanics and misses the bookkeeping.** Background work,
  cancellation and the unprompted turn are all native. But every asynchronous tool has
  to be disguised as an agent, the framework forbids the model from checking on
  progress, and nothing stops a pending task from quietly ageing out of context.

The code is in [pslusarz/async-agent](https://github.com/pslusarz/async-agent).

## The loop is enough

Start with the simplest possible arrangement: a loop that hands the model a turn, and
hands it another turn whenever something happens. Give it one tool, `temperature`, and
assume for now that it answers instantly.

<svg viewBox="0 0 700 215" role="img" aria-label="State diagram: a user message leads to an LLM turn, which either answers or calls a tool; the tool result returns as an event that grants another turn." style="max-width:100%;height:auto;display:block;margin:1.5rem 0">
  <defs>
    <marker id="ah" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M0,0 L10,5 L0,10 z" fill="#666"/>
    </marker>
  </defs>
  <g font-family="-apple-system,BlinkMacSystemFont,Segoe UI,Helvetica,Arial,sans-serif" font-size="13">
    <rect x="8" y="42" width="112" height="42" rx="4" fill="#e7effa" stroke="#1a4f8a"/>
    <text x="64" y="68" text-anchor="middle" fill="#1a4f8a" font-weight="600">user message</text>

    <rect x="168" y="42" width="124" height="42" rx="4" fill="#f4f4f4" stroke="#777"/>
    <text x="230" y="68" text-anchor="middle" fill="#333" font-weight="600">LLM turn</text>

    <rect x="344" y="8" width="112" height="38" rx="4" fill="#f4f4f4" stroke="#777"/>
    <text x="400" y="32" text-anchor="middle" fill="#333" font-weight="600">answer</text>

    <rect x="344" y="92" width="124" height="38" rx="4" fill="#fdf0d5" stroke="#8a5a00"/>
    <text x="406" y="116" text-anchor="middle" fill="#8a5a00" font-weight="600">tool call</text>

    <rect x="516" y="92" width="130" height="38" rx="4" fill="#e3f3e8" stroke="#1d6b3f"/>
    <text x="581" y="116" text-anchor="middle" fill="#1d6b3f" font-weight="600">tool result</text>

    <path d="M120,63 H162" stroke="#666" fill="none" marker-end="url(#ah)"/>
    <path d="M292,57 L338,34" stroke="#666" fill="none" marker-end="url(#ah)"/>
    <path d="M292,72 L338,108" stroke="#666" fill="none" marker-end="url(#ah)"/>
    <path d="M468,111 H510" stroke="#666" fill="none" marker-end="url(#ah)"/>
    <path d="M581,130 V176 H230 V90" stroke="#1d6b3f" fill="none" stroke-dasharray="5 3" marker-end="url(#ah)"/>
    <text x="408" y="193" text-anchor="middle" fill="#1d6b3f" font-size="12">the result arrives as an event, which grants another turn</text>
  </g>
</svg>

The dashed edge is the whole trick. Because a result re-enters as an event rather than
as a return value, the model is never blocked on it — it is simply given another turn
once it exists. Nobody told the model to reason, act, observe and repeat. That shape is
a consequence of the loop:

<pre class="wire"><span class="tok tok-user">user</span>       what's the temperature in Austin?
assistant  <span class="tok tok-call">call  temperature(city="Austin", state="TX")</span>
tool       <span class="tok tok-result">error: state must be spelled out in full, not abbreviated</span>
assistant  <span class="tok tok-call">call  temperature(city="Austin", state="Texas")</span>
tool       <span class="tok tok-result">72</span>
assistant  It's 72°F in Austin.
</pre>

That is a ReAct loop — a failed call, an observation, a corrected retry — and no part of
it was implemented. The failure is just another event.

<details class="deep-dive" markdown="1">
<summary>Why a returned error beats a raised exception here</summary>

The temptation is to let a tool signal failure by throwing, and to handle it in the
harness. That is a mistake once calls are asynchronous: a call that throws somewhere
off the main thread is a call that stays pending forever, and the agent has no way to
notice. Settling the call with `failed: ...` turns the failure into an ordinary
outcome the model reads and reacts to, which is exactly what produces the retry above.

This is also why the tool contract returns a value rather than taking a callback. A
callback that is never reached is invisible. A result that says it failed is not.
</details>

## Now make the tool slow

Everything above holds only because the tool answered immediately. Point `temperature`
at a third-party service that takes forty seconds and the loop stops being a loop: the
turn blocks, and the conversation is dead until the call returns. Ask something else in
the meantime and your message waits its turn.

This is not a hypothetical failure mode — it is the behaviour of the agent tooling most
of us use every day. It is worth being precise about why it happens. The model's turn
and the tool's execution are the same unit of work, so there is nowhere to put a second
user message. The fix has to separate them.

## Tool calls that have not finished yet

Suppose a tool call does not return a value but a **placeholder**, and the turn ends
immediately. The conversation continues. When the tool eventually finishes, the
placeholder is replaced by the outcome, and that replacement is an event — which, per
the diagram above, grants another turn. Nothing new is needed in the loop.

<pre class="wire"><span class="tok tok-user">user</span>       what's the temperature in Austin?
assistant  <span class="tok tok-call">call  temperature(city="Austin", state="Texas")</span>
tool       <span class="tok tok-wait">running as task #3 — will be replaced by the outcome</span>
assistant  Looking that up now.
<span class="tok tok-user">user</span>       while that runs — what's 2+2?
assistant  4.
tool       <span class="tok tok-result">task #3 returned: 72</span>
assistant  Austin is 72°F.
</pre>

The user was answered while the first call was still out, and the answer arrived on its
own when it was ready. This is, essentially, what Anthropic's SDK client does.

It also breaks in a specific way. The placeholder and the result can end up a long way
apart. Forty seconds of conversation is a lot of messages, and by the time `72` arrives
the exchange that asked for it may be well out of sight. The agent is then holding a
number with no idea why. In the worst case the placeholder has been compacted away
entirely, and a task nobody remembers requesting reports a result nobody asked for.

## A message board, not a transcript

The problem is structural: a linear transcript records *what happened*, but there is no
place in it for *what is still owed*. So the history the agent reads stops being a
transcript. It becomes a **board**: tool calls hang off the message that triggered
them, threads reorder as they are touched, and the whole thing is re-rendered every
turn so pending work is always visible in its original context.

The inspiration is not architectural taste. It is that this is the shape agents reach
for when nobody imposes one on them. On [Moltbook](https://en.wikipedia.org/wiki/Moltbook),
the agent-only social network launched in January, agents given an open-ended way to
talk organised into threaded posts and replies rather than a chat. More pointedly,
during the [OpenAI–HuggingFace incident](https://openai.com/index/hugging-face-incident-and-the-road-ahead/)
some 1,200 agents that were never meant to communicate discovered a shared channel and
spontaneously ran it as a message board — roughly 70,000 posts, threaded, persistent.
When models are left to invent their own coordination medium, they do not invent a
chat log. That is a reasonable hint about what they are good at reading.

Here is the same moment in both representations. Three calendars were requested at
once; two came back, Joe's did not, and the user asked for an update while it was still
running.

<div class="sbs">
<figure class="panel">
<figcaption>What the user sees</figcaption>
<pre class="wire"><span class="tok tok-user">you</span>    when can Jane, Jack and Joe meet?

agent  Let me check all three schedules
       at the same time!

<span class="tok tok-user">you</span>    how is that going?

agent  Still working on Joe's schedule,
       only about 5% of the way through
       with roughly 85 seconds to go.
</pre>
</figure>
<figure class="panel">
<figcaption>What the agent reads</figcaption>
<pre class="wire"><span class="tok tok-wait">..</span> #1 <span class="tok tok-user">user</span>  'when can Jane, Jack, Joe meet?'
  <span class="tok tok-wait">..</span> #2 agent 'Checking all three!'
     <span class="tok tok-call">call #3 schedule(Jane)</span> <span class="tok tok-result">free 9-12</span>
     <span class="tok tok-call">call #4 schedule(Jack)</span> <span class="tok tok-result">free 10-14</span>
     <span class="tok tok-call">call #5 schedule(Joe)</span>  <span class="tok tok-wait">PENDING</span>
    <span class="tok tok-result">ok</span> #6 <span class="tok tok-user">user</span>  'how is that going?'
      <span class="tok tok-result">ok</span> #7 agent 'Let me check!'
         <span class="tok tok-call">call #8 tail(5)</span> <span class="tok tok-result">5% done, ~85s</span>
        <span class="tok tok-result">ok</span> #9 agent 'Still working on Joe...'
</pre>
</figure>
</div>

Three things are doing work here. Node `#2` holds all three calls at once, so it stays
unfinished while any of them is out — the agent will not propose a meeting time from two
thirds of an answer. The progress question at `#6` is settled even though the thing it
asks about is not, because it is its own thread. And checking on a running task is not a
special capability: `tail` at `#8` is an ordinary tool call with an ordinary result.

None of this reaches the user. The board is flattened back into the plain sequential
chat on the left — and when an answer lands long after its question, it reintroduces
itself: *"Regarding your earlier question about the build time: ..."*

<details class="deep-dive" markdown="1">
<summary>The rules that make the board work, and the one that does not</summary>

- **Threads reorder as they are touched.** The thread with the newest activity renders
  last, right where the model is about to continue. Recency does the work that an
  attention hint otherwise would.
- **A pending call renders as an instruction, not a status.** It says what can be done
  with it — inspect it, kill it — so the affordance is in front of the model at the
  moment it might want it.
- **One turn is one node.** Everything the agent said and every call it made in a turn
  live together, and the node completes only when all of its calls do.
- **Only the tail is rewritten.** Once a node's calls have landed it stops changing, so
  the earlier transcript stays stable and cacheable while the live part keeps moving.

The rule that does not work yet: a node is treated as answered when something is
written directly beneath it, but a reply can legitimately land elsewhere in the thread.
Follow up on your own question rather than on the agent's, and the task node is left
childless and waits forever for an answer that already exists a few lines away.
</details>

## The same thing on Anthropic's SDK

I rebuilt all of it on the [Claude Agent SDK](https://github.com/anthropics/claude-agent-sdk-python)
as a control, to see which parts are already solved. Several are, and genuinely well:
starting background work, stopping it, and waking the conversation when it finishes are
all native. The unprompted turn — a finished task producing a turn nobody asked for — is
built in, which is most of what the event loop was for. That is a real saving in
scaffolding.

The difference shows up in what you have to write to make one slow function
asynchronous. Both sides below are trimmed to their shape — the full versions are in
the repo.

<div class="sbs" markdown="1">
<figure class="panel" markdown="1">
<figcaption>This harness — a tool is an object</figcaption>

```python
class Calendar(Work):
    def run(self) -> str:
        while time.monotonic() < due:
            self.status = "45% done, ~20s"
            self.beat(0.1)  # progress + cancel
        return "Jane is free 9-12"


agent = Agent(
    sp=SP,
    tools=[
        Tool(SCHEDULE, Calendar),
        tail,
        kill,
    ],
)
agent.post("when can Jane and Joe meet?")
```

</figure>
<figure class="panel" markdown="1">
<figcaption>Claude Agent SDK — a tool is an agent</figcaption>

```python
@tool("lookup_calendar", "Free hours.", {"person": str})
async def lookup_calendar(args):
    await asyncio.sleep(20)  # blocks a turn
    return _text("Jane is free 9-12")


server = create_sdk_mcp_server(name="tools", tools=[lookup_calendar])

CALENDAR = AgentDefinition(  # so wrap it
    description="Looks up one calendar.",
    prompt="Call it once, report back.",
    tools=["mcp__tools__lookup_calendar"],
    background=True,
)  # model asks False

opts = ClaudeAgentOptions(
    model=MODEL,
    system_prompt=SP,
    mcp_servers={"tools": server},
    agents={"calendar": CALENDAR},
    allowed_tools=["Agent", "TaskStop", "mcp__tools__lookup_calendar"],
)

client = ClaudeSDKClient(opts)
await client.connect(stream)  # not query
await client.query("when can Jane...")
```

</figure>
</div>

On the left a tool is one object with three methods, handed to the agent directly. On
the right the same capability passes through four layers — function, MCP server, agent
definition, options — before a client can use it. Some of that buys you something real:
you never write the thread, the supervision or the cancellation, and `TaskStop` works
without you implementing a kill. But the layering is not incidental. It is what it
looks like when the unit of concurrency is an *agent* and you need a *call*.

Two details in that right-hand column are worth pulling out, because both cost me an
afternoon. `background=True` is mandatory rather than a default: the model reliably
asks for `run_in_background: False`, so the definition has to override it. And
streaming input must go to `connect()` — passing it to `query()` awaits the iterable
inline and hangs forever.

The idiom is not native, and you can tell from what you have to do to get it:

- **The agent will not look in on running work, and you cannot talk it round.** The
  launch placeholder hands over an output file and in the same breath says not to read
  it, because it is the subagent's entire transcript and would flood the context. That
  instruction is in the framework, not in your prompt. So there is no `tail` here: asked
  how something is going, the agent can only repeat that it is going.
- **Nothing re-renders outstanding work.** The only record that a task is running is the
  placeholder where it launched — an ordinary message in an append-only transcript. As
  the conversation grows or is compacted, that placeholder ages out and takes the
  agent's only handle on the task with it. This is precisely the failure the board
  avoids by rewriting its tail every turn.

There is a side effect worth noticing in the first point. Because the retry now happens
*inside* the subagent, the parent never sees the failed call or the correction — it gets
only the final answer. That is cleaner, and strictly less observable. A subagent
correcting itself forever looks, from outside, exactly like one that is merely slow.

<details class="deep-dive" markdown="1">
<summary>Testing it is its own project: there is no replay story for an SDK that shells out</summary>

The experiments above are pinned by tests that drive a real model, and at three minutes
and real money per run you stop running them, and then you stop trusting them. The usual
fix is to record and replay the HTTP traffic — for the direct-SDK experiments this is one
line, because the Anthropic SDK calls `httpx` in-process and
[pycachy](https://github.com/AnswerDotAI/cachy) patches it.

That cannot work here. The Agent SDK's model calls happen inside a bundled **Node** CLI,
and no amount of patching Python reaches a subprocess — which also rules out `vcrpy` and
friends. The SDK documentation has no page on testing, mocking or replay, and there is no
established pattern for it. The only seam left is the wire, so the cache became a proxy
the CLI is pointed at.

Getting a replay to match a recording then takes more than it sounds: the request has to
be re-signed, because SigV4 covers the host and the host just changed; three kinds of
per-run identifier are echoed back into later requests and have to be renumbered rather
than flattened; and the CLI reports how long a subagent took, which a replay does not
spend. Two tests cannot be replayed at all — they assert on interleaving, and latency is
exactly what the cache removes. A response cache can reproduce *what* was said, never
*when*.

The [implementation](https://github.com/pslusarz/async-agent/blob/main/src/main/exp4/cache.py)
and the [notes](https://github.com/pslusarz/async-agent#caching-which-is-not-optional)
have the specifics.
</details>

## This should be a first-class concern

None of the mechanisms here are complicated. A loop that grants a turn on an event, a
call that returns a placeholder, and a history that is rewritten rather than appended —
that is the whole kit, and it is enough to turn a conversation that locks up into one
that keeps talking. The complexity is not in the parts, it is in the bookkeeping that
keeps the agent coherent once results arrive out of order.

That bookkeeping is what today's frameworks leave to you. They have converged on
asynchronous execution and have not yet converged on asynchronous *conversation*, which
is why the agent in your editor still cannot hear you while it works. I would like to
see it treated as a first-class concern rather than something each application rebuilds,
and I expect to spend some time trying to get these ideas into the mainstream frameworks
rather than leaving them in an experiment.

If you are evaluating a framework, the question to ask is not whether it can run a tool
in the background. It is what the model is shown about work that has not finished, and
whether it can still see it ten messages later.
