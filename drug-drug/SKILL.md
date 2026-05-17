---
name: drug-drug
description: >
  Evidence-based Drug-Drug Interaction (DDI) assessment skill modeled after the Micromedex Drug-Reax
  methodology. Trigger this skill whenever the user types /drug-drug, mentions "drug interaction",
  "DDI", "drug-drug", "can I take X with Y", "interaction between", "交互作用", "併用",
  polypharmacy, or asks whether two or more medications can be used together. Supports 2–5 drugs:
  automatically enumerates all pairwise combinations and produces individual DDI reports plus a
  summary matrix. This skill performs systematic literature retrieval via PubMed, CrossRef,
  OpenEvidence, and WebSearch, then produces structured assessment reports with Severity,
  Documentation, Onset, Mechanism, Clinical Effects, and Management —
  mirroring the Micromedex Drug-Reax classification framework.
---

# Drug-Drug Interaction (DDI) Evidence-Based Assessment

## Overview

This skill implements a Retrieval-Augmented Generation (RAG) approach to drug-drug interaction
assessment, modeled after the Micromedex Drug-Reax System methodology. Rather than relying on
LLM training data alone -- which may be outdated, incomplete, or hallucinated -- every claim in
the output is grounded in real-time literature retrieval from peer-reviewed sources with full
citation traceability. The structured report mirrors Micromedex Drug-Reax classification grades
and is designed to complement (not replace) certified drug information systems and clinical
pharmacist expertise.

## Trigger

- `/drug-drug DrugA DrugB [DrugC] [DrugD] [DrugE]` — accepts 2 to 5 drugs
- Any query mentioning two or more drug names and asking about interaction, safety, or concomitant use
- Keywords: DDI, interaction, contraindication, concomitant, polypharmacy, 交互作用, 併用, 多重用藥

---

## Workflow (execute strictly in order)

### Step 0: Drug Name Resolution

Extract all drug names from user input (minimum 2, maximum 5).

- If the user provides brand names, resolve to INN/generic names.
- If fewer than 2 drugs are provided or names are ambiguous, ask the user to clarify.
- **If more than 5 drugs are provided, stop and respond**: "This skill supports a maximum of 5 drugs at once (up to 10 pairs). Please reduce the number of drugs to 5 or fewer."

### Step 0.5: Polypharmacy Pair Enumeration

If 3 or more drugs are provided, enumerate **all unique pairwise combinations** and display them to the user before proceeding. Use this formula: N drugs → N×(N−1)/2 pairs.

**Example for 4 drugs (A, B, C, D) → 6 pairs:**

| # | Pair |
|---|------|
| 1 | A – B |
| 2 | A – C |
| 3 | A – D |
| 4 | B – C |
| 5 | B – D |
| 6 | C – D |

Inform the user: "I will now assess [N] pairs. This may take a moment."

Then **execute Steps 1–3 for each pair sequentially** before generating the final report in Step 4.

### Step 1: Literature Search

Execute **at least 5 distinct searches** to ensure adequate coverage:

1. **PubMed Search** (via web_search)
   - `site:pubmed.ncbi.nlm.nih.gov "DrugA" "DrugB" interaction`
   - `site:pubmed.ncbi.nlm.nih.gov "DrugA" "DrugB" pharmacokinetic`
   - `site:pubmed.ncbi.nlm.nih.gov "DrugA" "DrugB" CYP450`

2. **CrossRef Search** (via web_fetch)
   - `https://api.crossref.org/works?query=DrugA+DrugB+drug+interaction&rows=10&sort=relevance`

3. **General WebSearch**
   - `DrugA DrugB drug interaction clinical significance`
   - `DrugA DrugB interaction mechanism CYP enzyme`
   - `DrugA DrugB interaction case report adverse event`

4. **FDA Label / DailyMed**
   - `site:dailymed.nlm.nih.gov DrugA interaction`
   - `DrugA DrugB FDA drug interaction warning`

5. **PubMed MCP Tool** (if PubMed MCP server is connected)
   - Use `PubMed:search_articles` with `"DrugA" AND "DrugB" AND "drug interaction"`
   - Use `PubMed:get_article_metadata` for key articles

6. **OpenEvidence Skill Search**
   - Query the OpenEvidence skill with a direct clinical question: "What is the drug interaction between DrugA and DrugB? What are the clinical effects, mechanism, and management?"
   - Extract the synthesized clinical evidence and management recommendations from the OpenEvidence response.

### Step 2: Evidence Appraisal

Read `references/evidence-grading.md` for complete grading criteria.

For each retrieved article, extract and document:
- Study design (RCT / cohort / case-control / case report / in vitro / review)
- Sample size
- Key findings (AUC fold-change, Cmax change, clinical events)
- Interaction mechanism described
- Credibility assessment

**Traceability requirement**: Every factual claim in the final report must be attributable to a
specific source with PMID or DOI. If a claim is inferred from pharmacologic reasoning rather
than direct evidence, it must be explicitly labeled as such (e.g., "based on known CYP3A4
inhibition profile" rather than stated as established fact).

**Uncertainty signaling**: When evidence is limited or conflicting, the report must explicitly
state the level of uncertainty. Use phrases such as "limited evidence suggests," "based on
case reports only," or "no direct clinical studies available; assessment based on pharmacologic
reasoning." Never present uncertain inferences with the same confidence as RCT-level evidence.

### Step 3: Structured Classification

Classify using the Micromedex Drug-Reax framework. See `references/evidence-grading.md` for
detailed decision trees.

#### 3a. Severity

| Grade | Definition |
|-------|-----------|
| **Contraindicated** | Drugs are contraindicated for concurrent use |
| **Major** | Interaction may be life-threatening and/or require medical intervention to minimize or prevent serious adverse effects |
| **Moderate** | Interaction may result in exacerbation of the patient's condition and/or require a change in therapy |
| **Minor** | Interaction would have limited clinical effects; may augment side effects but generally does not require a change in therapy |

#### 3b. Documentation

| Grade | Definition |
|-------|-----------|
| **Excellent** | Controlled studies have clearly established the existence of the interaction |
| **Good** | Documentation strongly suggests the interaction exists, but well-controlled studies are lacking |
| **Fair** | Available documentation is poor, but pharmacologic considerations lead clinicians to suspect the interaction exists; or documentation is good for a pharmacologically similar drug |
| **Poor** | Documentation is very limited, e.g., only isolated case reports or theoretical rationale |
| **Unlikely** | No reasonable pharmacologic basis for the interaction |

#### 3c. Onset

| Grade | Definition |
|-------|-----------|
| **Rapid** | Clinical effects of the interaction occur within 24 hours |
| **Delayed** | Clinical effects of the interaction occur after 24 hours |
| **Not specified** | Onset not clearly documented in the literature |

#### 3d. Mechanism

Classify the interaction mechanism into:

**Pharmacokinetic (PK):**
- CYP450 enzyme inhibition (specify isoform: CYP3A4, CYP2D6, CYP2C19, CYP2C9, CYP1A2, etc.)
- CYP450 enzyme induction
- P-glycoprotein (P-gp) / transporter-related
- Protein binding displacement
- Renal tubular secretion competition
- Absorption-level (pH alteration, chelation, etc.)

**Pharmacodynamic (PD):**
- Additive effect (e.g., QTc prolongation, bleeding risk, CNS depression)
- Synergistic effect
- Antagonistic effect

### Step 4: Report Generation

#### 4a. Mode Selection

- **2-drug mode**: Produce a single detailed DDI report (format below).
- **3–5 drug mode (Polypharmacy)**: First produce a **DDI Summary Matrix**, then produce individual detailed reports for each pair — prioritizing pairs with Contraindicated or Major severity.

#### 4b. DDI Summary Matrix (Polypharmacy Mode Only)

```markdown
# Polypharmacy DDI Summary Matrix

## Drug List

| ID | Generic Name | Brand Name(s) |
|----|-------------|---------------|
| A  | [name]      | [names]       |
| B  | [name]      | [names]       |
| C  | [name]      | [names]       |

## Interaction Matrix

| Pair  | Severity        | Documentation | Onset         | Key Clinical Concern         |
|-------|-----------------|---------------|---------------|------------------------------|
| A – B | [Grade]         | [Grade]       | [Grade]       | [one-line summary]           |
| A – C | [Grade]         | [Grade]       | [Grade]       | [one-line summary]           |
| B – C | [Grade]         | [Grade]       | [Grade]       | [one-line summary]           |

> Severity color guide: Contraindicated = critical | Major = high risk | Moderate = caution | Minor = low risk

## Highest-Risk Pairs Requiring Immediate Attention

[List any Contraindicated or Major pairs with a brief clinical note]
```

#### 4c. Individual Pair Report (for each pair)

```markdown
# DDI Report: [DrugA] + [DrugB]

## Drug Pair

- **Drug A:** [Generic Name] ([Brand Names])
- **Drug B:** [Generic Name] ([Brand Names])

## Structured Classification

| Parameter       | Grade          | Note              |
|-----------------|----------------|-------------------|
| Severity        | [Grade]        | [note]            |
| Onset           | [Grade]        | [note]            |
| Documentation   | [Grade]        | [note]            |

## Interaction Effect

[Description of the clinical effect of the interaction — what happens when these drugs are used together]

## Clinical Management

[Specific clinical management recommendations:]

- Whether to avoid concomitant use
- Dose adjustments required
- Monitoring parameters
- Alternative drug suggestions

## Probable Mechanism

[Pharmacological mechanism — PK and/or PD, including specific enzymes, transporters, or receptors involved. Include PK data (AUC/Cmax changes) if available from studies.]

## Evidence Sources

[Key references: Author, Journal, Year, PMID/DOI]
```

#### 4d. Disclaimer (append to all reports)

```markdown
## Disclaimer

> **This report is generated by AI using a retrieval-augmented generation (RAG) approach and
> is intended as a clinical decision support aid only.** It does not constitute medical advice
> and cannot replace certified drug information systems (Micromedex, Lexicomp, Clinical
> Pharmacology) or the judgment of qualified healthcare professionals. AI-generated content
> carries inherent limitations including potential for incomplete literature retrieval,
> misinterpretation of source data, and inability to account for individual patient factors.
> All clinical decisions must be made by licensed practitioners with access to complete
> patient information, institutional formulary policies, and current prescribing guidelines.
> When in doubt, consult a clinical pharmacist or drug information specialist.
```

---

## Key Principles

1. **Retrieval before generation**: Never rely on LLM training data alone. Always perform real-time literature search before generating any assessment. Training data may be outdated, incomplete, or reflect unverified sources (social media, blogs). Real-time retrieval from peer-reviewed databases ensures currency and reliability.
2. **Multi-source cross-validation**: Use at least PubMed + CrossRef + WebSearch + OpenEvidence (four sources). Cross-validate findings across sources to reduce the risk of hallucination or bias from any single retrieval.
3. **Full citation traceability**: Every factual claim must cite its source (PMID, DOI, or URL). This enables clinicians to fact-check AI-generated answers against the original sources -- a core requirement for explainable AI (XAI) in clinical settings.
4. **Conservative grading**: When evidence is insufficient, err on the side of higher severity and lower documentation grade (err on the side of caution). Patient safety takes precedence over precision.
5. **Mechanism is king**: Even without clinical studies, if the pharmacologic mechanism is clear (e.g., known CYP inhibitor + known CYP substrate), document it and assign at least Fair documentation.
6. **Transparency of uncertainty**: The report must clearly distinguish between evidence-supported facts, pharmacologic inferences, and areas of genuine uncertainty. Never present inferred information with the same confidence as controlled-study evidence.
7. **Context matters**: Note dose-dependent or population-specific differences when evidence supports them (e.g., high-dose vs. low-dose regimens may carry different risk profiles).
8. **Complement, not replace**: This tool supports clinical decision-making but does not substitute for professional judgment. The output is designed to be reviewed by a clinician, not acted upon autonomously.
9. **Language**: The report body defaults to English. If the user writes in another language, respond in that language but keep drug names, grade labels, and pharmacological terms in English.

## Reference Files

- `references/evidence-grading.md`: Complete evidence grading criteria, decision trees, CYP450 quick reference, and common PD interaction patterns
