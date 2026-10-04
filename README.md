# Does Your Family Need a Local AI?

**A private, self-hosted AI assistant for a whole family, built on a base Mac mini M4 in a walk-in-closet office.**

My family uses AI every day. My dad is building businesses and kept hitting usage limits on paid AI plans. My sister runs a nonprofit and handles information that shouldn't be pasted into a public chatbot. So I built our own: one small computer at home, running open-source AI models, reachable from any family member's phone or laptop through a private, login-protected website.

No monthly subscription. No usage caps. Our conversations stay on hardware we own.

This repo documents what I used, every choice I made and why, the problems I hit, and how to build it yourself.

---

## Who this is for

- **Families** who want a shared AI assistant without everyone paying for their own subscription, and without their questions living on someone else's servers.
- **Small businesses and nonprofits** that want AI help with drafting, brainstorming and paperwork while keeping internal information in-house.
- **Anyone planning for the long term.** A family AI server is family infrastructure, like a shared photo archive or a home network. If you treat it that way, document it that way: who has admin access, where the credentials live, and how it gets handed off. That belongs in the same place you keep the rest of your digital-asset planning. (Talk to an estate attorney for the legal side. This repo covers the technical side.)

---

## How it works

```
Family member's phone or laptop
            │  HTTPS
            ▼
   Cloudflare Access  ── email one-time PIN; only invited addresses get in
            │
            ▼
   Cloudflare Tunnel  ── outbound-only connection; no ports opened on the home router
            │
            ▼
  ┌─────────────────── Mac mini M4 (16GB) ───────────────────┐
  │                                                           │
  │   Open WebUI (Docker, bound to 127.0.0.1:3000)            │
  │     accounts, chat history, custom assistants             │
  │                 │                                         │
  │                 ▼                                         │
  │   LM Studio server (127.0.0.1:1234)                       │
  │     runs the AI model on the Mac's GPU                    │
  └───────────────────────────────────────────────────────────┘
```

---

## Hardware

| Device | Role | Why |
|---|---|---|
| **Mac mini M4, base, 16GB** | The server. Runs everything. | Quiet, low power, always on. Apple Silicon shares one pool of memory between CPU and GPU, so a 16GB machine can run models a 16GB PC with a small graphics card can't. |
| MacBook Air | Day-to-day admin and editing | Keeps the server free for serving. |
| Alienware 15 R3 (GTX 1060 6GB) | Not used for AI | 6GB of VRAM is too little for the models worth running. |

**The real constraint is memory.** Of 16GB, about 11GB is practically available for models after macOS and the apps. Every model and setting choice below follows from that number.

---

## Software stack

| Layer | Tool | Why I chose it |
|---|---|---|
| Model server | **LM Studio** | Free, Mac-native, easy model downloads, an OpenAI-compatible API, and both GGUF and MLX engines. |
| Chat interface | **Open WebUI** (Docker) | A ChatGPT-style interface with per-person accounts, private chat history, admin controls, and custom assistants. |
| Remote access | **Cloudflare Tunnel** | Reaches the Mac without opening ports on the home router or exposing my home IP address. |
| Login gate | **Cloudflare Access** | Only email addresses I approve can even reach the login page. Family members get a one-time PIN by email. |

---

## Models

### Daily driver: Qwen 3.5 9B (Unsloth MTP GGUF, `UD-Q4_K_XL`, ~8GB)

Exposed to the family as a custom assistant called **"Qwik."**

- **Why 9B:** the largest model class that fits in ~11GB with room left for conversation memory. Smaller models are fast but unreliable. In testing, a 0.5B model confidently said there are 7 planets and invented a "Super-Earth."
- **Why a 4-bit quantization:** shrinks the model to fit in memory with little loss in quality for everyday use.
- **Why GGUF instead of MLX:** MLX is normally the faster format on a Mac. I switched to GGUF because the fix for a thinking-mode bug (below) works on GGUF but not MLX, and that bug was the single biggest thing slowing the assistant down.
- **About the MTP build and speculative decoding:** I started with the MTP (multi-token prediction) version, expecting a free speed boost from its built-in self-drafting. On a base M4 it turned out to be a net loss (see the benchmark below), so I run it with speculative decoding **off**. A plain GGUF of the same model would work just as well.
- It also reads images, so family members can ask it about photos and screenshots.

### Occasional use: Gemma 4 12B (GGUF)

A second model for image-heavy questions. **It can't be loaded at the same time as Qwen on 16GB**, so it's swapped in by hand when needed.

### Current LM Studio load settings (Qwen)

| Setting | Value | Reason |
|---|---|---|
| Context length | 16,384 | Long enough for real conversations, small enough to fit in memory. |
| GPU offload | Max | On Apple Silicon, the whole model should run on the GPU. Anything less is much slower. |
| Max concurrent predictions | 4 | Lets several family members chat at once. Each slot costs memory. |
| Speculative decoding | **Off** | Tested on a base M4 and found to be a net loss. See the benchmark. |

---

## Benchmark: does MTP actually help on a base M4?

All runs: same prompt ("give me 5 names for a lawn care business"), Qwen 3.5 9B GGUF (~8GB), full GPU offload, thinking off.

| Configuration | Speed | Time to first word | Draft tokens accepted |
|---|---|---|---|
| MTP on, 3 draft tokens | 12.53 tok/s | 0.88 s | 61.0% |
| MTP on, 2 draft tokens | 15.06 tok/s | 0.43 s | 53.5% |
| **MTP off** | **15.12 tok/s** | **0.42 s** | — |

**Conclusion: MTP made it slower on this hardware.** At best it tied having it off; at worst it was ~17% slower. With only ~60% of drafts accepted, the time spent verifying wrong guesses cost more than the right guesses saved. Speculative decoding helps most on GPUs with memory bandwidth to spare; a base M4 doesn't have the headroom. **I run with it off.**

The bigger lesson was that **turning thinking off mattered far more than any decoding trick.** Before the fix, the model spent 13 seconds "thinking" before saying hello. After it, replies start in under half a second. That one change is what took the assistant from sluggish to genuinely fast.

### Real-world result

Tested from a phone on cellular (T-Mobile 5G), away from home Wi-Fi: answers come back quickly enough to feel comparable to a commercial chatbot's *speed*. It's a 9B model, so it isn't as capable as a frontier cloud model on hard problems, but for everyday questions, drafting, and brainstorming it's fast, accurate, and completely private.

---

## Build it yourself

### 1. LM Studio

1. Install LM Studio, download your model, and load it with the settings above.
2. In the **Developer** tab, start the server (default `http://127.0.0.1:1234`).
3. Turn on auto-start for the server and auto-load for your model, so a reboot brings everything back on its own.

### 2. Open WebUI

```bash
docker run -d -p 127.0.0.1:3000:8080 \
  --add-host=host.docker.internal:host-gateway \
  -e OPENAI_API_BASE_URL=http://host.docker.internal:1234/v1 \
  -e OPENAI_API_KEY=lm-studio \
  -e ENABLE_OLLAMA_API=false \
  -v open-webui:/app/backend/data \
  --name open-webui --restart always \
  ghcr.io/open-webui/open-webui:main
```

- `127.0.0.1:3000` means only the Mac itself can reach Open WebUI. Without it, anyone on your Wi-Fi could skip the Cloudflare login entirely.
- The API key is a placeholder. LM Studio doesn't check it.
- All accounts and chats live in the `open-webui` Docker volume, so you can delete and recreate the container without losing anything.

Open `http://localhost:3000` and create your account. **The first account becomes the admin.**

### 3. Cloudflare Tunnel

You need a domain whose DNS is managed by Cloudflare.

1. In the Cloudflare **Zero Trust** dashboard, go to **Networks → Tunnels** and create a tunnel using `cloudflared`.
2. Install `cloudflared` on the Mac and run the install command the dashboard gives you. That sets it up as a service that starts automatically.
3. Add a public hostname, such as `ai.yourdomain.com`, pointing to `http://localhost:3000`.

### 4. Cloudflare Access

1. In Zero Trust, go to **Access → Applications** and add a **self-hosted** application for `ai.yourdomain.com`.
2. Add an **Allow** policy that includes only your family's email addresses.
3. Use **One-time PIN** as the login method.

Anyone not on the list never reaches your server.

### 5. Make models available to the family

Open WebUI hides models from regular users by default. Admins can see everything, so it will look fine to you and still be broken for everyone else.

- **Admin Panel → Settings → Models:** set the base model to **Public**.
- **Workspace → Models:** create a custom assistant (mine is "Qwik"), choose the base model, and set it to **Public** too.

**Both entries must be Public.** Open WebUI checks permissions on the custom assistant *and* on the model underneath it.

### 6. Give the assistant the date

Local models don't know today's date. In the custom assistant's system prompt:

```
Today is {{CURRENT_WEEKDAY}}, {{CURRENT_DATE}}. The current time is {{CURRENT_TIME}}.
You are talking with {{USER_NAME}}. Give clear, direct answers.
Keep responses concise unless asked for detail.
```

### 7. Keep it running

On the Mac: prevent automatic sleep, start Docker at login, and launch LM Studio at login with its server on auto-start. Then **reboot once as a real test**. If everything comes back up and a chat works from a phone, you're done.

---

## Problems I hit (and fixes)

**"Model not found" for everyone but me.** Admins bypass access checks, which hid the problem. The fix was making both the custom assistant *and* its base model Public.

**"Insufficient system resources" loading a second model.** This is LM Studio's memory guardrail doing its job: two models of this size won't fit in 16GB at once. Don't loosen it. Swap models instead.

**Thinking mode wouldn't turn off (this was the big one).** Qwen 3.5 "thinks" before answering, which took 13 seconds just to say hello. The LM Studio sidebar toggle only affected LM Studio's own chat window — requests coming from Open WebUI still triggered thinking. The `/no_think` command did nothing either. This is a known LM Studio issue with Qwen 3.5.

The fix that worked everywhere: edit the model's **chat template** (in LM Studio, **Load → Advanced Load Params → Chat Template**, GGUF only) and add this as the very first line, then eject and reload:

```
{%- set enable_thinking = false %}
```

The chat template rewrites every request before the model sees it, so this forces thinking off no matter which app or assistant sends the prompt. (Because the line forces thinking off unconditionally, typing `/think` in a message does not turn it back on.)

**Speculative decoding was greyed out.** A draft model must share the main model's vocabulary (same family), and LM Studio doesn't support speculative decoding on batched MLX models. MTP models solve both problems by drafting for themselves.

**A tiny model looked amazing at 231 tokens/sec.** That was the 0.5B draft model accidentally loaded as the main model, answering on its own, and badly. Never make a draft model available to users.

**GPU offload set itself low.** LM Studio may lower GPU offload based on its memory estimate. If the model won't load at full offload, reduce context length instead.

---

## What this is *not*

Being honest about limits matters more than the build itself.

- **Self-hosted is not automatically compliant.** Cloudflare decrypts traffic at its edge before forwarding it to your Mac, and the server admin can technically read any chat. That's fine for family use and general business work. It is **not** a substitute for HIPAA, 42 CFR Part 2, or other regulated-data compliance. Keep identifiable patient, client or legal information out unless a compliance professional has approved the setup.
- **It's not as smart as frontier cloud AI.** A 9B local model handles everyday questions, drafting and brainstorming well. For hard reasoning or large documents, a paid frontier model is still better. The two work well side by side.
- **It's not image generation.** Running Stable Diffusion well needs a dedicated NVIDIA GPU with 12GB+ of VRAM.

---

## Roadmap

- [x] Benchmark speculative decoding / MTP on a base M4 (see above) and publish the numbers.
- [x] Get thinking mode off by default so everyday answers are fast.
- [ ] A separate "thinking" assistant for reasoning-heavy work (business taxes, math) while the default stays fast — same base model, thinking turned back on for that assistant only.
- [ ] Better read-aloud voices (configure Open WebUI text-to-speech, including female voice options).
- [ ] Single sign-on: let Open WebUI trust Cloudflare's verified email (`WEBUI_AUTH_TRUSTED_EMAIL_HEADER=Cf-Access-Authenticated-User-Email`) so family members log in once instead of twice.
- [ ] Add a web search provider so the assistant can look things up.
- [ ] Hardware upgrade: a Mac mini with 32GB of memory to run larger models or two models at once.
- [ ] A dedicated NVIDIA GPU machine for local image generation.

---

## About

Built by Bernard Mathis Jr from a walk-in-closet home office, for a family of everyday AI users.

If you're thinking about building one for your family, business or nonprofit, open an issue. I'm happy to help.
