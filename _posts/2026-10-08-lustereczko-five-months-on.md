---
title: "Lustereczko, five months on"
date: 2026-10-08
description: "Lustereczko lets your agent generate interactive UIs and small apps that act on your machine. Five months in, here is what it can do and how it works in MCP."
---

OpenAI announced [Intelligent UI](https://openai.com/index/gpt-6-for-everyone/) on October 7, 151 days after the [debut of lustereczko-mcp](https://www.reddit.com/r/mcp/comments/1t8xgq9/llm_generated_uis_in_mcp_apps_actual_working/). This may be cause for celebration, in that a single developer is often able to innovate at a much higher rate than even the frontier labs. It is also an opportunity to reflect on over 4 months of use and progress that still leaves Intelligent UI in the dust as far as capabilities. Where is lustereczko now and what innovative ways of interacting with an agent does it offer?

Lustereczko-mcp is rooted in a belief that LLMs can interact with the user in a much richer way than through chat. We use LLMs to generate copious amounts of code every day, why not use the same capability to generate on-the-fly custom interfaces to communicate with the user? We can start with a simple example: let's visualize something. OpenAI gives us a [bicycle](https://help.openai.com/en/articles/20001598-intelligent-ui-in-chatgpt), but I think our readers could use something better - let's see if we can give you a good intuition why a positive result from a test that correctly identifies 99% of the positive cases can still mean you have less than a 10% chance of actually being sick. Here is what my agent built for that:

<video controls autoplay loop muted playsinline
       poster="/articles/docs/assets/2026-10-08-lustereczko-five-months-on/bayes-tree.png"
       style="max-width:100%;height:auto;display:block;margin:1.5rem 0;border:1px solid #d0d7de;border-radius:.5rem"
       aria-label="A decision tree of 1,000 people. They split into 1 sick and 999 healthy, then each group splits by the test result. The two positive groups are tallied: 1 sick to 10 healthy, a 9.1% chance of being sick. Moving the sliders to 50 sick people changes the tally to 50 to 10, and a test that clears 99.9% of healthy people gives 1 to 1.">
  <source src="/articles/docs/assets/2026-10-08-lustereczko-five-months-on/bayes-tree.mp4" type="video/mp4">
  <img src="/articles/docs/assets/2026-10-08-lustereczko-five-months-on/bayes-tree.png"
       alt="The finished decision tree: of the 11 people who tested positive, 1 is sick and 10 are healthy, a 9.1% chance of being sick.">
</video>

Can you see how effective UIs can be at conveying complicated ideas? Bret Victor made the same point years ago with his [circuit demo in Inventing on Principle](https://vimeo.com/36579366#t=1397s).

However, that is just a beginning, and our biggest bottleneck right now is our own habit of thinking of UI as difficult to change. Imagine trying to sort a few hundred photos in a directory into buckets. LLM can help you, but ultimately you need to glance at each photo to decide where it will go. You can use a couple of Finder windows with a gallery view, and stitch everything together yourself and then beat your head against the wall as you perform the mind-numbing file drags and renames, or... you can ask your agent to make you a custom UI in lustereczko.

<div class="sbs">
<figure class="panel" style="flex:1 1 13rem">
<figcaption>500 photos in Finder</figcaption>
<img src="/articles/docs/assets/2026-10-08-lustereczko-five-months-on/finder.png"
     alt="A Finder window with five empty folders (Animals, Delete, Food, Landscapes, People) above the first 20 of 500 photos, IMG_4001.jpg onwards, with a tiny scrollbar."
     style="max-width:100%;height:auto;border:1px solid #d0d7de;border-radius:.5rem">
</figure>
<figure class="panel" style="flex:2 1 22rem">
<figcaption>The same 500 in the sorter my agent built</figcaption>
<video controls autoplay loop muted playsinline
       poster="/articles/docs/assets/2026-10-08-lustereczko-five-months-on/photo-sorter.png"
       style="max-width:100%;height:auto;display:block;border:1px solid #d0d7de;border-radius:.5rem"
       aria-label="The photo sorter. Photos are filed into People, Landscapes, Animals, Food and Delete with number keys, one after another. Seven landscapes are picked with command-click and filed at once. Pressing Move 28 files moves them into folders and tells the agent what was sorted.">
  <source src="/articles/docs/assets/2026-10-08-lustereczko-five-months-on/photo-sorter.mp4" type="video/mp4">
  <img src="/articles/docs/assets/2026-10-08-lustereczko-five-months-on/photo-sorter.png"
       alt="The sorter mid-way: a large photo of a car, a grid of thumbnails with coloured dots for the ones already filed, and a Move 21 files button.">
</video>
</figure>
</div>

Ultimately, I think we will arrive at a human-LLM interaction model where the agent buries itself deep inside a custom application that it had generated to interact with the user. Something like a book author that is there to speak with you about their book, and answering all your questions, and going on tangents that there was no place for in the book... Maybe also quizzing you on the content to make sure you are learning the material. For a vision like that, MCP-App runs into limitations since you are still within the host chat UI and it is probably not the right platform. In fact, the lustereczko-mcp project originally started as a demo that dynamic UI could not be done within MCP, and I was expecting it to be proved in one evening.

I was trying to keep this post short, so you can stop reading here if you are just interested in the ideas. The rest of this post discusses some details about what capabilities exist and how some of this was accomplished within MCP and the MCP-App extension. The code is in [pslusarz/lustereczko-mcp](https://github.com/pslusarz/lustereczko-mcp).

## How it works

Lustereczko-mcp is a general purpose MCP-App server designed to be run locally. You can think of it as a custom app platform your agent can deploy to. In some ways this is not that different from Anthropic's artifacts, just built on an open platform and also designed to do useful things on your machine.

To allow a dynamic UI to display, we immediately hack the MCP-App static resource, replacing its blank canvas with code passed to the tool. We needed to use a couple of tricks for that, but it works (see [`display_ui_to_user`](https://github.com/pslusarz/lustereczko-mcp/blob/main/mcp/src/main/server.py#L102) and [the bridge](https://github.com/pslusarz/lustereczko-mcp/blob/main/mcp/src/main/templates.py#L19)). In order to stabilize things, we also expose some skills as tools on the server to tell the agent how to get the most out of the UI. The agent is strongly encouraged to use HTMX in order to avoid the complicated event flows that take place in React. This isn't a fundamental limitation, but a practical one, given current LLM capabilities and some practical reports: Simon Willison [tells his agents to avoid React](https://simonwillison.net/2025/Dec/10/html-tools/) because those attempts are more likely to crash, and the htmx folks make the case for [hypermedia MCP apps](https://htmx.org/essays/mcp-apps-hypermedia/).

<details class="deep-dive" markdown="1">
<summary>The trick behind the blank canvas</summary>

The resource the host loads is the same page every time: htmx, the MCP Apps bridge, and an empty container. The HTML the agent wrote travels in the tool result's `_meta`, and the bridge pours it into the container. Scripts inserted through `innerHTML` do not run, so the bridge re-creates each one. Both sides are trimmed to their shape.

<div class="sbs" markdown="1">
<figure class="panel" markdown="1">
<figcaption>Server: the tool hands back the HTML</figcaption>

```python
@mcp.tool(app=AppConfig(
    resource_uri="ui://display"))
def display_ui_to_user(html_fragment: str):
    return ToolResult(
        content=[TextContent(type="text",
            text="Content displayed to user.")],
        meta={"html": html_fragment},
    )
```

</figure>
<figure class="panel" markdown="1">
<figcaption>Bridge: the page pours it in</figcaption>

```js
app.ontoolresult = (result) => {
  const html = result._meta?.html;
  if (!html) return;
  container.innerHTML = html;
  htmx.process(container);
  // innerHTML does not run scripts,
  // so each one is re-created
  executeScripts(container);
};
```

</figure>
</div>
</details>

Next, we had to figure out how the agent can communicate with the UI bidirectionally. MCP app protocol has a provision for pushing a [tool result into the UI](https://github.com/modelcontextprotocol/ext-apps/blob/main/specification/2026-01-26/apps.mdx), but it is very clunky and allows only a single event to happen: the result of the call that rendered it. Initially, I implemented a logging tool, and instructed the agent how to instrument it for debugging. What works better is communication over named channels, because of lack of control over how host systems choose to run the MCP server (on Anthropic's platform, you will have several instances running, so messages do not get read without a named channel and durable backing). Here we still have some shortcomings, because we do not have a sure way to trigger the host system to keep polling. Many host systems have scheduled jobs, and this works on Anthropic's Claude Cowork desktop app, so long as the agent is reminded to use it.

The MCP spec also includes an obscure ability of the server to call back to the host's LLM, known as [sampling](https://modelcontextprotocol.io/specification/2026-07-28/client/sampling). The problem is that sampling is barely supported: according to [this compatibility matrix](https://caniuse.dev/capabilities/sampling), VS Code and Goose support it, while Claude Desktop, Claude Code, ChatGPT and Cursor do not. When I tried it from the Copilot agent in VS Code, the request never even left the server, because under the latest version of the protocol (2026-07-28) the server can no longer initiate requests to the host, and sampling itself is deprecated. So we chose not to use it in lustereczko.

<details class="deep-dive" markdown="1">
<summary>What a channel looks like</summary>

The agent picks the channel id before it renders, and bakes it into the HTML as a constant, so both sides know it without a handshake. Each channel is a pair of small JSON queues on disk behind a file lock, which is the durable backing: it does not matter which copy of the server a call lands on.

<pre class="wire"><span class="tok tok-call">agent</span>  channel = "ch-1749312345678"
<span class="tok tok-call">agent</span>  display_ui_to_user(html with MY_CHANNEL = "ch-1749312345678")
<span class="tok tok-user">ui</span>     poll_ui_messages(ch)      <span class="tok tok-result">[]</span>              every 1.5 s
<span class="tok tok-call">agent</span>  notify_ui(ch, "refresh")
<span class="tok tok-user">ui</span>     poll_ui_messages(ch)      <span class="tok tok-result">[refresh]</span>
<span class="tok tok-user">ui</span>     notify_agent(ch, "ack")
<span class="tok tok-call">agent</span>  poll_agent_messages(ch)   <span class="tok tok-result">[ack]</span>
</pre>

The last line is the weak spot from the paragraph above: nothing wakes the agent up to make that call.
</details>

Finally, we would like to equip our app with the capability to perform some actions on the user's system. This is done through custom MCP tool deployment, available via `add_custom_tool` and `run_custom_tool`. These tools can be run by both the agent and the UI, although they are mostly intended for the UI. The code backing them gets executed on the host machine.

<details class="deep-dive" markdown="1">
<summary>Why there is no tool to list custom tools</summary>

It is unnecessary, since the agent knows what tools it just deployed.
</details>

It may help to visualize how this is all put together on the example of a file rename tool:

<svg viewBox="0 0 800 420" role="img" aria-label="Topology: everything but the LLM runs on the local machine. The agent deploys a rename tool and a file browser through the host into lustereczko; the user renames a file in the app, which calls the tool directly; the app then tells the chat, which tells the agent." style="max-width:100%;height:auto;display:block;margin:1.5rem 0">
<defs>
<marker id="tp-user" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#1a4f8a"/></marker>
<marker id="tp-call" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#8a5a00"/></marker>
<marker id="tp-result" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#1d6b3f"/></marker>
<marker id="tp-wait" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#a8322d"/></marker>
<marker id="tp-grey" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#666"/></marker>
</defs>
<g font-family="-apple-system,BlinkMacSystemFont,Segoe UI,Helvetica,Arial,sans-serif">
<rect x="120" y="92" width="670" height="318" rx="10" fill="#fafafa" stroke="#666" stroke-dasharray="5 3"/>
<text x="134" y="112" text-anchor="start" fill="#555" font-size="12.5" font-weight="700">local machine</text>
<rect x="265" y="10" width="160" height="50" rx="24" fill="#f4f4f4" stroke="#666"/>
<text x="345" y="32" text-anchor="middle" fill="#333" font-size="13" font-weight="600">agent</text>
<text x="345" y="48" text-anchor="middle" fill="#666" font-size="11">LLM, remote</text>
<rect x="10" y="232" width="80" height="40" rx="4" fill="#e7effa" stroke="#1a4f8a"/>
<text x="50" y="257" text-anchor="middle" fill="#1a4f8a" font-size="13" font-weight="600">user</text>
<rect x="140" y="124" width="240" height="270" rx="8" fill="#fff" stroke="#666"/>
<text x="152" y="142" text-anchor="start" fill="#555" font-size="11.5" font-weight="600">MCP host (Claude Desktop)</text>
<rect x="156" y="154" width="208" height="88" rx="4" fill="#f4f4f4" stroke="#666"/>
<text x="260" y="172" text-anchor="middle" fill="#333" font-size="13" font-weight="600">chat</text>
<text x="260" y="192" text-anchor="middle" fill="#777" font-size="11" font-family="SFMono-Regular,Menlo,Consolas,monospace">add_custom_tool …</text>
<text x="260" y="208" text-anchor="middle" fill="#777" font-size="11" font-family="SFMono-Regular,Menlo,Consolas,monospace">display_ui_to_user …</text>
<text x="260" y="228" text-anchor="middle" fill="#1d6b3f" font-size="11.5">“renamed to kayak.jpg”</text>
<rect x="156" y="262" width="208" height="116" rx="4" fill="#f4f4f4" stroke="#666"/>
<text x="260" y="280" text-anchor="middle" fill="#333" font-size="13" font-weight="600">app (iframe)</text>
<rect x="176" y="292" width="168" height="20" rx="3" fill="#fff" stroke="#ddd"/>
<text x="186" y="306" text-anchor="start" fill="#444" font-size="11" font-family="SFMono-Regular,Menlo,Consolas,monospace">IMG_0411.jpg</text>
<rect x="176" y="318" width="168" height="20" rx="3" fill="#fff" stroke="#ddd"/>
<text x="186" y="332" text-anchor="start" fill="#444" font-size="11" font-family="SFMono-Regular,Menlo,Consolas,monospace">kayak.jpg  ✎</text>
<rect x="176" y="344" width="168" height="20" rx="3" fill="#fff" stroke="#ddd"/>
<text x="186" y="358" text-anchor="start" fill="#444" font-size="11" font-family="SFMono-Regular,Menlo,Consolas,monospace">IMG_0413.jpg</text>
<rect x="450" y="124" width="180" height="270" rx="8" fill="#fff" stroke="#666"/>
<text x="540" y="144" text-anchor="middle" fill="#333" font-size="13" font-weight="600">lustereczko</text>
<text x="540" y="159" text-anchor="middle" fill="#666" font-size="11">MCP server</text>
<text x="466" y="186" text-anchor="start" fill="#555" font-size="11" font-family="SFMono-Regular,Menlo,Consolas,monospace">add_custom_tool</text>
<text x="466" y="203" text-anchor="start" fill="#555" font-size="11" font-family="SFMono-Regular,Menlo,Consolas,monospace">display_ui_to_user</text>
<text x="466" y="220" text-anchor="start" fill="#555" font-size="11" font-family="SFMono-Regular,Menlo,Consolas,monospace">run_custom_tool</text>
<text x="466" y="237" text-anchor="start" fill="#555" font-size="11" font-family="SFMono-Regular,Menlo,Consolas,monospace">notify_ui …</text>
<rect x="466" y="270" width="148" height="108" rx="4" fill="#fff" stroke="#8a5a00"/>
<text x="540" y="288" text-anchor="middle" fill="#8a5a00" font-size="11.5" font-weight="600">custom tools</text>
<text x="540" y="310" text-anchor="middle" fill="#444" font-size="11" font-family="SFMono-Regular,Menlo,Consolas,monospace">rename_file.py</text>
<text x="540" y="330" text-anchor="middle" fill="#777" font-size="10.5" font-family="SFMono-Regular,Menlo,Consolas,monospace">def run(src, dst):</text>
<text x="540" y="346" text-anchor="middle" fill="#777" font-size="10.5" font-family="SFMono-Regular,Menlo,Consolas,monospace">  os.rename(…)</text>
<rect x="670" y="240" width="104" height="90" rx="4" fill="#fff" stroke="#666"/>
<text x="722" y="260" text-anchor="middle" fill="#333" font-size="12.5" font-weight="600">~/Photos</text>
<text x="722" y="280" text-anchor="middle" fill="#666" font-size="10.5" font-family="SFMono-Regular,Menlo,Consolas,monospace">IMG_0411.jpg</text>
<text x="722" y="296" text-anchor="middle" fill="#666" font-size="10.5" font-family="SFMono-Regular,Menlo,Consolas,monospace">kayak.jpg</text>
<text x="722" y="312" text-anchor="middle" fill="#666" font-size="10.5" font-family="SFMono-Regular,Menlo,Consolas,monospace">IMG_0413.jpg</text>
<path d="M330,60 V150" stroke="#8a5a00" stroke-width="1.8" fill="none" marker-end="url(#tp-call)"/>
<path d="M364,186 H446" stroke="#8a5a00" stroke-width="1.8" fill="none" marker-end="url(#tp-call)"/>
<path d="M450,232 C410,232 410,300 368,300" stroke="#8a5a00" stroke-width="1.8" fill="none" stroke-dasharray="5 3" marker-end="url(#tp-call)"/>
<path d="M90,252 C120,252 120,318 152,318" stroke="#1a4f8a" stroke-width="1.8" fill="none" marker-end="url(#tp-user)"/>
<path d="M364,340 H462" stroke="#1a4f8a" stroke-width="1.8" fill="none" marker-end="url(#tp-user)"/>
<path d="M614,310 C642,310 642,286 666,286" stroke="#1a4f8a" stroke-width="1.8" fill="none" marker-end="url(#tp-user)"/>
<path d="M300,262 V246" stroke="#1d6b3f" stroke-width="1.8" fill="none" marker-end="url(#tp-result)"/>
<path d="M352,154 V64" stroke="#1d6b3f" stroke-width="1.8" fill="none" marker-end="url(#tp-result)"/>
<circle cx="316" cy="105" r="9" fill="#8a5a00"/><text x="316" y="109" text-anchor="middle" fill="#fff" font-size="11" font-weight="700">1</text>
<circle cx="405" cy="176" r="9" fill="#8a5a00"/><text x="405" y="180" text-anchor="middle" fill="#fff" font-size="11" font-weight="700">2</text>
<circle cx="418" cy="262" r="9" fill="#8a5a00"/><text x="418" y="266" text-anchor="middle" fill="#fff" font-size="11" font-weight="700">3</text>
<circle cx="110" cy="270" r="9" fill="#1a4f8a"/><text x="110" y="274" text-anchor="middle" fill="#fff" font-size="11" font-weight="700">4</text>
<circle cx="405" cy="356" r="9" fill="#1a4f8a"/><text x="405" y="360" text-anchor="middle" fill="#fff" font-size="11" font-weight="700">5</text>
<circle cx="645" cy="322" r="9" fill="#1a4f8a"/><text x="645" y="326" text-anchor="middle" fill="#fff" font-size="11" font-weight="700">6</text>
<circle cx="318" cy="254" r="9" fill="#1d6b3f"/><text x="318" y="258" text-anchor="middle" fill="#fff" font-size="11" font-weight="700">7</text>
<circle cx="368" cy="105" r="9" fill="#1d6b3f"/><text x="368" y="109" text-anchor="middle" fill="#fff" font-size="11" font-weight="700">8</text>
</g>
</svg>

<ol class="flow-key">
<li><span class="tok tok-call">1</span> The agent calls lustereczko. Its tool calls go to the host.</li>
<li><span class="tok tok-call">2</span> The host relays <code>add_custom_tool</code>, which saves <code>rename_file.py</code>, and <code>display_ui_to_user</code>.</li>
<li><span class="tok tok-call">3</span> The HTML comes back in the tool result, and the host renders it in the app's iframe.</li>
<li><span class="tok tok-user">4</span> The user renames a file in the app.</li>
<li><span class="tok tok-user">5</span> The app calls <code>run_custom_tool("rename_file", …)</code>. The host relays it, but it never reaches the LLM.</li>
<li><span class="tok tok-user">6</span> The tool renames the file on disk.</li>
<li><span class="tok tok-result">7</span> The app posts the rename into the chat with <code>sendMessage</code>.</li>
<li><span class="tok tok-result">8</span> The agent reads it on its next turn.</li>
</ol>

<details class="deep-dive" markdown="1">
<summary>The same flow as a sequence, including the agent using the tool itself</summary>

<svg viewBox="0 0 800 804" role="img" aria-label="Sequence diagram: the agent deploys a rename_file tool and a file browser UI through lustereczko; the user renames a file in the UI, which calls the tool directly on the local machine without the LLM; the UI then posts the rename into the chat; finally the agent calls the same tool itself." style="max-width:100%;height:auto;display:block;margin:1.5rem 0">
<defs>
<marker id="sq-user" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#1a4f8a"/></marker>
<marker id="sq-call" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#8a5a00"/></marker>
<marker id="sq-result" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#1d6b3f"/></marker>
<marker id="sq-wait" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#a8322d"/></marker>
<marker id="sq-grey" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#666"/></marker>
</defs>
<g font-family="-apple-system,BlinkMacSystemFont,Segoe UI,Helvetica,Arial,sans-serif">
<rect x="0" y="78" width="800" height="208" fill="#fdf0d5" opacity="0.55"/>
<rect x="0" y="296" width="800" height="150" fill="#e7effa" opacity="0.55"/>
<rect x="0" y="456" width="800" height="92" fill="#e3f3e8" opacity="0.55"/>
<rect x="0" y="558" width="800" height="208" fill="#fdf0d5" opacity="0.55"/>
<path d="M45,62 V774" stroke="#bbb" stroke-dasharray="3 3"/>
<path d="M155,62 V774" stroke="#bbb" stroke-dasharray="3 3"/>
<path d="M280,62 V774" stroke="#bbb" stroke-dasharray="3 3"/>
<path d="M400,62 V774" stroke="#bbb" stroke-dasharray="3 3"/>
<path d="M590,62 V774" stroke="#bbb" stroke-dasharray="3 3"/>
<path d="M730,62 V774" stroke="#bbb" stroke-dasharray="3 3"/>
<rect x="222" y="2" width="236" height="66" rx="6" fill="none" stroke="#999" stroke-dasharray="4 3"/>
<text x="230" y="14" text-anchor="start" fill="#777" font-size="10.5" font-weight="600">MCP host (Claude Desktop)</text>
<rect x="10.0" y="20" width="70" height="40" rx="4" fill="#e7effa" stroke="#1a4f8a"/>
<text x="45" y="45" text-anchor="middle" fill="#1a4f8a" font-size="13" font-weight="600">user</text>
<rect x="105.0" y="20" width="100" height="40" rx="4" fill="#f4f4f4" stroke="#666"/>
<text x="155" y="37" text-anchor="middle" fill="#666" font-size="13" font-weight="600">agent</text>
<text x="155" y="52" text-anchor="middle" fill="#666" font-size="11">LLM, remote</text>
<rect x="232.0" y="20" width="96" height="40" rx="4" fill="#f4f4f4" stroke="#666"/>
<text x="280" y="45" text-anchor="middle" fill="#666" font-size="13" font-weight="600">chat</text>
<rect x="352.0" y="20" width="96" height="40" rx="4" fill="#f4f4f4" stroke="#666"/>
<text x="400" y="37" text-anchor="middle" fill="#666" font-size="13" font-weight="600">app</text>
<text x="400" y="52" text-anchor="middle" fill="#666" font-size="11">iframe</text>
<rect x="530.0" y="20" width="120" height="40" rx="4" fill="#f4f4f4" stroke="#666"/>
<text x="590" y="37" text-anchor="middle" fill="#666" font-size="13" font-weight="600">lustereczko</text>
<text x="590" y="52" text-anchor="middle" fill="#666" font-size="11">MCP server</text>
<rect x="675.0" y="20" width="110" height="40" rx="4" fill="#f4f4f4" stroke="#666"/>
<text x="730" y="37" text-anchor="middle" fill="#666" font-size="13" font-weight="600">local machine</text>
<text x="730" y="52" text-anchor="middle" fill="#666" font-size="11">file system</text>
<text x="8" y="96" text-anchor="start" fill="#8a5a00" font-size="12.5" font-weight="700">1 · The agent builds the app</text>
<path d="M45,122 H276" stroke="#1a4f8a" stroke-width="1.6" fill="none" marker-end="url(#sq-user)"/>
<text x="162.5" y="116" text-anchor="middle" fill="#1a4f8a" font-size="12" stroke="#fdf0d5" stroke-width="4" stroke-linejoin="round" style="paint-order:stroke">“give me a file browser for ~/Photos”</text>
<path d="M280,151 H159" stroke="#666" stroke-width="1.6" fill="none" marker-end="url(#sq-grey)"/>
<text x="217.5" y="145" text-anchor="middle" fill="#666" font-size="12" stroke="#fdf0d5" stroke-width="4" stroke-linejoin="round" style="paint-order:stroke">prompt</text>
<path d="M155,180 H586" stroke="#8a5a00" stroke-width="1.6" fill="none" marker-end="url(#sq-call)"/>
<circle cx="280" cy="180" r="3.5" fill="#fff" stroke="#8a5a00" stroke-width="1.4"/>
<text x="372.5" y="174" text-anchor="middle" fill="#8a5a00" font-size="11.5" font-family="SFMono-Regular,Menlo,Consolas,monospace" stroke="#fdf0d5" stroke-width="4" stroke-linejoin="round" style="paint-order:stroke">add_custom_tool(&quot;rename_file&quot;, code)</text>
<path d="M590,209 H726" stroke="#8a5a00" stroke-width="1.6" fill="none" marker-end="url(#sq-call)"/>
<text x="660.0" y="203" text-anchor="middle" fill="#8a5a00" font-size="12" stroke="#fdf0d5" stroke-width="4" stroke-linejoin="round" style="paint-order:stroke">saves rename_file.py</text>
<path d="M155,238 H586" stroke="#8a5a00" stroke-width="1.6" fill="none" marker-end="url(#sq-call)"/>
<circle cx="280" cy="238" r="3.5" fill="#fff" stroke="#8a5a00" stroke-width="1.4"/>
<text x="372.5" y="232" text-anchor="middle" fill="#8a5a00" font-size="11.5" font-family="SFMono-Regular,Menlo,Consolas,monospace" stroke="#fdf0d5" stroke-width="4" stroke-linejoin="round" style="paint-order:stroke">display_ui_to_user(html)</text>
<path d="M590,267 H404" stroke="#8a5a00" stroke-width="1.6" fill="none" stroke-dasharray="5 3" marker-end="url(#sq-call)"/>
<text x="495.0" y="261" text-anchor="middle" fill="#8a5a00" font-size="12" stroke="#fdf0d5" stroke-width="4" stroke-linejoin="round" style="paint-order:stroke">HTML, rendered by the host</text>
<text x="8" y="314" text-anchor="start" fill="#1a4f8a" font-size="12.5" font-weight="700">2 · The user works in the app, no LLM involved</text>
<path d="M45,340 H396" stroke="#1a4f8a" stroke-width="1.6" fill="none" marker-end="url(#sq-user)"/>
<text x="222.5" y="334" text-anchor="middle" fill="#1a4f8a" font-size="12" stroke="#e7effa" stroke-width="4" stroke-linejoin="round" style="paint-order:stroke">renames IMG_0412.jpg → kayak.jpg</text>
<path d="M400,369 H586" stroke="#1a4f8a" stroke-width="1.6" fill="none" marker-end="url(#sq-user)"/>
<text x="495.0" y="363" text-anchor="middle" fill="#1a4f8a" font-size="11.5" font-family="SFMono-Regular,Menlo,Consolas,monospace" stroke="#e7effa" stroke-width="4" stroke-linejoin="round" style="paint-order:stroke">run_custom_tool(&quot;rename_file&quot;, …)</text>
<path d="M590,398 H726" stroke="#1a4f8a" stroke-width="1.6" fill="none" marker-end="url(#sq-user)"/>
<text x="660.0" y="392" text-anchor="middle" fill="#1a4f8a" font-size="11.5" font-family="SFMono-Regular,Menlo,Consolas,monospace" stroke="#e7effa" stroke-width="4" stroke-linejoin="round" style="paint-order:stroke">os.rename(…)</text>
<path d="M590,427 H404" stroke="#1a4f8a" stroke-width="1.6" fill="none" stroke-dasharray="5 3" marker-end="url(#sq-user)"/>
<text x="495.0" y="421" text-anchor="middle" fill="#1a4f8a" font-size="12" stroke="#e7effa" stroke-width="4" stroke-linejoin="round" style="paint-order:stroke">ok</text>
<text x="8" y="474" text-anchor="start" fill="#1d6b3f" font-size="12.5" font-weight="700">3 · The app tells the chat</text>
<path d="M400,500 H284" stroke="#1d6b3f" stroke-width="1.6" fill="none" marker-end="url(#sq-result)"/>
<text x="340.0" y="494" text-anchor="middle" fill="#1d6b3f" font-size="11.5" font-family="SFMono-Regular,Menlo,Consolas,monospace" stroke="#e3f3e8" stroke-width="4" stroke-linejoin="round" style="paint-order:stroke">sendMessage(&quot;renamed to kayak.jpg&quot;)</text>
<path d="M280,529 H159" stroke="#1d6b3f" stroke-width="1.6" fill="none" marker-end="url(#sq-result)"/>
<text x="217.5" y="523" text-anchor="middle" fill="#1d6b3f" font-size="12" stroke="#e3f3e8" stroke-width="4" stroke-linejoin="round" style="paint-order:stroke">arrives as a user message</text>
<text x="8" y="576" text-anchor="start" fill="#8a5a00" font-size="12.5" font-weight="700">4 · The agent can call the same tool</text>
<path d="M45,602 H276" stroke="#1a4f8a" stroke-width="1.6" fill="none" marker-end="url(#sq-user)"/>
<text x="162.5" y="596" text-anchor="middle" fill="#1a4f8a" font-size="12" stroke="#fdf0d5" stroke-width="4" stroke-linejoin="round" style="paint-order:stroke">“now do the other kayak photos”</text>
<path d="M280,631 H159" stroke="#666" stroke-width="1.6" fill="none" marker-end="url(#sq-grey)"/>
<text x="217.5" y="625" text-anchor="middle" fill="#666" font-size="12" stroke="#fdf0d5" stroke-width="4" stroke-linejoin="round" style="paint-order:stroke">prompt</text>
<path d="M155,660 H586" stroke="#8a5a00" stroke-width="1.6" fill="none" marker-end="url(#sq-call)"/>
<circle cx="280" cy="660" r="3.5" fill="#fff" stroke="#8a5a00" stroke-width="1.4"/>
<text x="372.5" y="654" text-anchor="middle" fill="#8a5a00" font-size="11.5" font-family="SFMono-Regular,Menlo,Consolas,monospace" stroke="#fdf0d5" stroke-width="4" stroke-linejoin="round" style="paint-order:stroke">run_custom_tool(&quot;rename_file&quot;, …)</text>
<path d="M590,689 H726" stroke="#8a5a00" stroke-width="1.6" fill="none" marker-end="url(#sq-call)"/>
<text x="660.0" y="683" text-anchor="middle" fill="#8a5a00" font-size="11.5" font-family="SFMono-Regular,Menlo,Consolas,monospace" stroke="#fdf0d5" stroke-width="4" stroke-linejoin="round" style="paint-order:stroke">os.rename(…) ×3</text>
<path d="M155,718 H586" stroke="#8a5a00" stroke-width="1.6" fill="none" marker-end="url(#sq-call)"/>
<circle cx="280" cy="718" r="3.5" fill="#fff" stroke="#8a5a00" stroke-width="1.4"/>
<text x="372.5" y="712" text-anchor="middle" fill="#8a5a00" font-size="11.5" font-family="SFMono-Regular,Menlo,Consolas,monospace" stroke="#fdf0d5" stroke-width="4" stroke-linejoin="round" style="paint-order:stroke">notify_ui(channel, &quot;refresh&quot;)</text>
<path d="M590,747 H404" stroke="#8a5a00" stroke-width="1.6" fill="none" stroke-dasharray="5 3" marker-end="url(#sq-call)"/>
<text x="495.0" y="741" text-anchor="middle" fill="#8a5a00" font-size="12" stroke="#fdf0d5" stroke-width="4" stroke-linejoin="round" style="paint-order:stroke">picked up by the app’s next poll</text>
<circle cx="14" cy="790" r="3.5" fill="#fff" stroke="#666" stroke-width="1.4"/>
<text x="24" y="794" text-anchor="start" fill="#666" font-size="11">the host relays the agent’s call and shows it in the chat; the app’s calls are relayed too, but not shown</text>
</g>
</svg>
</details>

I see these capabilities as a bit of a pyramid, with dynamic UI at the bottom, and increasing power and capabilities having fewer and fewer applications.

<svg viewBox="0 0 780 300" role="img" aria-label="A pyramid of capabilities. From the bottom: dynamic UI, talking to the agent, acting on your machine with custom tools, and saved apps at the top. Above it, a dashed box: deployed elsewhere, by hand for now. An arrow on the right points up: more power, fewer uses." style="max-width:100%;height:auto;display:block;margin:1.5rem 0">
<defs>
<marker id="py-user" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#1a4f8a"/></marker>
<marker id="py-call" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#8a5a00"/></marker>
<marker id="py-result" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#1d6b3f"/></marker>
<marker id="py-wait" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#a8322d"/></marker>
<marker id="py-grey" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#666"/></marker>
</defs>
<g font-family="-apple-system,BlinkMacSystemFont,Segoe UI,Helvetica,Arial,sans-serif">
<polygon points="0.0,290.0 600.0,290.0 535.0,234.0 65.0,234.0" fill="#e7effa" stroke="#fff" stroke-width="2"/>
<text x="300" y="258.0" text-anchor="middle" fill="#1a4f8a" font-size="13" font-weight="700">Dynamic UI</text>
<text x="300" y="275.0" text-anchor="middle" fill="#1a4f8a" font-size="11" font-family="SFMono-Regular,Menlo,Consolas,monospace">display_ui_to_user</text>
<polygon points="65.0,234.0 535.0,234.0 470.0,178.0 130.0,178.0" fill="#cfdff3" stroke="#fff" stroke-width="2"/>
<text x="300" y="202.0" text-anchor="middle" fill="#1a4f8a" font-size="13" font-weight="700">Talking to the agent</text>
<text x="300" y="219.0" text-anchor="middle" fill="#1a4f8a" font-size="11" font-family="SFMono-Regular,Menlo,Consolas,monospace">notify_ui · notify_agent</text>
<polygon points="130.0,178.0 470.0,178.0 405.0,122.0 195.0,122.0" fill="#9dbde4" stroke="#fff" stroke-width="2"/>
<text x="300" y="146.0" text-anchor="middle" fill="#0f3560" font-size="13" font-weight="700">Acting on your machine</text>
<text x="300" y="163.0" text-anchor="middle" fill="#0f3560" font-size="11" font-family="SFMono-Regular,Menlo,Consolas,monospace">add_custom_tool · run_custom_tool</text>
<polygon points="195.0,122.0 405.0,122.0 340.0,66.0 260.0,66.0" fill="#1a4f8a" stroke="#fff" stroke-width="2"/>
<text x="300" y="90.0" text-anchor="middle" fill="#fff" font-size="13" font-weight="700">Saved apps</text>
<text x="300" y="107.0" text-anchor="middle" fill="#fff" font-size="11" font-family="SFMono-Regular,Menlo,Consolas,monospace">load_app</text>
<rect x="222" y="16" width="156" height="40" rx="4" fill="none" stroke="#1a4f8a" stroke-dasharray="5 3"/>
<text x="300" y="33" text-anchor="middle" fill="#1a4f8a" font-size="12" font-weight="600">Deployed elsewhere</text>
<text x="300" y="48" text-anchor="middle" fill="#666" font-size="11">by hand, for now</text>
<path d="M640,284 V30" stroke="#666" stroke-width="1.6" fill="none" marker-end="url(#py-grey)"/>
<text x="652" y="40" text-anchor="start" fill="#555" font-size="12">more power,</text>
<text x="652" y="56" text-anchor="start" fill="#555" font-size="12">fewer uses</text>
<text x="652" y="268" text-anchor="start" fill="#555" font-size="12">every</text>
<text x="652" y="284" text-anchor="start" fill="#555" font-size="12">conversation</text>
</g>
</svg>

Lastly, at the top of the pyramid is the capability to save an application for the future. A useful app that the user and the agent chiseled together should not have to be recreated each time. The next logical step after that may be to deploy an app off of the local machine. This is how [Caribbean Fish Recall](https://caribbean-fish-recall-production.up.railway.app) originated. I did not provide for automation of this, although the agent was able to peel the code away ([pslusarz/caribbean-fish-recall](https://github.com/pslusarz/caribbean-fish-recall)) and deploy it to Railway with a few instructions.

If you would like to try it for yourself, it is verified to work in GitHub Copilot in VS Code, and in Claude Cowork (not Claude Code!) in the Claude desktop app on a Mac. Point your agent to the repo, [pslusarz/lustereczko-mcp](https://github.com/pslusarz/lustereczko-mcp), and tell it to install it as a local MCP server.

Thank you for reading this far, and let me know your thoughts.
