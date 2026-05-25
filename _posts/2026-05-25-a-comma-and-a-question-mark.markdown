---
title: 'A Comma and a Question Mark Redux'
date: 2026-05-25 15:00:00Z
layout: post
description: Two shell commands — a comma and a q — that give my terminal a little extra help.
---

I am a decent user of the terminal, but I am not strong at remembering `find` flags - or `rsync`, or `grep` for that matter.  

I read [Rémi Louf's post](https://www.thetypicalset.com/blog/a-comma-and-a-question-mark) about wiring a comma and a question mark into his shell and immediately wanted the same thing. The idea is simple: type `, <description>` and get a shell command that does what you described. Type `? <question>` and get an AI answer right in your terminal.

Rémi runs a local Qwen model through llama.cpp. I don't have a local model, but I do have [pi](https://pi.dev),  a CLI chat agent,  and I have my routing  set through OpenRouter. Pi was already configured and working on my machine. So I took the idea and adapted it.

## The comma 

When I want nice shell commands, now all I have to do is type a comma followed by a plain English description of what I want to do. A few seconds later I get a suggested command copied to my clipboard. For example:

```
, find the 5 largest files in the current directory
```

A second later:

```
ls -lS
```

is copied to my clipboard. I press Cmd+V, the command lands on my prompt line. I read it, maybe edit it, then press Enter myself.

```
$ ls -lS
```

The comma is slightly safe because it won't automatically execute. It copies to my clipboard and prints the command - so that I can judge and apply the keystroke between "here's a suggestion" and "yes, do that".

Under the hood it's a thin shell script in `~/.dotfiles/bin/,` on my PATH:

<pre><code>#!/usr/bin/env zsh
local command
command=$(pi --print -p --no-tools --thinking off \
  --system-prompt "output exactly one shell command —
  the best one — with no numbering, no explanation,
  no markdown, no backticks. Just the raw command
  on a single line." "$desc" 2>/dev/null)

echo -n "$command" | pbcopy
echo "$command"
</code></pre>

Pi uses whichever model is currently selected. Typically for me this is on OpenRouterusing DeepSeek v4 Flash or Gemini 3.5 Flash. These have low or free API cost.

I skipped the JSON Schema trick from the original. I don't think pi exposes a structured output mode, so I just made the prompt tight and stripped backticks in post.

## The question mark 

Likewise, when I just have a short question, I now have the `q` script. Instead of launching a whole `pi` session, I can get a quick answer with minimal fuss: 

```
q what's the weather like today in Brutus, MI?
```

The `q` command also invokes Pi, but now Pi can use a narrow toolset including from some extensions:

<pre><code>pi --print -p \
  --system-prompt "You are a helpful, concise assistant
  running in a macOS terminal. Answer clearly and accurately.
  You can read files from disk and search the web — use those
  when you need current or file-specific information." \
  --tools "read,web_search,url_extract,web_fetch,batch_web_fetch" \
  "$question"
</code></pre>


You can find the script files [in my dotfiles repo on GitHub](https://github.com/z3ugma/.dotfiles/commit/a1453d02b3da53af4a442f45ddc3a938406cc5d2). 