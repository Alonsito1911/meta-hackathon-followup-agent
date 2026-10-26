# Project Context

> Working context for this repo: goals, rules, decisions, and open questions.
> Written for the author and for any coding assistant helping on the project.
> Project name: TBD. Update this file as decisions change.

## 1. What this is

A submission to the **Meta Global AI Developer Hackathon** (Cerebral Valley x Meta).

- Building period: **Oct 27 - Nov 4, 2026**
- **Submission deadline: Wed Nov 4, 9:00 PM ET** (23:00 in Chile). Late submissions are not considered. Target: submit on **Tue Nov 3**.
- Participation: solo (max team size is 4). Likely track: Indie Developer (confirm on the submission form).
- Problem statement we fit: **Build for Your World** (solve a problem known firsthand using Meta's models).

## 2. The idea (one sentence)

A voice agent that handles calls for a public health system (confirming appointments, rescheduling, structured follow-up check-ins) and gives the human team a **ranked, explainable list of who to contact first**, reducing their workload.

### What it does NOT do (important for the rules)

- It does **not** give medical advice, diagnose, or interpret symptoms.
- It only receives calls, records information, and orders work for staff.
- Any alarm signal mentioned by a patient is **escalated to a human**; the agent only says a team member will follow up.
- All demo data is **synthetic**. No real patient data anywhere.

The hackathon bans "any project using AI to generate and give medical advice". Everything in the README, demo, and UI must frame this as **operational support for staff**, not a health assistant. Avoid words like "AI doctor", "detects risk", "cares for your health".

## 3. Three modules (one flow)

1. **Mass scheduling / confirmation**: outbound calls to confirm or reschedule appointments and refill released slots.
2. **Patient follow-up**: structured check-ins (attendance, medication adherence, how they have been feeling), recorded as data, not interpreted.
3. **Prioritization**: a rules-based, explainable score that orders the staff's call list, with the reason for each position.

Plan: demo one module in depth, show the others as modules of the same loop. Decide which is the core after the first working end-to-end prototype.

## 4. Problem statement (evidence, mostly Chile)

Use these figures with their sources. Prefer Chilean data; it is the strongest.

- Non-GES waiting list at **31 Mar 2026: 2,971,312 records** (2,513,203 specialty consults + 458,109 surgeries). Source: Minsal.
- About **850k** consult requests have waited **over a year**, **85k over three years**. Source: The Clinic (Aug 31, 2026).
- Median wait is about **8 months** for specialty consults. Source: Minsal.
- **19%** of specialty consults are lost to unannounced no-shows, about **1.4M per year** (Minsal REM 2019, cited in the FONDEF report).
- 2022: **7,575,359** specialty consults scheduled, **1,185,393** no-shows. Source: U. de Chile thesis.
- Primary care (Renca): cardiovascular control visits with a doctor had **15.4%** no-shows; psychologist visits **22.6%**.
- Reminders by call, text or WhatsApp reduced absenteeism by roughly **5-8 percentage points**, depending on channel (FONDEF project).
- Chile's cardiovascular program (PSCV) had **2,546,345** registered hypertensive patients (2018). Effective coverage in the national health survey: 33.4% (hypertension) and 55.4% (type 2 diabetes); these are old figures, look for newer ones before citing.
- Americas: only **36%** of adults with hypertension have it controlled; a third do not know they have it (PAHO).
- US: under half of hypertensive adults have it controlled (48.3%, CDC/NHANES 2015-16).

Caveats:
- The widely repeated "US$150B per year lost to US no-shows" traces back to a 2020 industry study, not an official source. Avoid it or cite it as such.
- Evidence about voice agents reducing no-shows comes from vendor blogs. **Do not cite it.** Present the agent as a hypothesis to test.
- No direct evidence found on how staff prioritize today. Treat as a hypothesis or validate with someone from a primary care center.

## 5. Meta models and APIs (verified by search; test before relying)

### Meta Model API (closed models)
- OpenAI-compatible. Base URL: `https://api.meta.ai/v1`
- Key in env var `MODEL_API_KEY` (the OpenAI SDK does not read it automatically; pass it explicitly).
- Default model: `muse-spark-1.3` (1,048,576-token context). Supports structured output, parallel tool calling, vision.
- Muse Spark does **not** accept audio input/output, so voice goes through Voice Transcribe.
- Official recipes: github.com/meta-models/meta-model-cookbook

```python
import os
from openai import OpenAI

client = OpenAI(base_url="https://api.meta.ai/v1", api_key=os.environ["MODEL_API_KEY"])
resp = client.chat.completions.create(
    model="muse-spark-1.3",
    messages=[{"role": "user", "content": "Hello, world!"}],
)
print(resp.choices[0].message.content)
```

### Muse Voice Transcribe (speech to text)
- Streaming ASR over WebSocket: `wss://api.meta.ai/v1/asr/realtime`
- LiveKit plugin: `pip install livekit-plugins-meta` (`MODEL_API_KEY` or `META_API_KEY`). Expects mono PCM16 at 24 kHz, keep sending audio (including silence), max session 60 minutes.
- Supports Spanish (also keyword biasing and language biasing). **Must test with Chilean Spanish audio early.**
- File transcription path exists, but the exact endpoint was not confirmed in official docs. OpenRouter also serves it at `/api/v1/audio/transcriptions` (base64 audio) as a fallback.
- No text-to-speech model was found in Meta's list. The agent's spoken output likely needs browser speech synthesis or another service. **Decision pending.**

### Muse Spark Open via Baseten (for the open-weights bonus prize)
- OpenAI-compatible: base URL `https://inference.baseten.co/v1`, key `BASETEN_API_KEY`.
- Public docs use `meta-models/muse-spark-open`. **For this hackathon use `meta-models/muse-spark-open-1026`** (per the participant guide).
- Bonus prize "Best Build on Meta Open Weight Models" ($10K) requires the core component to run on Muse Glimmer or Muse Spark Open via your own hardware or a partner (Baseten/Modal/Fireworks).

### Credits to claim (each person individually, by Nov 4, 9:00 PM ET)
- Meta Model API: $150 (needs a credit card on file; temporary pre-authorization is released). Set a spend-limit alert in Billing.
- Baseten: $100 + 2M TPM (request, reviewed).
- Modal: $100 (first 1,000 participants).
- Fireworks: $25 (shortlisted teams only).

## 6. Proposed architecture (draft)

```
[Browser mic / simulated call]
        |
        v
Muse Voice Transcribe (streaming ASR, Spanish)
        |
        v
Agent loop (Muse Spark / Muse Spark Open): scripted questions, tool calls
        |
        v
Structured extraction -> JSON fields (no interpretation)
        |
        v
Rules-based priority score (explainable) + escalation flag
        |
        v
Staff dashboard: ranked list, reason per position, transcript
```

Design rules:
- The ranking is **rules plus extracted fields**, not "the model decides". Every position must show why.
- Escalation of alarm signals to a human is deterministic and visible in the demo.
- Simulating the call in the browser is enough for the demo; real telephony is optional.

## 7. Hackathon rules that shape the work

- **Repo must be public** from the start.
- **New work only**: everything shown must be built during the event (Oct 27 - Nov 4). Make it clear what is new; unclear contributions risk **disqualification**. Do not reuse code from earlier voice-agent or health projects. If an earlier idea is reused, declare it in the README.
- Commit early and often; the history is evidence.
- Banned or generic projects: AI medical advice, basic RAG apps, simple image analyzers, simple education chatbots, job-application screeners, nutrition coaches, personality analyzers, sports analyzers, anything political or involving war.
- Must genuinely use Meta's models (Stage 1 is pass/fail on theme fit and real use of Meta Model API or Meta open weights).

## 8. Judging

- Stage 1: pass/fail (fits theme, really uses Meta models).
- Stage 2 (0-10): Impact 30%, Use of Meta AI tooling 25%, Demo 25%, Technical depth and execution 20%.
- Stage 3: top 5 per track judged live.
- Likely bonus targets: Best Agentic Use-Case ($5K), Best Build on Open Weights ($10K), Most Viral ($20K, needs an X post with the demo).

## 9. Submission checklist

- [ ] Public repo, README with problem, solution, architecture, models used, how to run
- [ ] README section: **"Built during the event"** and **"Prior ideas (if any)"**
- [ ] **3-minute demo video**, public link that opens without signing in (test in an incognito window)
- [ ] First 20 seconds state the problem, what it does, and which Meta models it uses
- [ ] Demo includes a routine call and a call with an alarm signal escalated to a human
- [ ] On-screen note that data is synthetic
- [ ] All participants listed on the submission page
- [ ] Submitted through the Cerebral Valley platform (link is in section 7 of the participant guide)
- [ ] Optional: 30-45 second clip posted on X with the submission link for the viral prize

Language: video narration, README, UI text and the X post in **English**. Demo calls can be in Chilean Spanish with English subtitles, if transcription works well.

## 10. Plan

| When | Goal |
|---|---|
| Oct 26 | Claim credits, join Discord, test Voice Transcribe with Spanish audio, first Muse Spark call |
| Oct 27-28 | Repo created from scratch, minimal flow: audio -> transcript -> JSON -> score. Watch Agents 101 (Oct 27) and Agents 102 (Oct 28) |
| Oct 29-31 | Dashboard, escalation, synthetic patients, Baseten integration |
| Nov 1-2 | Polish, record demo takes, write README |
| Nov 3 | Submit |
| Nov 4 | Buffer only |

## 11. Open questions

- Which module is the core of the demo?
- Does Voice Transcribe handle Chilean Spanish well enough, or should the demo be in English?
- How will the agent speak (TTS)? Browser speech synthesis or another service?
- How are the track and team members entered on the submission form?
- Does the core reasoning run on Muse Spark Open (Baseten) to qualify for the open-weights prize?

## 12. Sources

- Minsal, non-GES waiting list: https://www.minsal.cl/?p=151868
- The Clinic, waiting list over one year: https://www.theclinic.cl/2026/08/31/la-primera-prueba-del-minsal-en-listas-de-espera-retrasos-ges-presentan-baja-inedita-pero-cirugias-atrasadas-aumentaron-y-ya-superan-las-475-mil/
- FONDEF report on no-shows: https://dev.eventos.cmm.uchile.cl/fondefnsp/wp-content/uploads/sites/138/2022/07/Resumen-Ejecutivo-FONDEF-NSP_02_v29jun2022.pdf
- U. de Chile, specialty no-shows 2022: https://repositorio.uchile.cl/handle/2250/203932
- U. de Chile, no-shows in Renca primary care: https://repositorio.uchile.cl/handle/2250/203949
- PAHO, HEARTS in Chile: https://paho.org/sites/default/files/hearts-2019-paises-1-cohorte-chile-carolina-neira-4es_0.pdf
- PAHO, hypertension in primary care (2025): https://www.paho.org/es/noticias/16-5-2025-abordar-hipertension-atencion-primaria-salud-es-clave-para-reducir-muertes
- Meta Model API cookbook: https://github.com/meta-models/meta-model-cookbook
- LiveKit plugin for Muse Voice Transcribe: https://pypi.org/project/livekit-plugins-meta/
- Muse Spark Open on Baseten: https://www.baseten.co/library/muse-spark-open/
