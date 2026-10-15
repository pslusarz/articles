---
title: "Two-way conversation between an MCP App and its agent"
date: 2026-10-15
description: "MCP Apps can post into the chat but never hear back. Here is how two shared tools, a file and a watcher give an app a real conversation with the agent."
image: /docs/assets/2026-10-15-mcp-app-agent-two-way/social-card.png
---

<video controls muted playsinline preload="metadata"
       poster="/articles/docs/assets/2026-10-08-lustereczko-five-months-on/declaration-reader.png"
       style="max-width:100%;height:auto;display:block;margin:1.5rem 0;border:1px solid #d0d7de;border-radius:.5rem"
       aria-label="Reading the Declaration of Independence inside GitHub Copilot chat in VS Code. The user highlights a passage, asks why it is capitalized so strangely, and the agent's answer appears in a comment card next to the passage. The user then highlights unalienable Rights, asks whether it is a misspelling, and that answer appears in its own card.">
  <source src="/articles/docs/assets/2026-10-08-lustereczko-five-months-on/declaration-reader.mp4" type="video/mp4">
  <img src="/articles/docs/assets/2026-10-08-lustereczko-five-months-on/declaration-reader.png"
       alt="Two highlighted passages of the Declaration of Independence, each with a comment card in the margin holding the user's question and the agent's answer.">
</video>

*Questions about a passage, answered next to it, without anyone typing in the chat. The agent is GitHub Copilot in VS Code; the video is real time, not sped up.*

The MCP protocol lets a server call back into the host's LLM through [sampling](https://modelcontextprotocol.io/specification/2026-07-28/client/sampling), but hardly anyone supports it: according to [this compatibility matrix](https://caniuse.dev/capabilities/sampling), not even Anthropic, the creators of MCP, support it in Claude Desktop or Claude Code, and the 2026-07-28 version of the protocol [deprecates it](https://github.com/modelcontextprotocol/modelcontextprotocol/pull/2577). Sampling would not give the server the host's chat history either. The [MCP Apps extension](https://github.com/modelcontextprotocol/ext-apps/blob/main/specification/2026-01-26/apps.mdx) gives the app two ways to talk to the chat: [`updateModelContext()`](https://apps.extensions.modelcontextprotocol.io/api/classes/app.App.html#updatemodelcontext) and [`sendMessage()`](https://apps.extensions.modelcontextprotocol.io/api/classes/app.App.html#sendmessage). Neither one gives the agent a turn. The context update waits for the user's next message, and in the hosts I have tried, the message lands in the user's input box and waits for them to press send. Even then, the agent's answer goes into the chat, not back into the app.

Given those limits, is an MCP App stuck with a fixed set of behaviors? Not if you think outside the box for a moment. In this post I show how the app in the video above holds a seamless conversation with the agent. For a reference implementation, ask [lustereczko-mcp](https://github.com/pslusarz/lustereczko-mcp) to build you one; how lustereczko generates apps on the fly is a topic for another post.

Many tutorials jump right into implementation without explaining the key concepts, and we will not make that mistake here. The concepts are simple, and they are built right on top of existing MCP functionality.

## The idea

The one layer that both the agent and the app can reach is the MCP server's tools. The agent calls tools as it always does, through the host. The app can call the same server's tools with [`app.callServerTool()`](https://apps.extensions.modelcontextprotocol.io/api/classes/app.App.html#callservertool), which the host relays without involving the LLM. So if one side writes a message through a tool and the other side reads it through another tool, they can talk.

The app can poll for messages on a timer. The agent cannot: it only acts during a turn, and something has to give it one. For that, the agent sets up either a cron job or a watcher, a small background command that exits when the app has posted a message. The host notices the exit and wakes the agent.

There is one more wrinkle. A local MCP server runs over [stdio](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/stdio): the host launches it as a subprocess, restarts it when it dies, and owns its lifecycle. The spec calls the protocol stateless, and some hosts, Claude Desktop among them, run several copies of the server at once. The agent's call and the app's call may land in different processes, so nothing can live in a process's memory. The two sides need a named channel, and storage that every process shares.

That is the whole trick. Here it is in one picture, numbered in the order of one question and its answer:

<svg viewBox="0 0 840 500" role="img" aria-label="Data flow of a lustereczko channel. Two lanes run between the agent on the left and the app on the right. In the top lane the agent calls notify_ui, which appends to the agent-to-app queue file, and the app drains it with poll_ui_messages. In the bottom lane the app calls notify_agent, which appends to the app-to-agent queue file, and the agent drains it with poll_agent_messages. Whichever of several server processes handles a call, they all share the same two files. A watcher reads the app-to-agent file directly and exits when a message arrives, which wakes the agent." style="max-width:100%;height:auto;display:block;margin:1.5rem 0">
<defs>
<marker id="bf-user" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#1a4f8a"/></marker>
<marker id="bf-call" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#8a5a00"/></marker>
<marker id="bf-result" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#1d6b3f"/></marker>
<marker id="bf-wait" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#a8322d"/></marker>
<marker id="bf-grey" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#666"/></marker>
</defs>
<g font-family="-apple-system,BlinkMacSystemFont,Segoe UI,Helvetica,Arial,sans-serif">
<rect x="10" y="10" width="110" height="34" rx="17" fill="#e7effa" stroke="#1a4f8a"/>
<text x="65" y="32" text-anchor="middle" fill="#1a4f8a" font-size="13" font-weight="600">user</text>
<rect x="720" y="10" width="110" height="34" rx="17" fill="#e7effa" stroke="#1a4f8a"/>
<text x="775" y="32" text-anchor="middle" fill="#1a4f8a" font-size="13" font-weight="600">user</text>
<path d="M65,44 V96" stroke="#666" stroke-width="1.8" fill="none" marker-end="url(#bf-grey)"/>
<text x="75" y="75" text-anchor="start" fill="#666" font-size="11.5">“build me a reader”</text>
<path d="M775,44 V96" stroke="#1a4f8a" stroke-width="1.8" fill="none" marker-end="url(#bf-user)"/>
<text x="765" y="75" text-anchor="end" fill="#1a4f8a" font-size="11.5">highlights, asks</text>
<rect x="10" y="100" width="110" height="240" rx="6" fill="#f4f4f4" stroke="#666"/>
<text x="65" y="124" text-anchor="middle" fill="#333" font-size="13" font-weight="600">agent</text>
<text x="65" y="140" text-anchor="middle" fill="#666" font-size="11">LLM, in the chat</text>
<text x="65" y="314" text-anchor="middle" fill="#777" font-size="10.5">calls tools</text>
<text x="65" y="328" text-anchor="middle" fill="#777" font-size="10.5">through the host</text>
<rect x="720" y="100" width="110" height="240" rx="6" fill="#f4f4f4" stroke="#666"/>
<text x="775" y="124" text-anchor="middle" fill="#333" font-size="13" font-weight="600">app</text>
<text x="775" y="140" text-anchor="middle" fill="#666" font-size="11">iframe in the chat</text>
<text x="65" y="194" text-anchor="middle" fill="#8a5a00" font-size="11">answers</text>
<text x="65" y="262" text-anchor="middle" fill="#1a4f8a" font-size="11">questions</text>
<text x="775" y="194" text-anchor="middle" fill="#8a5a00" font-size="11">answers</text>
<text x="775" y="262" text-anchor="middle" fill="#1a4f8a" font-size="11">questions</text>
<text x="775" y="314" text-anchor="middle" fill="#777" font-size="10.5" font-family="SFMono-Regular,Menlo,Consolas,monospace">app.</text>
<text x="775" y="328" text-anchor="middle" fill="#777" font-size="10.5" font-family="SFMono-Regular,Menlo,Consolas,monospace">callServerTool</text>
<rect x="177" y="88" width="150" height="240" rx="6" fill="#fff" stroke="#ccc"/>
<rect x="171" y="94" width="150" height="240" rx="6" fill="#fff" stroke="#aaa"/>
<rect x="165" y="100" width="150" height="240" rx="6" fill="#fff" stroke="#666"/>
<text x="240" y="124" text-anchor="middle" fill="#333" font-size="13" font-weight="600">lustereczko</text>
<text x="240" y="140" text-anchor="middle" fill="#666" font-size="11">1 of N processes, stdio</text>
<text x="240" y="168" text-anchor="middle" fill="#8a5a00" font-size="12" font-weight="600" font-family="SFMono-Regular,Menlo,Consolas,monospace">notify_ui</text>
<text x="240" y="184" text-anchor="middle" fill="#777" font-size="10.5">appends</text>
<text x="240" y="273" text-anchor="middle" fill="#1a4f8a" font-size="11.5" font-weight="600" font-family="SFMono-Regular,Menlo,Consolas,monospace">poll_agent_messages</text>
<text x="240" y="289" text-anchor="middle" fill="#777" font-size="10.5">reads and clears</text>
<rect x="537" y="88" width="150" height="240" rx="6" fill="#fff" stroke="#ccc"/>
<rect x="531" y="94" width="150" height="240" rx="6" fill="#fff" stroke="#aaa"/>
<rect x="525" y="100" width="150" height="240" rx="6" fill="#fff" stroke="#666"/>
<text x="600" y="124" text-anchor="middle" fill="#333" font-size="13" font-weight="600">lustereczko</text>
<text x="600" y="140" text-anchor="middle" fill="#666" font-size="11">1 of N processes, stdio</text>
<text x="600" y="168" text-anchor="middle" fill="#8a5a00" font-size="12" font-weight="600" font-family="SFMono-Regular,Menlo,Consolas,monospace">poll_ui_messages</text>
<text x="600" y="184" text-anchor="middle" fill="#777" font-size="10.5">reads and clears</text>
<text x="600" y="273" text-anchor="middle" fill="#1a4f8a" font-size="11.5" font-weight="600" font-family="SFMono-Regular,Menlo,Consolas,monospace">notify_agent</text>
<text x="600" y="289" text-anchor="middle" fill="#777" font-size="10.5">appends</text>
<text x="420" y="104" text-anchor="middle" fill="#555" font-size="12.5" font-weight="700">durable storage</text>
<text x="420" y="120" text-anchor="middle" fill="#666" font-size="11" font-family="SFMono-Regular,Menlo,Consolas,monospace">logs/channels/</text>
<path d="M355,145 H473 L485,157 V200 H355 Z" fill="#fdf0d5" stroke="#8a5a00"/>
<path d="M473,145 V157 H485" fill="none" stroke="#8a5a00"/>
<text x="420" y="169" text-anchor="middle" fill="#8a5a00" font-size="11" font-weight="600" font-family="SFMono-Regular,Menlo,Consolas,monospace">ui_ch-….json</text>
<text x="420" y="186" text-anchor="middle" fill="#8a5a00" font-size="10.5">agent → app</text>
<path d="M355,250 H473 L485,262 V305 H355 Z" fill="#e7effa" stroke="#1a4f8a"/>
<path d="M473,250 V262 H485" fill="none" stroke="#1a4f8a"/>
<text x="420" y="274" text-anchor="middle" fill="#1a4f8a" font-size="11" font-weight="600" font-family="SFMono-Regular,Menlo,Consolas,monospace">agent_ch-….json</text>
<text x="420" y="291" text-anchor="middle" fill="#1a4f8a" font-size="10.5">app → agent</text>
<text x="420" y="228" text-anchor="middle" fill="#666" font-size="10.5">flock on every call</text>
<path d="M120,172 H161" stroke="#8a5a00" stroke-width="1.8" fill="none" marker-end="url(#bf-call)"/>
<path d="M315,172 H351" stroke="#8a5a00" stroke-width="1.8" fill="none" marker-end="url(#bf-call)"/>
<path d="M485,172 H521" stroke="#8a5a00" stroke-width="1.8" fill="none" stroke-dasharray="5 3" marker-end="url(#bf-call)"/>
<path d="M675,172 H716" stroke="#8a5a00" stroke-width="1.8" fill="none" stroke-dasharray="5 3" marker-end="url(#bf-call)"/>
<path d="M720,277 H679" stroke="#1a4f8a" stroke-width="1.8" fill="none" marker-end="url(#bf-user)"/>
<path d="M525,277 H489" stroke="#1a4f8a" stroke-width="1.8" fill="none" marker-end="url(#bf-user)"/>
<path d="M355,277 H319" stroke="#1a4f8a" stroke-width="1.8" fill="none" stroke-dasharray="5 3" marker-end="url(#bf-user)"/>
<path d="M165,277 H124" stroke="#1a4f8a" stroke-width="1.8" fill="none" stroke-dasharray="5 3" marker-end="url(#bf-user)"/>
<rect x="165" y="380" width="150" height="50" rx="6" fill="#fbe6e4" stroke="#a8322d"/>
<text x="240" y="400" text-anchor="middle" fill="#a8322d" font-size="12.5" font-weight="600">watcher</text>
<text x="240" y="418" text-anchor="middle" fill="#a8322d" font-size="10.5">background command</text>
<path d="M420,305 V405 H319" stroke="#a8322d" stroke-width="1.8" fill="none" stroke-dasharray="2 3" marker-end="url(#bf-wait)"/>
<text x="430" y="372" text-anchor="start" fill="#a8322d" font-size="11">reads the file directly,</text>
<text x="430" y="387" text-anchor="start" fill="#a8322d" font-size="11">not through MCP</text>
<path d="M165,405 H65 V344" stroke="#a8322d" stroke-width="1.8" fill="none" marker-end="url(#bf-wait)"/>
<text x="112" y="422" text-anchor="middle" fill="#a8322d" font-size="11">exit wakes</text>
<text x="112" y="436" text-anchor="middle" fill="#a8322d" font-size="11">the agent</text>
<circle cx="798" cy="70" r="9" fill="#1a4f8a"/><text x="798" y="74" text-anchor="middle" fill="#fff" font-size="11" font-weight="700">1</text>
<circle cx="698" cy="260" r="9" fill="#1a4f8a"/><text x="698" y="264" text-anchor="middle" fill="#fff" font-size="11" font-weight="700">2</text>
<circle cx="408" cy="340" r="9" fill="#a8322d"/><text x="408" y="344" text-anchor="middle" fill="#fff" font-size="11" font-weight="700">3</text>
<circle cx="143" cy="260" r="9" fill="#1a4f8a"/><text x="143" y="264" text-anchor="middle" fill="#fff" font-size="11" font-weight="700">4</text>
<circle cx="143" cy="155" r="9" fill="#8a5a00"/><text x="143" y="159" text-anchor="middle" fill="#fff" font-size="11" font-weight="700">5</text>
<circle cx="698" cy="155" r="9" fill="#8a5a00"/><text x="698" y="159" text-anchor="middle" fill="#fff" font-size="11" font-weight="700">6</text>
<path d="M10,480 H40" stroke="#666" stroke-width="1.8"/>
<text x="48" y="484" text-anchor="start" fill="#555" font-size="11">call that appends</text>
<path d="M170,480 H200" stroke="#666" stroke-width="1.8" stroke-dasharray="5 3"/>
<text x="208" y="484" text-anchor="start" fill="#555" font-size="11">call that reads and clears</text>
<path d="M375,480 H405" stroke="#a8322d" stroke-width="1.8" stroke-dasharray="2 3"/>
<text x="413" y="484" text-anchor="start" fill="#555" font-size="11">watcher peeks, removes nothing</text>
<text x="830" y="484" text-anchor="end" fill="#888" font-size="11">arrows show where the data goes</text>
</g>
</svg>

<ol class="flow-key">
<li><span class="tok tok-user">1</span> The user highlights a passage in the app and asks about it.</li>
<li><span class="tok tok-user">2</span> The app calls <code>notify_agent</code>, which appends the question to the channel's app-to-agent file.</li>
<li><span class="tok tok-wait">3</span> The watcher sees the file is no longer empty and exits. The host tells the agent, which gives it a turn.</li>
<li><span class="tok tok-user">4</span> The agent calls <code>poll_agent_messages</code>, which returns the question and clears the file.</li>
<li><span class="tok tok-call">5</span> The agent answers with <code>notify_ui</code>, which appends to the agent-to-app file, and starts the watcher again.</li>
<li><span class="tok tok-call">6</span> The app's next <code>poll_ui_messages</code> returns the answer, and the app shows it next to the passage.</li>
</ol>

<details class="deep-dive" markdown="1">
<summary>The same round trip as a sequence</summary>

<svg viewBox="0 0 800 1036" role="img" aria-label="Sequence diagram of one question in the Declaration reader. The agent starts a watcher and renders the reader with a channel id. The user asks about a passage; the app calls notify_agent, which appends to the app-to-agent file. The watcher sees the message and exits, which gives the agent a turn. The agent drains the question with poll_agent_messages, answers with notify_ui, and starts the watcher again. The app's next poll_ui_messages picks up the answer and shows it in a comment card." style="max-width:100%;height:auto;display:block;margin:1.5rem 0">
<defs>
<marker id="bs-user" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#1a4f8a"/></marker>
<marker id="bs-call" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#8a5a00"/></marker>
<marker id="bs-result" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#1d6b3f"/></marker>
<marker id="bs-wait" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#a8322d"/></marker>
<marker id="bs-grey" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#666"/></marker>
</defs>
<g font-family="-apple-system,BlinkMacSystemFont,Segoe UI,Helvetica,Arial,sans-serif">
<rect x="0" y="74" width="800" height="193" fill="#fdf0d5" opacity="0.55"/>
<rect x="0" y="277" width="800" height="135" fill="#e7effa" opacity="0.55"/>
<rect x="0" y="422" width="800" height="106" fill="#fbe6e4" opacity="0.55"/>
<rect x="0" y="538" width="800" height="135" fill="#e7effa" opacity="0.55"/>
<rect x="0" y="683" width="800" height="135" fill="#fdf0d5" opacity="0.55"/>
<rect x="0" y="828" width="800" height="164" fill="#e3f3e8" opacity="0.55"/>
<path d="M40,62 V1006" stroke="#bbb" stroke-dasharray="3 3"/>
<path d="M135,62 V1006" stroke="#bbb" stroke-dasharray="3 3"/>
<path d="M390,62 V1006" stroke="#bbb" stroke-dasharray="3 3"/>
<path d="M515,62 V1006" stroke="#bbb" stroke-dasharray="3 3"/>
<path d="M630,62 V1006" stroke="#bbb" stroke-dasharray="3 3"/>
<path d="M740,62 V1006" stroke="#bbb" stroke-dasharray="3 3"/>
<rect x="7.0" y="20" width="66" height="40" rx="4" fill="#e7effa" stroke="#1a4f8a"/>
<text x="40" y="45" text-anchor="middle" fill="#1a4f8a" font-size="13" font-weight="600">user</text>
<rect x="83.0" y="20" width="104" height="40" rx="4" fill="#f4f4f4" stroke="#666"/>
<text x="135" y="37" text-anchor="middle" fill="#666" font-size="13" font-weight="600">app</text>
<text x="135" y="52" text-anchor="middle" fill="#666" font-size="11">iframe</text>
<rect x="338.0" y="20" width="104" height="40" rx="4" fill="#f4f4f4" stroke="#666"/>
<text x="390" y="37" text-anchor="middle" fill="#666" font-size="13" font-weight="600">lustereczko</text>
<text x="390" y="52" text-anchor="middle" fill="#666" font-size="11">any process</text>
<rect x="463.0" y="20" width="104" height="40" rx="4" fill="#f4f4f4" stroke="#666"/>
<text x="515" y="37" text-anchor="middle" fill="#666" font-size="13" font-weight="600">channel files</text>
<text x="515" y="52" text-anchor="middle" fill="#666" font-size="11">logs/channels</text>
<rect x="578.0" y="20" width="104" height="40" rx="4" fill="#fbe6e4" stroke="#a8322d"/>
<text x="630" y="37" text-anchor="middle" fill="#a8322d" font-size="13" font-weight="600">watcher</text>
<text x="630" y="52" text-anchor="middle" fill="#a8322d" font-size="11">background</text>
<rect x="688.0" y="20" width="104" height="40" rx="4" fill="#f4f4f4" stroke="#666"/>
<text x="740" y="37" text-anchor="middle" fill="#666" font-size="13" font-weight="600">agent</text>
<text x="740" y="52" text-anchor="middle" fill="#666" font-size="11">LLM, in the chat</text>
<text x="8" y="94" text-anchor="start" fill="#8a5a00" font-size="12.5" font-weight="700">1 · The agent opens a channel and renders the reader</text>
<text x="8" y="297" text-anchor="start" fill="#1a4f8a" font-size="12.5" font-weight="700">2 · The user asks in the app</text>
<text x="8" y="442" text-anchor="start" fill="#a8322d" font-size="12.5" font-weight="700">3 · The watcher wakes the agent</text>
<text x="8" y="558" text-anchor="start" fill="#1a4f8a" font-size="12.5" font-weight="700">4 · The agent reads the question</text>
<text x="8" y="703" text-anchor="start" fill="#8a5a00" font-size="12.5" font-weight="700">5 · The agent answers, and waits again</text>
<text x="8" y="848" text-anchor="start" fill="#1d6b3f" font-size="12.5" font-weight="700">6 · The answer appears next to the passage</text>
<path d="M740,137 H634" stroke="#8a5a00" stroke-width="1.6" fill="none" marker-end="url(#bs-call)"/>
<text x="685.0" y="131" text-anchor="middle" fill="#8a5a00" font-size="12" stroke="#fdf0d5" stroke-width="4" stroke-linejoin="round" style="paint-order:stroke">start</text>
<path d="M740,166 H394" stroke="#8a5a00" stroke-width="1.6" fill="none" marker-end="url(#bs-call)"/>
<text x="565.0" y="160" text-anchor="middle" fill="#8a5a00" font-size="11.5" font-family="SFMono-Regular,Menlo,Consolas,monospace" stroke="#fdf0d5" stroke-width="4" stroke-linejoin="round" style="paint-order:stroke">display_ui_to_user(html with MY_CHANNEL)</text>
<path d="M390,195 H139" stroke="#8a5a00" stroke-width="1.6" fill="none" stroke-dasharray="5 3" marker-end="url(#bs-call)"/>
<text x="262.5" y="189" text-anchor="middle" fill="#8a5a00" font-size="12" stroke="#fdf0d5" stroke-width="4" stroke-linejoin="round" style="paint-order:stroke">HTML, rendered by the host</text>
<path d="M135,224 H386" stroke="#8a5a00" stroke-width="1.6" fill="none" stroke-dasharray="5 3" marker-end="url(#bs-call)"/>
<text x="262.5" y="218" text-anchor="middle" fill="#8a5a00" font-size="11.5" font-family="SFMono-Regular,Menlo,Consolas,monospace" stroke="#fdf0d5" stroke-width="4" stroke-linejoin="round" style="paint-order:stroke">poll_ui_messages(ch) → [ ]</text>
<text x="398" y="228" text-anchor="start" fill="#777" font-size="11" stroke="#fdf0d5" stroke-width="4" stroke-linejoin="round" style="paint-order:stroke">every 1.2 s, all along</text>
<path d="M630,253 H519" stroke="#a8322d" stroke-width="1.6" fill="none" stroke-dasharray="2 3" marker-end="url(#bs-wait)"/>
<text x="572.5" y="247" text-anchor="middle" fill="#a8322d" font-size="12" stroke="#fdf0d5" stroke-width="4" stroke-linejoin="round" style="paint-order:stroke">peeks: empty</text>
<path d="M40,340 H131" stroke="#1a4f8a" stroke-width="1.6" fill="none" marker-end="url(#bs-user)"/>
<text x="87.5" y="334" text-anchor="middle" fill="#1a4f8a" font-size="12" stroke="#e7effa" stroke-width="4" stroke-linejoin="round" style="paint-order:stroke">highlights, asks</text>
<path d="M135,369 H386" stroke="#1a4f8a" stroke-width="1.6" fill="none" marker-end="url(#bs-user)"/>
<text x="262.5" y="363" text-anchor="middle" fill="#1a4f8a" font-size="11.5" font-family="SFMono-Regular,Menlo,Consolas,monospace" stroke="#e7effa" stroke-width="4" stroke-linejoin="round" style="paint-order:stroke">notify_agent(&quot;ask&quot;, {quote, question, …})</text>
<path d="M390,398 H511" stroke="#1a4f8a" stroke-width="1.6" fill="none" marker-end="url(#bs-user)"/>
<text x="452.5" y="392" text-anchor="middle" fill="#1a4f8a" font-size="12" stroke="#e7effa" stroke-width="4" stroke-linejoin="round" style="paint-order:stroke">append</text>
<path d="M515,485 H626" stroke="#a8322d" stroke-width="1.6" fill="none" stroke-dasharray="2 3" marker-end="url(#bs-wait)"/>
<text x="572.5" y="479" text-anchor="middle" fill="#a8322d" font-size="12" stroke="#fbe6e4" stroke-width="4" stroke-linejoin="round" style="paint-order:stroke">1 message waiting</text>
<path d="M630,514 H736" stroke="#a8322d" stroke-width="1.6" fill="none" marker-end="url(#bs-wait)"/>
<text x="685.0" y="508" text-anchor="middle" fill="#a8322d" font-size="12" stroke="#fbe6e4" stroke-width="4" stroke-linejoin="round" style="paint-order:stroke">exits: a new turn</text>
<path d="M740,601 H394" stroke="#1a4f8a" stroke-width="1.6" fill="none" marker-end="url(#bs-user)"/>
<text x="565.0" y="595" text-anchor="middle" fill="#1a4f8a" font-size="11.5" font-family="SFMono-Regular,Menlo,Consolas,monospace" stroke="#e7effa" stroke-width="4" stroke-linejoin="round" style="paint-order:stroke">poll_agent_messages(ch)</text>
<path d="M515,630 H394" stroke="#1a4f8a" stroke-width="1.6" fill="none" stroke-dasharray="5 3" marker-end="url(#bs-user)"/>
<text x="452.5" y="624" text-anchor="middle" fill="#1a4f8a" font-size="12" stroke="#e7effa" stroke-width="4" stroke-linejoin="round" style="paint-order:stroke">read and clear</text>
<path d="M390,659 H736" stroke="#1a4f8a" stroke-width="1.6" fill="none" stroke-dasharray="5 3" marker-end="url(#bs-user)"/>
<text x="565.0" y="653" text-anchor="middle" fill="#1a4f8a" font-size="11.5" font-family="SFMono-Regular,Menlo,Consolas,monospace" stroke="#e7effa" stroke-width="4" stroke-linejoin="round" style="paint-order:stroke">[{&quot;event&quot;: &quot;ask&quot;, …}]</text>
<path d="M740,746 H394" stroke="#8a5a00" stroke-width="1.6" fill="none" marker-end="url(#bs-call)"/>
<text x="565.0" y="740" text-anchor="middle" fill="#8a5a00" font-size="11.5" font-family="SFMono-Regular,Menlo,Consolas,monospace" stroke="#fdf0d5" stroke-width="4" stroke-linejoin="round" style="paint-order:stroke">notify_ui(&quot;answer&quot;, {thread_id, text})</text>
<path d="M390,775 H511" stroke="#8a5a00" stroke-width="1.6" fill="none" marker-end="url(#bs-call)"/>
<text x="452.5" y="769" text-anchor="middle" fill="#8a5a00" font-size="12" stroke="#fdf0d5" stroke-width="4" stroke-linejoin="round" style="paint-order:stroke">append</text>
<path d="M740,804 H634" stroke="#8a5a00" stroke-width="1.6" fill="none" marker-end="url(#bs-call)"/>
<text x="685.0" y="798" text-anchor="middle" fill="#8a5a00" font-size="12" stroke="#fdf0d5" stroke-width="4" stroke-linejoin="round" style="paint-order:stroke">start again</text>
<path d="M135,891 H386" stroke="#8a5a00" stroke-width="1.6" fill="none" marker-end="url(#bs-call)"/>
<text x="262.5" y="885" text-anchor="middle" fill="#8a5a00" font-size="11.5" font-family="SFMono-Regular,Menlo,Consolas,monospace" stroke="#e3f3e8" stroke-width="4" stroke-linejoin="round" style="paint-order:stroke">poll_ui_messages(ch)</text>
<path d="M515,920 H394" stroke="#8a5a00" stroke-width="1.6" fill="none" stroke-dasharray="5 3" marker-end="url(#bs-call)"/>
<text x="452.5" y="914" text-anchor="middle" fill="#8a5a00" font-size="12" stroke="#e3f3e8" stroke-width="4" stroke-linejoin="round" style="paint-order:stroke">read and clear</text>
<path d="M390,949 H139" stroke="#8a5a00" stroke-width="1.6" fill="none" stroke-dasharray="5 3" marker-end="url(#bs-call)"/>
<text x="262.5" y="943" text-anchor="middle" fill="#8a5a00" font-size="11.5" font-family="SFMono-Regular,Menlo,Consolas,monospace" stroke="#e3f3e8" stroke-width="4" stroke-linejoin="round" style="paint-order:stroke">[{&quot;event&quot;: &quot;answer&quot;, …}]</text>
<path d="M135,978 H44" stroke="#1d6b3f" stroke-width="1.6" fill="none" marker-end="url(#bs-result)"/>
<text x="87.5" y="972" text-anchor="middle" fill="#1d6b3f" font-size="12" stroke="#e3f3e8" stroke-width="4" stroke-linejoin="round" style="paint-order:stroke">comment card</text>
<text x="8" y="1026" text-anchor="start" fill="#666" font-size="11">One lustereczko lifeline stands for every server process: whichever one handles a call, it reads and writes the same files.</text>
</g>
</svg>
</details>

## The four tools

In lustereczko, a channel is four tools and two files, one file per direction ([source](https://github.com/pslusarz/lustereczko-mcp/blob/main/mcp/src/main/tools/bidirectional_streaming.py)). You could back a channel with a proper database, but a file is the default and it is enough.

| Tool | Called by | What it does |
|---|---|---|
| [`notify_ui`](https://github.com/pslusarz/lustereczko-mcp/blob/main/mcp/src/main/tools/bidirectional_streaming.py#L62) | agent | Appends `{event, data}` to `ui_<channel>.json` |
| [`poll_ui_messages`](https://github.com/pslusarz/lustereczko-mcp/blob/main/mcp/src/main/tools/bidirectional_streaming.py#L91) | app, every 1-1.5 s | Returns everything in `ui_<channel>.json` and empties it |
| [`notify_agent`](https://github.com/pslusarz/lustereczko-mcp/blob/main/mcp/src/main/tools/bidirectional_streaming.py#L111) | app | Appends `{event, data}` to `agent_<channel>.json` |
| [`poll_agent_messages`](https://github.com/pslusarz/lustereczko-mcp/blob/main/mcp/src/main/tools/bidirectional_streaming.py#L130) | agent, when woken | Returns everything in `agent_<channel>.json` and empties it |

The two tools the app uses are declared with app-only [visibility](https://github.com/modelcontextprotocol/ext-apps/blob/main/specification/2026-01-26/apps.mdx), so the host leaves them out of the agent's tool list, and the agent cannot drain the app's queue by mistake.

The agent picks the channel id before it renders, for example `ch-1749312345678`, and bakes it into the HTML as a constant. Both sides know it from the start, so there is no handshake, and every render gets a fresh channel. An old copy of the app that is still running keeps polling its own channel and receives nothing meant for the new one. The id doubles as part of the file name, so the server only accepts letters, digits, hyphens and underscores ([`_CHANNEL_RE`](https://github.com/pslusarz/lustereczko-mcp/blob/main/mcp/src/main/tools/bidirectional_streaming.py#L18)), which keeps a channel id from turning into a path.

### Why it is safe across processes

Every append and every drain opens the file and takes an exclusive [`flock`](https://man7.org/linux/man-pages/man2/flock.2.html) on it before reading, then writes the new contents and lets go when the file closes ([`_queue_append`](https://github.com/pslusarz/lustereczko-mcp/blob/main/mcp/src/main/tools/bidirectional_streaming.py#L36), [`_queue_drain`](https://github.com/pslusarz/lustereczko-mcp/blob/main/mcp/src/main/tools/bidirectional_streaming.py#L48)). The lock belongs to the operating system, not to a process, so it does not matter which copy of the server a call lands on: two appends cannot interleave, and an append cannot slip in between a drain's read and its clear. Reading and clearing in one locked step means each message is handed out once. That assumes one reader per direction, which is the case: one app polls one channel, and one agent drains it.

The watcher is the exception that proves the rule. It does not go through MCP at all: it reads the agent's file straight off the disk, without the lock, and it never writes. If it catches the file in the middle of a write, it sees an empty or half-written file, treats that as nothing, and looks again half a second later. Because it removes nothing, the worst it can do is wake the agent half a second late. The real read still goes through `poll_agent_messages`, under the lock.

<details class="deep-dive" markdown="1">
<summary>The drain and the watcher side by side</summary>

Both are trimmed to their shape. The watcher is not part of the server: the agent writes it from the [bidirectional events recipe](https://github.com/pslusarz/lustereczko-mcp/blob/main/skills/recipes/bidirectional-events.md) and runs it as a background command.

<div class="sbs" markdown="1">
<figure class="panel" markdown="1">
<figcaption>Server: read and clear under the lock</figcaption>

```python
def _queue_drain(queue_file):
    with open(queue_file, "r+") as f:
        fcntl.flock(f, fcntl.LOCK_EX)
        items = json.loads(f.read() or "[]")
        f.seek(0)
        f.truncate()
        json.dump([], f)
    # closing the file releases the lock
    return items
```

</figure>
<figure class="panel" markdown="1">
<figcaption>Watcher: peek, and exit when there is mail</figcaption>

```python
q = CHANNELS / f"agent_{channel}.json"
while time.monotonic() < deadline:
    try:
        items = json.loads(q.read_text()
                           or "[]")
    except (FileNotFoundError,
            json.JSONDecodeError):
        items = []  # mid-write: look again
    if items:
        sys.exit(0)  # the exit wakes the agent
    time.sleep(0.5)
sys.exit(2)
```

</figure>
</div>

`fcntl.flock` exists on macOS and Linux. On Windows the server would need a different lock, or a database.
</details>

## Waking the agent

The app's half is easy: it polls `poll_ui_messages` on a timer for as long as it is open. The agent's half depends on what the host offers.

- **A watcher.** If the host tells the agent when a background command finishes, the agent starts the watcher before it renders the app, and starts it again after every answer. GitHub Copilot in VS Code does this with [background terminal commands](https://code.visualstudio.com/docs/copilot/agents/agent-tools#_continue-terminal-commands-in-background), and so does Claude Code with [background Bash commands](https://code.claude.com/docs/en/interactive-mode#background-bash-commands) and its [Monitor tool](https://code.claude.com/docs/en/tools-reference#monitor-tool), which only interrupts the agent when the script prints a line. Waiting costs no tokens, and the chat stays free for the user.
- **A cron job.** Hosts with scheduled prompts, such as Claude Code's [`/loop` and scheduled tasks](https://code.claude.com/docs/en/scheduled-tasks), can run `poll_agent_messages` on a schedule instead. It is simpler, but the schedule fires a prompt, not a script: every run is a turn for the model, even when there is no mail, and a reply can take up to a full interval, at least a minute.

Background wake-ups are not instant either; in Copilot they can take 10 to 30 seconds. For the video, the agent stayed in its turn and ran the watcher in the foreground, answering each question as soon as it arrived. More detail on both options is in the [bidirectional events recipe](https://github.com/pslusarz/lustereczko-mcp/blob/main/skills/recipes/bidirectional-events.md), and the reader itself follows the [margin comments recipe](https://github.com/pslusarz/lustereczko-mcp/blob/main/skills/recipes/margin-comments.md).

## Try it

If you would like to try this yourself, install [lustereczko-mcp](https://github.com/pslusarz/lustereczko-mcp) as a local MCP server, and ask your agent to build you the Declaration of Independence interactive reader. It is tested in the Claude desktop app, where MCP App support has moved from Cowork to Code, so make sure you are in the Code tab, and in GitHub Copilot in VS Code. Please let me know if you get it working in any other MCP host.

Thank you for reading, and let me know your thoughts.
