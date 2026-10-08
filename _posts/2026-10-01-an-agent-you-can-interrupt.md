---
title: "Towards a responsive agentic behavior"
date: 2026-10-01
description: "Keeping an LLM agent responsive while its tools are still running: a message board instead of a transcript, progress probes, timeouts and cancellation, compared against Anthropic's Claude Agent SDK and LangChain's async subagents."
image: /docs/assets/2026-10-01-an-agent-you-can-interrupt/social-card.png
---

**Summary:** How do we go from a turn-based chat to an agent that can manage multiple long-running tool calls while conversing with the user? This is the problem we are tackling here. It is accomplished via an event loop and a dynamically rewritten message board data structure, and I provide an implementation app so the reader can get a feel for what this interaction is like.

<video controls autoplay loop muted playsinline
       poster="/articles/docs/assets/2026-10-01-an-agent-you-can-interrupt/demo-kill-and-retry.png"
       style="max-width:100%;height:auto;display:block;margin:1.5rem 0;border:1px solid #d0d7de;border-radius:.5rem"
       aria-label="The demo playing. A build lookup finishes as a green dot. A second tool reports progress and then stops dead; the harness notices it is overdue, and the agent tails it, kills it — the dot turns red — and starts a fresh call that completes. The panel on the right shows the board the model reads.">
  <source src="/articles/docs/assets/2026-10-01-an-agent-you-can-interrupt/demo-kill-and-retry.mp4" type="video/mp4">
  <img src="/articles/docs/assets/2026-10-01-an-agent-you-can-interrupt/demo-kill-and-retry.png"
       alt="The demo mid-conversation: a finished build lookup as a green dot, a killed test lookup as a red one, and a fresh call still spinning.">
</video>

> **See it first:** [async-agent-demo-production.up.railway.app](https://async-agent-demo-production.up.railway.app/)
> — a short walkthrough of the harness in the browser. Ask about the weather in three
> cities, then interrupt it while the lookups are still running. It takes a minute, and
> the rest of this will make more sense once you have watched a conversation carry on
> over the top of work that has not finished.

**Motivation:** Historically, large language models were document completion machines. In order to make them usable, they were then trained on chats as the documents to complete, so they are conditioned to respond in turns. The problem is that chat recordings are not how users interact — there are tool calls and multiple things happening all at once, and chat is just a flat unrolling of a more complex interaction. Unfortunately, interaction design is not keeping up with LLM capabilities, and most agentic frameworks still work off a chat as their underlying data structure, leading to a bad user experience down the pipe. As a typical example, the current iteration of GitHub Copilot has the concept of "steering", but it will still get blocked by a long-running tool. This post, and the associated project, are an exploration of an alternate data and interaction model which allows the LLM to handle interaction with the user in a much more responsive manner.

**Audience:** If you are writing an agentic app, you should probably get familiar with the issues described here. I initially ran into this when writing an agentic app which did the work in the background, but with user occasionally connecting to it, asking for progress update and helping it get unstuck. This is probably too long for any human to read to the end in this day and age, so I suggest you point your agent at this article and ask it questions you want answered. While I don't necessarily recommend using this code as-is, current harnesses do not provide satisfying structures either (comparisons with ClaudeAgentSDK and LangChain's async subagents are included).

**Contents**

- [Main findings](#main-findings)
- [The loop is enough](#the-loop-is-enough)
- [Now make the tool slow](#now-make-the-tool-slow)
- [Tool calls that have not finished yet](#tool-calls-that-have-not-finished-yet)
- [A message board, not a transcript](#a-message-board-not-a-transcript)
- [Knowing when to give up](#knowing-when-to-give-up)
- [Putting it all together in a responsive application](#putting-it-all-together-in-a-responsive-application)
  - [The UI gets notified via server sent events](#the-ui-gets-notified-via-server-sent-events)
  - [Teaching it to stop narrating](#teaching-it-to-stop-narrating)
  - [Failures, and the retries behind them](#failures-and-the-retries-behind-them)
  - [What the board holds, and what the user sees](#what-the-board-holds-and-what-the-user-sees)
- [The same thing on Anthropic's SDK](#the-same-thing-on-anthropics-sdk)
- [The same thing on LangChain's async subagents](#the-same-thing-on-langchains-async-subagents)
- [Final remarks](#final-remarks)


## Main findings

- **An event loop is sufficient to produce ReAct.** A ReAct agent calls tools in response to user prompt, observes tool results, and tries to answer the initial user question, sometimes iterating over a number of tool calls before addressing the user. It was interesting to see that this behavior is emergent, once an event loop for each tool call result is introduced. 
- **A dynamic message board can replace chat transcript as the underlying data structure.** Once results can arrive late,
  a linear history cannot say what is still pending explicitly. Restructuring the history the agent
  reads into a **message board** with latest threads prominently at the end accomplishes that better. 
- **All tool calls are asynchronous**
  This is the sensible default, and it does not complicate the underlying harness, rather it simplifies things.
- **All tools provide interface for progress update and kill** The progress update is modeled on the tail(num_lines) command. These are the only two tools that need to return immediately, and they get handled in a special way by the harness, allowing the agent to respond on the same turn. 
- **Anthropic's SDK gets the mechanics and misses the bookkeeping.** First thing to recognize with Anthropic SDK is that tool calls need to be wrapped as (sub-)agents, because all regular tool calls are synchronous. The framework explicitly prevents agent from polling for progress from these subagents.
- **LangChain's async subagents can be made to behave, but only by adding the missing parts.** Out of the box the supervisor is never told that a task has finished. Two small tools give it progress and a wake-up timer, and then it answers unprompted. That is the same idea as here, built from more parts and paid for in model calls and latency.
- **Reference app included** - a fully responsive app is implemented, surfacing pending tool calls to both the user and agent. There is an interactive demo deployed.

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
place in it for *what is still pending*. I think this implies a different data structure, and in this investigation we explore one such structure - a dynamically rewritten message board. The tool calls hang off the message that triggered
them, threads reorder as they are touched, and the whole thing is re-rendered every
turn so pending work is always visible in its original context and presented towards the end of the conversation, where agent is paying attention.

When crafting tools and data structures for agents, I try to use techniques agents are already good at. These are largely emergent properties that come from LLM's foundational training, many of them discovered by practitioners that work with agents day to day (ie SudoLang, cli tool use, bash shell). The inspiration for this data structure comes from two widely publicized social behaviors. [Moltbook](https://en.wikipedia.org/wiki/Moltbook),
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

Ordinary to the model, at least. To the harness `tail` is the other kind of tool, and
the difference is only about *when it answers*. A **background** tool starts work that
outlives the turn that called it and hands back a task id. A **synchronous** tool —
`tail`, `kill` — answers in the same breath, because it acts on a task rather than
becoming one. That means a turn cannot end on a synchronous call: the agent asked how
something was going, and a turn that closed before it read the reply would throw the
answer away. So the loop goes round again immediately with the result in hand.

I decided to introduce these special synchronous tools for efficiency, even though they do complicate the underlying harness. There is a tradeoff here that may need to be examined in the future, but executing these tools immediately and presenting the results to the agent simplifies the asynchronous cases the tests have to handle (ie imagine a kill command followed by tool result events arriving at the same time and needing reconciliation).

None of this reaches the user. The board is flattened back into the plain sequential
chat on the left — and when an answer lands long after its question, it reintroduces
itself: *"Regarding your earlier question about the build time: ..."*

<details class="deep-dive" markdown="1">
<summary>The rules that make the board work</summary>

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

## Knowing when to give up

One case is still open: the tool that never finishes. Someone has to decide to stop it,
and that decision splits between the model and the harness. Only the harness knows that time has passed — the
model has no clock and cannot notice elapsed time between turns. But only the model
knows whether the thing it is waiting for is worth waiting for. Rather than introduce a heartbeat event at this point, we ask the model to include a timeout parameter with each tool call. If the tool does not complete by the time timeout is reached, timeout gets processed just like another event, encouraging the model to inspect the tool progress and kill or restart it.

The instruction to consider killing the task lives on the event and nowhere else: not
in the system prompt, not in any tool description. Only the harness knows the clock ran
out, so only the harness mentions it, and only when it is true.

<pre class="wire"><span class="tok tok-wait">..</span> #1 <span class="tok tok-user">user</span>  'temperature in Warsaw, MO?'
  <span class="tok tok-wait">..</span> #2 agent 'Checking that right now!'
     <span class="tok tok-call">call #3 temperature(Warsaw)</span> <span class="tok tok-wait">KILLED</span>
    <span class="tok tok-result">ok</span> #4 agent 'Let me see how that is coming along.'  <span class="tok tok-wait">&lt;- overdue</span>
       <span class="tok tok-call">call #5 tail(3)</span> <span class="tok tok-result">almost done...</span>
      <span class="tok tok-result">ok</span> #6 agent "It's almost there, just about finished!"
         <span class="tok tok-call">call #7 tail(3)</span> <span class="tok tok-result">almost done... (nothing has moved)</span>
        <span class="tok tok-result">ok</span> #8 agent "I'll wait — I'll be told when it moves."
    <span class="tok tok-result">ok</span> #9 agent 'Let me see how it is doing!'           <span class="tok tok-wait">&lt;- overdue</span>
       <span class="tok tok-call">call #10 tail(3)</span> <span class="tok tok-wait">I'm stuck and cannot make progress.</span>
      <span class="tok tok-result">ok</span> #11 agent 'That has stalled. Stopping it.'
         <span class="tok tok-call">call #12 kill(3)</span> <span class="tok tok-wait">killed — out of retries, ask instead</span>
          <span class="tok tok-result">ok</span> #13 agent <span class="tok tok-call">call #14 new_thread(...)</span>
<span class="tok tok-result">ok</span> #15 agent 'The tool keeps stalling. How would you like to proceed?'
</pre>

The agent was asked twice and answered differently each time. At `#4` the task said it
was almost done, so it left it alone — which re-arms the timer, so the question comes
back. At `#9` the same task said it was stuck, and it killed it.

This is the point at which the board earns its keep. `#4` and `#9` are **siblings**:
two separate visits to the same task, ten seconds apart, each hanging off the call it
is about rather than being appended to the end of a conversation. And `#15` is a **new
root**, because a thread that has run out of things to try is a bad place to ask a
question — the agent opened a fresh one to ask it. This is generally transparent to the user, the agent responses are clipped, unless the agent decides to start a new thread and notify the user of issues. One such issue is when a terminal number of retries has been reached. The harness keeps track of the retries (this is made easier because related retries are on the same board thread), and instructs the agent to consult with the user after the allowed number of retries have run out.

<details class="deep-dive" markdown="1">
<summary>Three things I got wrong on the way here</summary>

**Budgets for the thing, not the branch.** Killing a task spends from a retry budget,
and my first version counted kills per conversation thread. That is wrong as soon as
the agent fires three lookups at once: one failure drains the allowance for all of
them. The budget is now counted along the *branch* that led to the call — ancestors
only — so calls made side by side never spend each other's retries. The tree already
had the information; I just was not reading it.

**Forbidding is weaker than informing.** Given every tool for the whole turn, the agent
polled: it called `tail` three times in a row, using it as a sleep, burning eleven
seconds inside a single turn — and since the loop is one thread, that stalls the whole
conversation. Writing "looking twice in the same breath tells you nothing" into the tool
description did not stop it. What stopped it was giving it the fact instead: `tail` now
says when nothing has moved since the last look, and promises it will be told when
something does. The agent replied *"I'll keep waiting — no need to keep checking since
I'll be notified as soon as the result comes in"* and ended its turn. The promise is
kept by the re-armed timer.

**But a true statement still has to win on position.** I only found the limit of that
when I ran the whole thing on a smaller model. Out of retries, `kill` returns a result
that says to open a new thread and ask the user how to proceed. Sonnet does. Haiku read
the same sentence and answered in prose instead, in the thread it had just been told was
dead — because the harness then appended its own fallback line, `Respond to [#12].`, and
that was the last thing in the prompt. The instruction I meant sat seven levels deep
inside a bracketed tool result; the one that contradicted it was flush left at the end.
The stronger model reconciled them. The smaller one obeyed the nearer one.

The fix was not a better instruction, and it was certainly not an example. It was
noticing that the harness was emitting a contradiction at all — it already knew the
retries were gone, and generated the fallback anyway.
</details>

## Putting it all together in a responsive application

Everything so far is about what the *agent* reads. However, the point of this experiment is for a better user experience, and so I searched for some options to surface pending tool calls to the user.

So the last experiment is the same harness with the pending work displayed to the user. Every call the
agent starts appears beside the turn that started it — a spinner carrying the tool's
name and arguments while it runs, shrinking to a green dot when it returns and a red one
when it fails or is killed. I think in the real application we would consider giving the user an easy way to peek at the progress and kill the tools from within the UI, but here in the reference app I kept it deliberately simple and chose to skip these features.

![Three schedule lookups and a temperature lookup in flight at once. Jane's schedule is still spinning, Jack's and Joe's have shrunk to green dots, and the temperature question asked in the middle has already been answered.](/articles/docs/assets/2026-10-01-an-agent-you-can-interrupt/responsive-app.png)

Three calendars and a temperature are in flight at once. Jack and Joe have come back,
Jane has not, and the question asked in the middle has already been answered.

### The UI gets notified via server sent events

The page now holds one **server-sent events** stream (traffic goes one way, so a
websocket would buy nothing). Whenever the board changes, the server pushes a freshly
rendered transcript — so the same push carries both halves of the moment: the spinner
shrinking to a green dot, and the late answer landing beside it.

### Teaching it to stop narrating

Drawing the tasks makes most of the agent's commentary redundant, and there was a lot of
it. Every look began with "Let me take a peek at how that's coming along!" before
anything useful was said, and every expired timer produced a paragraph of percentages
that the spinner was displaying anyway.

The obvious fix — instruct the model to be quiet — is the wrong one, because those words
are not decoration. They are the agent's own record of why it did what it did, and the
next turn reads them back. What needed to change was not whether the agent speaks but
who it is speaking to.

So the harness decides, per message. A turn is triggered by one of three things — the
user, a result landing, or a timer — and each message the turn writes either looks at a
task, starts one, or is the turn's last word. Nine combinations, of which three are the
agent talking to the reader:

| the turn was triggered by | a look at a task | starting work | the last word |
|---|---|---|---|
| **the user** | board only | **shown** | **shown** |
| **a result landing** | board only | board only | **shown** |
| **a timer** | board only | board only | board only |

- **A look is never shown**, not even when the user asked for it, because the turn's
  last word already reports what the look found. Showing both is what produced the
  duplicate reply.
- **A timer-driven turn says nothing at all.** It still happens — the agent looks,
  decides, and records the decision — but the user sees a spinner still spinning, which
  is the same fact without the prose.
- **Work started after a result is silent**, because the agent is not answering yet. It
  is retrying.

Nothing is deleted. Everything suppressed still goes on the board; the agent's context
is exactly what it was. Only the projection into the chat is filtered.

### Failures, and the retries behind them

A call that is killed, or that settles as a failure, turns its dot red. That is the one
piece of task state the user gets without asking for it.

The retry behind it stays quiet, and the suppression is of words rather than of rows, so
a silent restart still puts a fresh spinner on the screen — you watch it try again
without being told that it is. What does break the silence is giving up: once the
retries are spent, the kill result tells the agent to open a new thread and ask, and the
message `new_thread` posts is shown, because putting a question to the user is not
bookkeeping.

### What the board holds, and what the user sees

This is the board behind the screenshot above, at that moment. The gutter marks what the
reader got.

<pre class="wire"><span class="tok tok-result">chat</span> #1 <span class="tok tok-user">user</span>  'when can Jane, Jack and Joe meet today?'
<span class="tok tok-result">chat</span>   #2 agent 'I'll look up all three schedules at the same time!'
          <span class="tok tok-call">call #3 schedule(Jane)</span> <span class="tok tok-wait">PENDING</span>
          <span class="tok tok-call">call #4 schedule(Jack)</span> <span class="tok tok-result">free 10-14</span>
          <span class="tok tok-call">call #5 schedule(Joe)</span>  <span class="tok tok-result">free 11-15</span>
<span class="tok tok-result">chat</span>     #6 <span class="tok tok-user">user</span>  'meanwhile, what's the temperature in Austin TX?'
<span class="tok tok-wait">····</span>       #7 agent <span class="tok tok-call">tail(3)</span> 7%  <span class="tok tok-call">tail(4)</span> 19%  <span class="tok tok-call">tail(5)</span> 21%
<span class="tok tok-result">chat</span>         #11 agent 'Kicked off Austin. Jane 7%, Jack 19%, Joe 21%.'
                 <span class="tok tok-call">call #12 temperature(Austin)</span> <span class="tok tok-result">72F</span>
<span class="tok tok-result">chat</span>           #25 agent 'Austin, TX is currently a comfortable 72°F!'
<span class="tok tok-wait">····</span>     #13 agent 'Let me check on Joe.'  <span class="tok tok-call">tail(5)</span> 79%
<span class="tok tok-wait">····</span>       #15 agent 'Joe is 79% done, about 3s to go.'
<span class="tok tok-wait">····</span>     #16 agent 'Let me check on Jack.' <span class="tok tok-call">tail(4)</span> 95%
<span class="tok tok-wait">····</span>       #18 agent 'Jack is nearly done at 95%.'
<span class="tok tok-wait">····</span>     #19 agent 'Let me check on Jane.' <span class="tok tok-call">tail(3)</span> 49%
<span class="tok tok-wait">····</span>       #21 agent 'Jane is halfway. Let me check the others too.'
                 <span class="tok tok-call">tail(4)</span> <span class="tok tok-result">finished</span>  <span class="tok tok-call">tail(5)</span> <span class="tok tok-result">finished</span>
<span class="tok tok-wait">····</span>         #24 agent 'Jack and Joe are both done. Still waiting on Jane.'
</pre>

Eleven agent messages; three of them reached the screen. The eight that did not are
every look at a task, and every word a timer prompted — including `#24`, where the agent
worked out that Jack and Joe overlap between 11 and 2 and kept it to itself, because it
still could not answer the question that was actually asked.

The three timer nudges are worth a second look. `#13`, `#16` and `#19` are **siblings**
hanging off `#2`, not a chain appended to the end of the conversation, so each one sits
with the calls it is about. And the entire Austin exchange — `#7`, `#11`, `#25` — hangs
off `#6`, which is why it could be answered and closed while the thread above it was
still open.

One leak is visible in `#11`. It is shown, because the user asked and the turn was
starting work, and the agent chose to pack the progress percentages into the same
message. The rule is about which turn is speaking, not about what it says, so narration
still gets through when the user's question happens to land next to running work. The
spinners make it redundant rather than wrong, but it is the seam where this approach
shows.

## The same thing on Anthropic's SDK

If I were to evaluate every harness and framework on this behavior, we could write a whole book, and still not provide much value, since these frameworks are undergoing a fast evolution, and new ones pop up. Nevertheless value of going through exercise like that is that one has the mental tools to evaluate frameworks at a much deeper level than what their own documentation allows. And so as an exercise, here I evaluate current state of Anthropic API.

I rebuilt all of it on the [Claude Agent SDK](https://github.com/anthropics/claude-agent-sdk-python)
as a control — first the asynchronous conversation, then the timeouts on top of it — to
see which parts are already solved. Several are, and genuinely well: starting
background work, stopping it, and waking the conversation when it finishes are all
native. The unprompted turn — a finished task producing a turn nobody asked for — is
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

- **The model may not look in on running work — but your code may.** The launch
  placeholder hands over an output file and in the same breath says not to read it,
  because it is the subagent's entire transcript and would flood the context. That
  instruction is in the framework, not in your prompt. So there is no `tail` here:
  asked how something is going, the agent can only repeat that it is going.

  The prohibition binds the *model*, though, not the process. That output file is
  live-updating JSONL sitting on disk, and nothing stops the harness from reading it.
  Mine does: it pulls the last couple of events out and folds one line into the
  timeout nudge, so the model decides on evidence rather than on elapsed time alone.
  And if the harness can read it, so can anything else you are building — a UI is free
  to render a progress indicator for work the agent itself is forbidden to discuss.
  The information is not missing. It is just not routed to the model.
- **Nothing re-renders outstanding work.** The only record that a task is running is the
  placeholder where it launched — an ordinary message in an append-only transcript. As
  the conversation grows or is compacted, that placeholder ages out and takes the
  agent's only handle on the task with it. This is precisely the failure the board
  avoids by rewriting its tail every turn.

There is a side effect worth noticing in the first point. Because the retry now happens
*inside* the subagent, the parent never sees the failed call or the correction — it gets
only the final answer. That is cleaner, and strictly less observable. A subagent
correcting itself forever looks, from outside, exactly like one that is merely slow.

The timeouts, on the other hand, port over almost intact — because the clock was never
part of the API to begin with. `TaskStarted` carries a task id and the session can be
spoken to at any moment, which is the whole of what a timer needs. Given the nudge, the
agent behaves just as it does on the board: told once that a task had been running
twenty seconds with an empty transcript it judged that "within normal startup range" and
left it alone, and told again twenty seconds later it said it was "still empty after two
check-ins" and called `TaskStop`. Nobody typed anything after the opening question.

What does not port is the model's own judgement about how long to wait. The
backgrounding schema belongs to the framework, so there is nowhere to put an expected
interval, and the timeout becomes your policy instead of the model's estimate. And
speaking to the session is the only way in, so the nudge arrives as a *user* turn: it
has to be labelled as machinery, or the agent thanks you for checking in on it.

<details class="deep-dive" markdown="1">
<summary>Anthropic API provides no way to cache LLM responses easily</summary>

This is my annoyance with many harnesses. We need to write unit tests to verify the application code and the LLM work together harmoniously, but we do not need to awaken a costly and slow LLM each time we run the same scenario. Somehow this is not on any framework priority list.

The experiments above are pinned by tests that drive a real model, and at three minutes
and real money per run you stop running them, and then you stop trusting them. The usual
fix is to record and replay the HTTP traffic — for the direct-SDK experiments this is one
line, because the Anthropic SDK calls `httpx` in-process and
[pycachy](https://github.com/AnswerDotAI/cachy) patches it.

That cannot work with Anthropic's harness. The Agent SDK's model calls happen inside a bundled **Node** CLI,
and no amount of patching Python reaches a subprocess — which also rules out `vcrpy` and
friends. The SDK documentation has no page on testing, mocking or replay, and there is no
established pattern for it. The only seam left is the wire, so the cache became a proxy
the CLI is pointed at.

Getting a replay to match a recording then takes more than it sounds: the request has to
be re-signed, because SigV4 covers the host and the host just changed; three kinds of
per-run identifier are echoed back into later requests and have to be renumbered rather
than flattened; and the CLI reports how long a subagent took, which a replay does not
spend. Some tests cannot be replayed at all — they turn on interleaving or on a
wall-clock timer, and latency is exactly what the cache removes. A response cache can
reproduce *what* was said, never *when*.

The [implementation](https://github.com/pslusarz/async-agent/blob/main/src/main/exp4/cache.py)
and the [notes](https://github.com/pslusarz/async-agent#caching-which-is-not-optional)
have the specifics.
</details>

## The same thing on LangChain's async subagents

LangChain recently shipped [async subagents](https://docs.langchain.com/oss/python/deepagents/async-subagents)
for its deep agents. Of the mainstream frameworks, it comes closest to the behavior described here.
A supervisor launches a subagent, gets a task id back immediately, and is able to continue chatting with the user.
Five tools come with it: start, check, update, cancel and list. The task
bookkeeping lives in a dedicated `async_tasks` state channel, kept out of the message
history so that summarisation cannot drop it. That is the same problem the board solves
by re-ordering threads. Seeing it addressed head-on in a framework is encouraging.

The calendar scenario was implemented with LangChain native constructs. There are
two deep agents, a supervisor and a researcher, registered with a local `langgraph dev`
server. A subagent here is a thread and a run on an
[Agent Protocol](https://github.com/langchain-ai/agent-protocol) server, so it lives
in a different process from the agent that started it. The chat app is a plain SDK
client that posts each user message as another run on the supervisor's thread.

**As shipped, it is pull-only.** The UI, which
reads the subagent's thread directly, showed the researcher finished, but the supervisor was never notified and never said anything to the user. The supervisor
still held the task as `running`. It found out only when I typed "any
update?" and it called `check_async_task`. That tool returns a status, plus the final
message once the run is over, so there is nothing to look at while the work is going.
The documentation's troubleshooting section even names the failure you get when the
model compensates: *the supervisor calls check in a loop right after launching*. The
recommended fix is a sentence in the system prompt.

**It can be taught.** Two tools, about sixty lines, built from nothing but LangGraph SDK
calls ([`watch.py`](https://github.com/pslusarz/async-agent/blob/main/src/main/exp9/watch.py)):

```python
@tool
async def check_back_in(seconds: int, note: str, runtime: ToolRuntime) -> str:
    """Arrange to be woken after `seconds` to look at a running task again."""
    thread = runtime.config["configurable"]["thread_id"]
    await disarm(cli, thread)  # one wake-up armed at a time
    await cli.runs.create(
        thread,
        "supervisor",
        input={"messages": [{"role": "user", "content": f"[automatic] {note}"}]},
        multitask_strategy="enqueue",
        after_seconds=seconds,
    )
```

`peek_async_task` is the `tail` equivalent. It reads the subagent's own thread state: its
latest words, and which tools it has started and finished. The data was always there;
the supervisor just had no tool for asking. `check_back_in` is the timer. It schedules
a run on the supervisor's *own* thread for later, so the server wakes it up with a note
it wrote to itself. With both in place, the supervisor starts the research, says so,
and about thirty seconds later comes back with the meeting time, without another word
from the user.

So it can be done, but it is the equivalent of a sleeping, polling thread - structurally not the right mechanism. It spends tokens on every wake-up, and the answer waits for the next one. Also, note how much extra code had to be written just to surface the fundamentals of user experience.

The code for
this experiment is in [`exp9`](https://github.com/pslusarz/async-agent/tree/main/src/main/exp9).

## Final remarks

While LLM capabilities progress at unprecedented rate, the harnesses that turn these LLMs into actual useful applications are lagging behind. My frustration as a user and agentic application author led to this deep dive. None of the mechanisms here are complicated. A loop that grants a turn on an event, a
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
in the background. It is what the model is shown about work that has not finished,
whether it can still see it ten messages later, and who gets to decide when to stop
waiting.
