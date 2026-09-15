# coreml-datasets-2026

For planning legal review of datasets, and perhaps working on datasets themselves.

Initial plan: [legal/pretraining_dataset_plan.md](https://github.com/shaneb-cerebras/coreml-datasets-2026/blob/main/legal/pretraining_dataset_plan.md)

Prior work:
- Joel collected some of our rollout prompts notes into this spreadsheet: [Pipelined_Jacobi_Demo_Dataset.xlsx](https://nnetx.sharepoint.com/:x:/s/engineering/IQBlhzx-R-DsSIFScuwEerjQAWRYE0yT7yYTH0idCdUVpi4?e=t8TJpM).
- Gavia made some notes that were useful when discussing with legal here: https://gist.github.com/gaviag-cerebras/ac9a53bd491162dfc0d2d8a9b5c22b29

## On Common Pile "approval"

Mostafa says: don't use those subsets of CommonPile
- news_filtered : we ran a script and found that includes materials from websites that are not copyright-free (e.g., nytimes.com)
- youtube_filtered: to be on the safe side, we are avoiding it because not all YouTube video owners give permission to use their subtitles as data
- stackexchange_filtered: also to be on the safe side we are avoiding it because there are news articles here and there about Stack Exchange  restricting the user of their data to train LLMs

Matt Hulse did approve the "Don't Drop Dropout" paper when we retrained with data coming from CommonPile (with Mostafa's notes about removing "new-filtered", "youtube_filtered", "stackexchange_filtered", etc., removed)

Joel's notes on the approval process:
> - RE: CommonPile: As far as I’m aware, we don’t have a blanket, general approval for CommonPile (or any datasets, yet?). I’m not sure if legal would ever give written general approval of any dataset; Essentially everything has copyrights by default, so unlikely they’d ever commit to a dataset as completely “approved”. There’s always risk using data not created or synthesized by Cerebras using purely Cerebras resources. We need to review each use and check with legal on riskier things.
> However, as Mostafa notes, the Don’t Drop Dropout paper was approved using CommonPile. We also have tacit acknowledgement that CommonPile does not violate any of the restrictions that legal has specified. It is likely one of the safest datasets we could use currently (maybe just for pretraining, fine-tuning?).

## On the general process

Joel:
> legal gives the following restrictions for datasets we should not use: [AI Training Legal Advice.docx](https://nnetx.sharepoint.com/:w:/s/team-legal/IQD-JN_ga3S0SKGiOEd3EVJXAUZDFiQCptoX6e0cmMNMAbU?e=Pb05DK) . Summarizing: Don’t use pirated or paywalled data. Don’t use data from litigious organizations. Document which data we’re using
