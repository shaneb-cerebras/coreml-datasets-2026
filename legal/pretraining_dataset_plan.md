# Pretraining dataset plan

Proposed data strategy for legally conservative LLM pretraining.

This is an engineering analysis intended to identify a small set of datasets for legal review, not legal advice.

## Summary

| Dataset | Type | Approx. scale | Licensing / rights basis | Proposed action |
|---|---|---:|---|---|
| Common Pile v0.1 | Human | ~464B filtered unique tokens; ~400B in our current version | Public domain + licenses meeting Open Definition 2.1 | Already approved |
| Common Corpus | Human | 2.27T tokens current release; substantial non-English content | Public domain + free/open licenses allowing use for any purpose | Ask legal to approve |
| K2 Horizon synthetic releases | Synthetic | ~10T tokens estimated; exact total not published | Apache 2.0 at dataset level | Resolve provenance, then ask legal |
| Our own reasoning traces, e.g. Kimi K2.6 | Synthetic | TBD | Depends on model/output terms and prompt provenance | Investigate |
| Stack v3 | Code | Very large upstream corpus | Mixed, including unlicensed code | Do not curate ourselves; wait for a rights-filtered derivative |

## Proposed strategy

I have looked at public/open training recipes such as Marin and IFM/K2 to get an idea of which sources are likely to be effective.

I think the pretraining dataset strategy that makes the most sense is:

- **Human-generated:** Common Pile + Common Corpus
- **Synthetic:** K2 Horizon synthetic data, potentially supplemented by reasoning traces we generate ourselves

The objective is not to reproduce the licensing/provenance work already done by projects such as Common Pile and Common Corpus. Instead, we should rely on a small number of well-documented curated datasets and keep the legal-review surface as small as possible.

## Common Pile v0.1

Common Pile deliberately restricts itself to public-domain material and works under licenses satisfying the Open Knowledge Foundation's Open Definition 2.1, including licenses such as CC0, CC-BY, CC-BY-SA, MIT, BSD and Apache 2.0.

The approximate progression is:

> 8 TB raw → 1.84 TB after Common Pile filtering/deduplication → ~464B unique tokens → ~400B tokens in our version after removing some subsets such as YouTube and reserving validation data

The ~464B figure is the amount of unique filtered data. The published Comma training mixtures reach 1T or 2T training tokens by reweighting/repeating subsets rather than by providing 1T or 2T unique tokens.

**Status:** Legal has already approved our use of Common Pile.

## Common Corpus

Common Corpus has essentially the same motivation as Common Pile: aggregate large quantities of text for which there is an explicit open-use basis rather than relying on arbitrary copyrighted web crawling.

The current Common Corpus release contains **2.27T tokens** and describes every document as either uncopyrighted or freely licensed. It includes:

- ~967B OpenCulture
- ~579B OpenGovernment
- ~283B OpenSource/code
- ~281B OpenScience
- ~89B OpenWeb
- ~68B OpenSemantic

It provides document-level license/provenance metadata and states that its data can be filtered by licensing class, including public-domain-only or selected free-license subsets.

Common Corpus is substantially multilingual. In the earlier ~2T snapshot, only about half of the natural-language tokens were English. Therefore, the full 2.27T headline substantially overstates the amount we would use for an English-centric model.

An earlier Common Corpus training effort used roughly **1.1T tokens in its more aggressively filtered training corpus**. The corpus has expanded since then, so this should not be interpreted as 2.27T currently filtering directly to 1.1T.

### Expected incremental value

Based on the published source composition, my rough estimate is that Common Corpus may provide **on the order of 1T English/code tokens not already present in Common Pile**.

This is only an estimate. Before training we would need to:

1. restrict the corpus to languages/data classes we want;
2. apply our own quality selection and sampling weights; and
3. deduplicate against Common Pile.

The incremental data is not uniformly valuable. There is a large amount of historical OCR text, government/legal material and other specialized data that we may want to downweight, while the additional permissively licensed code and scientific text are particularly attractive.

### Licensing

Common Corpus includes public-domain material plus "free/open" licenses that permit use for any purpose, including commercial use. In practice this includes broadly the same classes of licenses accepted by Common Pile.

**Proposed legal ask:** approve Common Corpus.

## Code and Stack v3

Common Pile and Common Corpus have already done the work of extracting permissively licensed code from earlier versions of The Stack.

The Stack v3 is now dramatically larger than Stack v1/v2, but the upstream release contains both permissively licensed code and code for which no license was detected.

I do **not** think we should build our own legal/license-filtering pipeline for Stack v3. That would make us responsible for deciding how reliable license detection is, handling mixed-license repositories, deciding which licenses to allow, and dealing with other provenance issues.

**Recommendation:** wait for Common Pile, Common Corpus, or another credible rights-focused project to release a permissively licensed Stack v3 derivative.

## K2 Horizon synthetic data

IFM reports that each K2 Horizon model uses approximately **20T pretraining tokens**, of which approximately **10T are synthetic**.

Nearly 17% of K2 pretraining is explicit problem-solving trajectories. IFM also generates other synthetic forms including mathematical dialogues, study guides, rewrites, planning data and code reasoning.

IFM has now released three particularly large datasets:

| Dataset | Content | Records | Compressed size | License |
|---|---|---:|---:|---|
| IFM/Pretrain-Behaviors | reasoning, general text, planning, data science, games, format rewrites | 1.64B | 8.37 TB | Apache 2.0 |
| IFM/Math-Reasoning | mathematical reasoning, rewriting, dialogue | 2.02B | 4.46 TB | Apache 2.0 |
| IFM/Code-Reasoning | code reasoning and task synthesis | ~452M | 3.28 TB | Apache 2.0 |

The dataset cards do not give total token counts. Based on compressed size and comparison with other tokenized text corpora, I estimate that the three datasets collectively contain **on the order of ~10T tokens**, but this should be treated as an estimate rather than a published number.

### Provenance issue

Each release is divided into identifiable subsets. For example, Math-Reasoning contains:

- `math-thinking-qwen`
- `math-thinking-oss`
- `math-rewrite`
- `math-dialogue`
- `socratic-math-dialogue`

IFM states:

> Individual subsets may have undergone source-specific filtering, cleaning, deduplication, quality scoring, or synthetic-data generation. Users should evaluate each subset for their target use case and inspect the available provenance metadata.

However, I have not found sufficiently detailed public provenance describing, for each subset:

1. the source/seed prompts or documents;
2. the licenses/rights associated with those seeds; and
3. the exact generator model(s).

This matters because a synthetic dataset is not automatically free of upstream licensing issues. If a generated reasoning trace starts with a copyrighted or restricted problem/prompt, generation of a new response does not necessarily eliminate the issue associated with the prompt.

Similarly, generator-model terms may impose conditions on use of model outputs.

### Proposed next step

Before involving legal, ask IFM whether they can provide a mapping from each released subset to:

- source/seed datasets and their licenses;
- generator model(s); and
- any relevant restrictions on commercial model training using the released data.

If IFM can provide a clean provenance statement, these datasets become an attractive legal-review target because IFM has explicitly released them under Apache 2.0 and the potential data volume is enormous.

**Proposed legal ask:** only after resolving the provenance question, ask legal to approve the relevant K2 Horizon subsets.

## Internally generated synthetic data

We could also generate our own reasoning/synthetic data, for example using Kimi K2.6.

For this case we should establish:

1. whether the model/provider terms permit using outputs to train our models;
2. where our prompts/seeds originate;
3. whether any source material embedded in those prompts carries restrictions; and
4. whether we retain enough provenance to explain the construction of the resulting corpus.

If the prompts are Cerebras-created or come from already approved datasets and the generator's terms permit model training, this may provide a particularly clean way of expanding the corpus.

## General provenance principle

A permissive top-level dataset license is useful but is not by itself sufficient evidence that every input to the dataset was permissively usable.

For human data, the relevant issue is primarily the provenance/license of the underlying documents.

For synthetic data, there are two additional questions:

- **Prompt/seed provenance:** where did the source problem, prompt or context come from?
- **Generator terms:** what model generated the response and what restrictions apply to its outputs?

This is why Common Pile/Common Corpus are attractive: provenance and rights filtering are central to their construction. For K2 synthetic data, the top-level Apache 2.0 release is attractive, but we should first resolve the less-documented upstream provenance.

## Requested legal approval

There are four useful levels of approval:

1. **Internal research:** use the dataset for Cerebras research experiments with no public disclosure of the research.
2. **Published research without dataset disclosure:** publish research results without identifying use of the dataset.
3. **Published research with dataset disclosure:** publish research results and explicitly disclose that the dataset was used.
4. **Model release:** publicly release models/checkpoints trained on the dataset.

For our immediate research needs, I suggest asking for **level 3** approval.

This lets us conduct normal publishable research and accurately disclose our training data, while avoiding the broader question of public redistribution of trained weights.

If legal does not approve level 3 for a particular dataset, we can separately ask whether level 1 or level 2 use is acceptable.

## Recommended legal-review sequence

1. **Common Pile:** already approved.
2. **Common Corpus:** request level 3 approval, ideally under the same license allowlist already accepted for Common Pile.
3. **K2 Horizon synthetic data:** first resolve subset-level provenance with IFM; if satisfactory, request level 3 approval.
4. **Internally generated synthetic data:** establish generator terms and prompt provenance, then determine whether separate legal approval is needed.
5. **Do not currently ask legal to review raw Stack v3 or broad web-crawl datasets.**

The goal is to give legal a small number of well-scoped decisions rather than asking them to approve individual upstream datasets or make broad judgments about arbitrary copyrighted web data.
