# Building a Production AI Investigation Agent: What We Learned About Trustworthy Root-Cause Analysis

*An engineering case study of an LLM-assisted operational investigation workflow, retrospective evaluation, and gradual production rollout.*

> **About this writeup:** This is a high-level account of my contributions to a collaborative production project, not an open-source implementation. It excludes proprietary code, prompts, customer information, internal endpoints, and sensitive operational details. Any public sharing remains subject to my employer's confidentiality requirements.

## The problem: investigations require more than a plausible answer

Support engineers often investigate issues by moving between technical documentation, past support tickets, and operational data. Finding relevant evidence is time-consuming, and similar symptoms do not always have the same root cause.

We built an AI-assisted investigation workflow to help engineers assemble context, explore potential root causes, and understand whether an issue might affect more than one device or case. The goal was to **accelerate investigation while keeping engineers responsible for the final judgment**, not to automate incident decisions without oversight.

## What we built and what I owned

The team developed a production LLM-based investigation pipeline and introduced it gradually to support engineers. I served as the **primary architect of the overall pipeline** and was one of the main engineers driving its development and rollout. Implementation was collaborative, with different engineers responsible for different components.

The workflow had several stages:

1. **Technical documentation retrieval.** Semantic search surfaced documentation relevant to the reported symptoms.
2. **Historical ticket retrieval.** A retrieval component searched prior support cases and ranked potentially relevant tickets. I co-developed this component.
3. **Operational evidence gathering.** The workflow used available data schemas to retrieve operational records relevant to the investigation.
4. **Root-cause analysis.** An LLM-based component combined the current issue, retrieved evidence, and relevant historical context to suggest possible explanations. I contributed to this component.
5. **Impact-scope analysis.** Another component helped assess whether a suspected issue was isolated or potentially broader. This component was owned by another engineer.
6. **Evaluation, sanity checks, and investigation tracking.** I independently owned these parts of the project, which helped us review output quality and iterate on the workflow.

The implementation was primarily Python-based, with Apache Airflow used for workflow orchestration.

## The reliability problem: convincing reasoning can still be wrong

The failure mode I worried about most was an agent producing a **credible-sounding root-cause explanation that the evidence did not support**. Operational data can be incomplete, symptoms can overlap, and a model can mistake correlation for causation. A fluent explanation is not proof.

I encountered a concrete version of this problem while building our evaluation framework. Our initial LLM-as-a-judge approach scored investigation reports soon after generation. During case reviews, I found examples where the judge rated a report favorably even though its root-cause conclusion was wrong or rested on unsupported assumptions.

This created a second-order reliability problem: **the tool we were using to measure agent quality could itself reward the wrong behavior**. If we treated that score as ground truth, we risked becoming more confident in an unreliable signal.

## My key change: evaluate against later investigation evidence

I drove a redesign from immediate report scoring toward **retrospective evaluation after the support ticket was closed**.

By that point, subsequent engineer investigation notes could provide additional evidence about the actual issue. We used that later evidence to assess the quality of the original AI-generated report, including whether its conclusions were supported and whether it missed important information.

One boundary mattered: **ticket-closure evidence was available to the evaluator, not retroactively supplied to the original report generator**. Otherwise, we would have been measuring a system with information it could not have known at the time.

We reviewed a range of closed cases and compared model-based assessments with human judgments. Already-closed historical cases also gave us a practical way to compare candidate versions offline without waiting for every new live investigation to close.

The tradeoff was slower feedback for live cases, but a more evidence-grounded signal for quality monitoring and engineering decisions. I would not claim this made the judge infallible; it made the evaluation better aligned with the question we actually cared about: **was the investigation correct and useful, not merely well written?**

## Two engineering decisions shaped by evidence

**Use historical cases as reasoning context.** Initially, related support tickets were available mainly as references. I proposed incorporating relevant historical cases into the root-cause-analysis context so the agent could consider recurring patterns alongside current operational evidence. Manual reviews showed promising examples, but we did not establish a controlled, quantified improvement at that stage.

**Do not assume a larger model fixes the bottleneck.** We evaluated a proposed upgrade to the model used in retrieval. The observed improvement in end-to-end investigation quality was limited, so the team chose not to make that upgrade. We considered retrieval relevance itself as a possible area to improve, rather than assuming model capability was the limiting factor.

## Rolling out a system engineers could question

The pipeline reached production through a gradual rollout to support engineers. Some engineers were understandably cautious about the accuracy of AI-generated investigations, so we positioned the system as an **optional investigation accelerator**, not an autonomous authority.

We kept human review central, listened to feedback from real cases, and iterated on reliability. Production deployment was a milestone, not proof that every generated conclusion could be trusted.

## Takeaways

Building this system changed how I think about production agents:

- **An explanation must be supported, not just plausible.** This matters especially when the input is incomplete operational evidence.
- **The evaluator is part of the reliability problem.** A judge that rewards persuasive but incorrect conclusions can hide the very failures it is meant to detect.
- **Evaluate against the real user decision.** Retrospective evidence and human assessments were more useful than a convenient score detached from factual outcomes.
- **Deployment and trust are different milestones.** Gradual rollout and human oversight were important because support engineers, not the model, remained accountable for incident decisions.

The biggest lesson was that a useful investigation agent needs more than a strong language model. It needs relevant evidence, an evaluation process that can detect convincing mistakes, and a workflow that helps humans make better decisions.

---

**My role:** Primary architect of the overall pipeline; one of the main engineers driving development and gradual production rollout; co-developer of historical-ticket retrieval; contributor to root-cause analysis; independent owner of evaluation, sanity checks, and investigation tracking.

**Disclosure:** This is a technical case study of a collaborative proprietary project. No company source code or internal operational data is included.
