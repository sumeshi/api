# Do Local LLMs Dream of Forensic Investigators?
On the design philosophy behind FORENSIA, an AI digital forensics harness.

## Introduction

Strangely enough, requests to investigate security incidents **tend to arrive on Fridays** (or right before long holidays).

No organization wants a security incident, but I suspect it often plays out something like this:

- Late on a weekend night, when nobody is at the office, an attacker breaks into the system.
- On Monday, someone notices the system acting strange, but without any certainty, they decide to sleep on it.
- On Tuesday, "the system might be behaving oddly" gets shared at the morning meeting, and an internal investigation begins.
- By the end of Wednesday, they finally accept that this is not just an outage — it looks like an incident.
- On Thursday, they decide to consult a security vendor, and your company gets the call. But there's latency between the front office and the engineers.
- On Friday morning, the request finally reaches you. Mysteriously, a report is already promised for Monday. The data to investigate hasn't even arrived yet.

The system is under threat *right now*. You need to grasp the situation, find the root cause, and take countermeasures as fast as possible.

At the same time, incident response is much more than investigation: containment, recovery, coordinating with stakeholders. **If you concentrate your limited staff on investigation alone, other critical work falls behind.**
And yet, if you keep working straight through weekends and weekdays alike, fatigue will inevitably degrade the quality of your judgment.

Wouldn't it be nice to have someone who could organize evidence and test hypotheses — tirelessly, sleeplessly, methodically — so you could focus on the decisions that only you can make?
Someone who would keep the investigation going over the weekend, in your place...

There is no such person.
Except for your **AI friends**.


## About FORENSIA

![](https://gist.githubusercontent.com/sumeshi/c2f430d352ae763273faadf9616a29e5/raw/afaaa9317a3170fc22b17e0a833632b0018acd12/forensia.svg)

Out of this background, I built [FORENSIA](https://github.com/sumeshi/forensia).

It ingests artifacts already extracted from the machines under investigation, then automatically generates and verifies investigative hypotheses using a local LLM, continuously updating a report as it goes.

![](https://github.com/user-attachments/assets/36225144-70ea-4ecd-96f9-b9f84ce9e30d)

Fig. A dashboard for monitoring investigation progress.

![](https://github.com/user-attachments/assets/c6d61b5e-16e2-4e73-82e2-bde4b29fac99)

Fig. The investigation report.

This article explains the design philosophy behind this software.


## Our Battlefield

AI has advanced at a staggering pace in recent years.
Both Anthropic and OpenAI are producing models good enough to be called **frontier AI**.

And yet, there are plenty of cases where you simply cannot bring these models into a security incident investigation as-is. Why?

Attackers target systems worth compromising — systems that handle important, sensitive information.
When the victim is already on edge after a breach, there is no way you can bring yourself to ask, "Mind if we send data from your compromised system to an external service?"

Even setting that aside, sectors under legal and regulatory control — defense, energy, finance — demand the utmost care in how information is handled, depending on what they hold.

In these cases, using a managed generative AI platform`*1` is out of the question, and even uploading data to cloud infrastructure`*2` is difficult. There are cases where you must physically destroy the machines used in the investigation and submit a certificate of destruction, after all.

More fundamentally, processing data on a network-connected machine creates new risk in itself. Investigating a compromised machine means handling malware and attack code whose true nature you do not yet understand. Run things carelessly and leave them unattended, and the next time you look at the screen, you might be greeted by a ransomware note.

**When you gaze into an incident, the incident gazes also into you.**

> *1 You are free to believe that an enterprise contract with a training opt-out makes this fine. But contracts are not always honored — they get broken whenever the benefit of breaking them outweighs the cost. The point is not whether it actually happens, but that you must keep in mind that it can.

> *2 Same as above. And in the first place, how does a user guarantee that data they no longer hold is being handled correctly? If you can explain that convincingly to the affected party, fine. Personally, I consider placing sensitive data somewhere outside your full control to be no different from a secondary incident.


## Design Principles

For the reasons above, I defined the requirements this project must satisfy.

### 1. Fight Alone

The software must keep working completely offline. Leave a network-connected system unattended over the weekend, and what you find on Monday morning might be a ransomware extortion screen.


### 2. Don't Over-Trust the LLM

I'd love to use a huge model if I could. But the hardware to run one is expensive. A specialized firm that handles forensic investigations year-round might justify it, but it is hard to imagine a corporate CSIRT that only occasionally runs investigations getting hundreds of thousands — or millions — of dollars in capital expenditure approved.

Realistically, we're talking about investigation gear that fits within a consumables budget or a small equipment budget: a GPU costing a few hundred dollars new, or $150–200 used if you're an individual. What runs on that is, at best, a small model in the 4B–8B range.

Toss them a log with a casual "investigate this, would you?" and without rigorous prompting they will twist your instructions. Feed them a long document in one go and they will drop important details. On top of that they are terribly amnesiac, unable to remember an exchange from just moments ago.

So the LLM is only entrusted with tasks that are hard to build statically: generating hypotheses, proposing verification strategies, writing the report. Everything else — searching and selecting relevant data, and so on — should be handled mechanically by the harness architecture.

These days humanity seems to be at the beck and call of generative AI, but compared to a few-billion-parameter model, we humans are still somewhat slower and somewhat smarter. **Let us reclaim our pride as the lords of creation and work these models like cogs in a machine.**


### 3. Spend Time Like Water

There is no need to reach a perfect conclusion in a single run.

Human-driven digital forensics doesn't work that way either, so neither should AI-driven forensics. We run inference over the same hypothesis again and again, recording the process and the memory with full observability. This lets a human — or an LLM — later understand why a given piece of evidence led to a given conclusion.

The report, too, is not written in one batch at the end; it is continuously updated as the investigation progresses.

That said, spending time freely and never finishing are entirely different things. Even human-driven forensics has to wrap up somewhere — budgets and deadlines exist.

The ideal is to hold multiple hypotheses, keep updating their priorities, and iteratively refine the findings.


## Architecture

At the heart of FORENSIA is the same loop that matters in human-driven forensics: repeated hypothesis and verification.
Simplified, the processing flow looks like this:

1. Ingest evidence and normalize it into an investigable format
2. Generate hypotheses, starting from detections by predefined rules
3. Select the next hypothesis to verify
4. Search the actual evidence and verify the hypothesis
5. Judge and record the result, and reflect it in the report
6. Feed unresolved questions back into the next investigation cycle

In this loop, only the work where a language model genuinely helps — generation and interpretation — goes to the LLM. Evidence search, priority computation, state transitions, persisting verification results: everything that can be handled deterministically runs in code.

But just spinning this loop is not enough — a weak LLM will quickly get lost.
It will treat baseless hypotheses drawn from its own meager internal knowledge as fact, or forget the context and repeat the same reasoning over and over.

So I built the loop on three pillars that make it viable:

1. **Evidence-bound hypothesis verification**
2. **Structured memory**
3. **Auditability of the investigation process**


### 1. Evidence-Bound Hypothesis Verification

A hypothesis or judgment generated by an LLM is not, by itself, an investigative finding.

"This logon might be lateral movement." "This process may have been executed by the attacker." However plausible such statements sound, they are nothing more than hypotheses to be checked.
So the LLM's output is never stored directly as fact; the actual evidence is searched, and only after cross-checking against those results is the state updated.

Hypothesis judgments are managed as five states: `confirmed`, `refuted`, `inconclusive`, `untestable`, and `newlead` (a new lead was discovered). But an LLM answering `confirmed` does not settle anything on its own. The code side also checks the evidence that was referenced, the independence of the pieces of evidence from each other, any contradictions, and whether the activity was even observable in the first place.

#### Evidence of Absence

Even if an evidence search returns zero rows, that alone does not mean "the activity never happened."
The LLM's generated search query may have been wrong; the evidence may have existed but been deleted or tampered with by the attacker; or the machine under investigation may simply never have been configured to record that evidence in the first place.

Only after accounting for multiple factors — was that evidence actually collected, does it fall within the searched time range, and so on — do we consider the possibility of refutation.

#### Declaring Investigative Intent in Detection Rules

Hypothesis generation is not left to the LLM's free imagination.

Rules declare not just detection conditions, but also — in natural language — what was detected, why it matters, and what should be investigated next as a consequence. If you only hand the LLM the fact that a condition matched, it has no idea why that match was selected.

Since there is no need for real-time detection here, we can afford to embed this kind of information.
The downside is that the rule author's bias creeps in — but that is still somewhat better than nonsensical hallucinations rooted in model knowledge. Background knowledge and additional perspectives not covered by the rules are supplemented from the Knowledge store described later.


```yaml
id: windows-security-4624-rdp-logon
query: SELECT ... FROM evtx_events WHERE event_id = 4624 AND logon_type = '10'
finding:
  title: RDP logon for {target_user}
hypotheses:
- id: lateral_movement_via_rdp
  description: RDP logon from {src_ip} to {computer} may indicate lateral movement
  confirm_when: { co_observed_event_ids: [4624], same_host: true, within_minutes: 5 }
  refute_when: { zero_rows: true }
  follow_up_questions:
  - Was this RDP session followed by process execution (4688) on {computer}?
correlate_with:
- { event_ids: [4625, 4768, 4776], rationale: preceding failures or kerberos/NTLM auth }
```


### 2. Structured Memory

What matters most in an AI harness is maintaining context. FORENSIA manages memory in three layers: Case State, Trace State, and Working State.

#### Case State

The state representing **what exists in the current case**. Long-term memory.

As the single source of truth, it stores the ingested evidence, plus findings from detection rules, generated hypotheses, the mappings between hypotheses and evidence, tasks, and report progress.

#### Trace State

The state representing **why we arrived at the current state**. Trace memory — a concept humans don't really have.
It stores the inputs and outputs of each step, decisions, state transitions, and the score breakdown behind choosing the next investigation target.

Thanks to this, the reasoning process behind any conclusion can be reviewed afterwards.

#### Working State

The state for **passing the LLM only the context needed for the current task**. Working memory.
Something in between long-term memory and the short-term memory that evaporates with each LLM session.

From Case State and Trace State, it extracts confirmed facts, key events, related hypotheses, and unresolved questions, and condenses them into a size the LLM can handle. Rather than passing all evidence and all past processing every time, it reconstructs the context with only the information needed for the current role and target of investigation.

Unconfirmed information is quarantined per hypothesis, never allowed to bleed into other hypotheses.
And note: this is a projection for maintaining the LLM's context, never an authoritative copy of the evidence.

#### Knowledge

Separate from the three State layers above, this is external memory used RAG-style, managed in the [OKF format](https://cloud.google.com/blog/products/data-analytics/how-the-open-knowledge-format-can-improve-data-sharing).
Natural language is more human-readable and easier to update, and it's easy to have an LLM classify and reorganize collected documents and web pages.

Of course, for publicly known information there is no problem at all with using managed AI services — so information that can be managed separately should be gathered and cultivated aggressively, then copied in when needed.

```yaml
---
type: knowledge
title: Short title
description: One-line summary. Used for index display and relevance scoring.
tags: [windows, eventlog, logon]
timestamp: 2026-07-13
---
# Body (Markdown)

## Section heading

Sections are selected and injected individually.
```

Which documents get included in the context is decided by narrowing candidates with tags, then scoring relevance from title, description, and body. The same input always selects the same documents, so we can explain why the LLM reasoned with a given document.

Injected knowledge is tagged as such, with an explicit authority boundary in the prompt: it is reference material, not instructions.


### 3. Auditability of the Investigation Process

Merely persisting states and history does not make the findings auditable. The relationships between conclusions, interpretations, observations, and evidence must be explicit, and it must be possible to trace from the report all the way back to the original evidence.

We record which evidence was examined, which hypotheses were verified, why each judgment was reached, and what ultimately went into the report. What matters is not that the AI believes it is right, but that **a human can verify its correctness afterwards**.

#### Tracing from Conclusion Back to Evidence

Information handled during an investigation is managed separately by role:

- **Evidence**: evidence normalized from EVTX, MFT, and the like
- **Finding**: an event observed in the Evidence
- **Hypothesis**: an interpretation yet to be verified
- **Claim**: an assertion presented to the reader in the report
- **Gap**: missing evidence or information needed for a judgment

Mix these together and you get unverified hypotheses remembered as facts, or a single log entry over-generalized into the report.

#### The Report Is Part of the Investigation State

The report is not a document to be batch-generated by the LLM at the end of the investigation.

Parts that can be expressed in fixed formats — tables, listings — are generated in code; the LLM writes only the parts requiring explanation or interpretation, paragraph by paragraph. The evidence referenced by each section, and the key Claims within the text, are recorded as well.

As the investigation progresses and new facts are confirmed, the relevant sections are updated. Parts that cannot yet be explained are not padded out; they are returned to the next investigation cycle as Gaps.

#### Never Delete Evidence — Record Its Treatment as State

Data that should be excluded from analysis is (naturally) managed separately, without touching the originals.

For example, when excluding an anomalous timestamp from the analysis window, the value itself is kept, along with the recorded reason for its exclusion. If you pretend it never existed, you lose the trail when other evidence surfaces later — so instead we record: "this was excluded from analysis and display, at this time, for this reason."


## Design Choices

### Naming the Software

Forensics + AI = FORENSIA. Good enough — that's roughly how carefully I named it.

Also, the games I made as a student had names like `amnesia` and `insomnia` — Latin-derived "suffixes of state" — so I vaguely felt it had a fittingly forensic-junkie ring to it.


### How It Differs from Existing Tools

There are several attempts to automate forensics with LLMs; perhaps the best known is [Mecha Hayabusa](https://github.com/Yamato-Security/mecha-hayabusa). Most of these use AI to operate existing analysis tools and speed up investigations — which is a slightly different concept from this project.

What FORENSIA attempts is to keep an investigation going: forming the next question from processing results, selecting the evidence it needs, and carrying unresolved points into the next cycle. In essence, it is a PoC that decomposes the human investigative procedure into fine-grained steps and implements them — an idea given form, exploring what such an architecture should look like.

Also, real forensics does not always leave evidence in convenient, log-like forms. It is by nature the work of combining a mess of evidence — possibly incomplete, possibly tampered with — and iterating hypothesis and verification. The eventual goal is to handle that.


### Why Not Reuse Existing Detection Rules

[Sigma](https://github.com/sigmahq/sigma) is probably the richest rule base out there, but as described above, FORENSIA's rules are not mere detection rules — they embed investigative intent and hypothesis hints.

Being able to reuse existing assets would surely be better, but like the Knowledge store, I see this as something to be grown through use.

### Why No Internal Tool Calling

I went back and forth on this quite a bit, from the standpoint of loose coupling, but decided against it because it could narrow the choice of backend models.

At the moment, gemma-4 is remarkably good for running on a cheap PC, yet whether its Tool Calling is stable enough to rely on is highly questionable.
And having to revisit the logic every time someone says "a better model came out, we're switching" or "our hardware runs Kimi-K3 just fine" felt like it would defeat the purpose.


## Future Directions

FORENSIA is still in the early stages of development.

The current goal is not a high-precision investigation tool that can immediately replace real-world work. First, I am testing how architecture can compensate — using a weak local LLM — for the parts that static rules and hard-coded processing struggle to handle.

### Broadening the Evidence Types

The main artifacts currently supported are [EVTX](https://github.com/sumeshi/evtx2es), [MFT](https://github.com/sumeshi/mft2es), and [Prefetch](https://github.com/sumeshi/prefetch2es). All three use parsers I wrote myself and can be ingested in a relatively uniform format, which is why I chose them as the first targets.

Going forward, I would like to incorporate messier artifacts such as browser records and the registry. But complete coverage is not the goal. The registry in particular has different structure and semantics for every key — frankly, there's no way I'm doing all of them. I plan to add artifacts in order of effort-to-impact: those that yield the most value with the least labor, without requiring endless rounds of inference.

Multi-host support also matters. In real incident investigations, cases involving just a single machine are not the norm.

This is not simply a matter of repeating single-host processing N times: each host must be treated as an independent subject while relationships — accounts, IP addresses, processes, timelines — are examined across them. That looks like a major design change. Some way off, I suspect.


### How to Measure Investigation Quality

In my own trials using [CFReDS](https://cfreds-archive.nist.gov/data_leakage_case/data-leakage-case.html) data, it got close to the answer on about 8–9 of 12 questions. But this is an observation from a handful of runs with one specific model ([google/gemma-4-e2b](https://huggingface.co/google/gemma-4-E2B)) and configuration, not a fixed official result.

Moreover, a rising answer count does not by itself mean the overall quality of investigation has improved.
If, in overfitting to benchmark answers, hypotheses skew toward particular artifacts, untraceable prose increases, or unconfirmed speculation leaks into memory — that is not improvement.

So going forward, alongside answer accuracy, I want to track hypothesis diversity, memory duplication, LLM call patterns, and more.


## Conclusion

Through developing FORENSIA, I feel I've come to see many of the challenges peculiar to weak LLMs.
In the end, you need a capable commander (in this tool, the harness's mechanical approach plays that role), and the choice comes down to two options: implement that role doggedly by brute force, or let a strong model absorb the problem for you.

**Honestly, local LLMs are rough!!!!**

The end.
