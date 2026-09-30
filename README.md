# LLM-Assisted Evidence Mapping of Nature-Based Solutions for Urban Heat and Health Outcomes

![Python](https://img.shields.io/badge/Python-3776AB?style=for-the-badge&logo=python&logoColor=white)
![Data: PubMed](https://img.shields.io/badge/Data-PubMed-555555?style=for-the-badge)
[![ORCID](https://img.shields.io/badge/ORCID-A6CE39?style=for-the-badge&logo=orcid&logoColor=white)](https://orcid.org/0009-0007-5849-1761)

Final project for EPID 695: AI for Public Health and Biomedicine, CUNY Graduate School of Public Health and Health Policy (May 2026).

An end-to-end pipeline that retrieves PubMed abstracts, discovers topics with BERTopic, benchmarks against a TF-IDF baseline, generates PMID-cited evidence summaries with an LLM, and audits those summaries manually for accuracy. AI-use as a tool assistant was highly encouraged for this course.

---

## Background

Extreme heat events in urban areas have become an increasing public health concern, with documented associations between heat exposure and adverse health outcomes including heat stroke and mortality. Urban environments are particularly susceptible to heat accumulation due to heat-absorbing materials such as metal, concrete, and brick, which contribute to the urban heat island (UHI) effect. In response, city planners and researchers have increasingly turned to **nature-based solutions (NBS)**, including urban green infrastructure (e.g., green walls, urban forests), blue infrastructure, and hybrid blue-green infrastructure, as a potential mitigation strategy.

Synthesizing the literature on NBS effectiveness in reducing urban heat and associated health outcomes is a timely task, and LLM-assisted methods offer a way to rapidly organize large bodies of literature, identify emerging trends, and surface gaps in evidence.

## Objective

To evaluate whether BERTopic can be effectively employed to identify the dominant types of NBS studied for urban heat mitigation, and to characterize the heat-related and health-related outcomes most commonly reported in the literature.

## Pipeline

```
PubMed API query  →  Filtering & cleaning  →  Text representation  →  Topic discovery  →  LLM summaries  →  Manual quality audit
   (323 records)       (296 abstracts)        TF-IDF (baseline)       BERTopic (primary)   GPT-4o-mini        12 abstracts,
                                              Sentence embeddings     k-means (baseline)   PMID-cited         3-point rubric
```

## Methods

### Corpus construction

Abstracts and metadata (title, abstract, publication year, journal) were retrieved from PubMed via API. The Boolean query combined terms across four domains:

1. **NBS and green infrastructure** (e.g., "green infrastructure," "urban forest\*," "nature-based solution\*," "green roofs")
2. **Urban setting** (e.g., "urban," "city," "metropolitan")
3. **Heat** (e.g., "extreme heat," "urban heat island," "heatwave\*," "thermal")
4. **Health outcomes** (e.g., "heat-related illness," "mortality," "hospitalization," "morbidity")

**Included:** empirical studies or systematic reviews in English, published 2016–2026, discussing NBS in relation to urban heat and heat-related health outcomes in an urban context.
**Excluded:** articles without any form of NBS; rural, non-metropolitan, or non-urban settings; unrelated to urban heat or heat-related health outcomes; outside the ten-year window; editorials, letters, or non-peer-reviewed publications.

### Text representation and topic discovery

| Component | Primary approach | Baseline |
|---|---|---|
| Representation | Sentence embeddings | TF-IDF |
| Clustering | BERTopic (UMAP + HDBSCAN), `nr_topics` max 10, six seed topic lists | k-means, k = 6–10 |
| Evaluation | Topic coherence and interpretability | Silhouette score |

UMAP's random state was fixed at 42 and the `umap_model` was passed directly into BERTopic, which resolved topic instability observed in earlier runs and makes results reproducible.

### LLM layer and guardrails

Clusters were labeled from their top TF-IDF keywords, with LLM-assisted labeling constrained to those keywords and associated abstracts. GPT-4o-mini then generated a one-to-two paragraph evidence summary per theme. To reduce hallucination risk and keep outputs traceable, the LLM was instructed to:

- summarize **only** from the provided abstracts,
- cite **PubMed identifiers (PMIDs)** in all outputs, and
- avoid introducing any facts not present in the source material.

### Quality check

A manual audit rated abstract–summary pairings on a three-point rubric: **accurate**, **partially accurate**, or **hallucinated**.

## Results

**Corpus:** The PubMed query returned 323 articles. After removing articles outside 2016–2026, duplicates, missing abstracts, and abstracts containing rural or non-urban terminology, **296 abstracts** remained.

**Topics:** BERTopic identified four coherent topics, below the `nr_topics` maximum of 10. 106 of 296 abstracts (36%) were assigned to the outlier cluster (Topic −1), reflecting genuine heterogeneity in a corpus spanning epidemiology, urban planning, ecology, and public health.

| Topic | LLM label | Abstracts (N) | Top keywords |
|---|---|---|---|
| 0 | Urban Heat Island Mitigation | 117 | tree planting, tree canopy, forests, trees |
| 1 | Greenspace and Respiratory Health | 19 | birth outcomes, asthma, vegetation index |
| 2 | Urban Environmental Health Assessment | 32 | environmental exposures, impact assessment, sustainable |
| 3 | Green Spaces and Mental Health | 22 | sustainability, environmental exposures, initiatives |

**Baseline comparison:** TF-IDF k-means silhouette scores were uniformly low, ranging from 0.0033 (k = 7) to 0.0042 (k = 10). This indicates that term-frequency representations could not meaningfully separate this corpus, supporting sentence embeddings as the primary method.

**Time trends:** Publication activity was low and stable across topics from 2016 to 2019, with a broad increase from 2020 onward. Topic 0 was the most consistently published theme, peaking at 19 abstracts in 2024. Topics 1 and 3 emerged largely after 2022, suggesting respiratory health, birth outcomes, and mental health framings are relatively recent additions to this literature. The 2026 decline likely reflects incomplete indexing, as the search was conducted mid-year.

**Quality audit:** Of 12 representative abstracts across all four topics, **5 were rated accurate, 7 partially accurate, and none hallucinated**. The most common issue was **PMID misattribution**: the LLM correctly captured a cluster's thematic content but assigned specific findings to the wrong abstract within the same topic. A secondary issue was **omission**, where representative abstracts included in the input were not referenced in the summary.

## Key Findings

- **Tree-based urban greening dominates the literature.** Topic 0 (n = 117) focused on tree canopy, urban forests, and UHI cooling. Green roofs, wetlands, and blue-green infrastructure did not appear as distinct clusters despite being in the query, suggesting they are understudied relative to their potential relevance.
- **Health impacts extend beyond heat stroke and mortality.** Topic 1 surfaced birth outcomes and childhood asthma, and Topic 3 captured psychological wellbeing, thermal comfort in older adults, and equity dimensions related to structural racism and environmental health disparities.
- **Direct NBS-to-health links are sparse.** Causal links between a specific NBS intervention and a measurable health outcome were inconsistently reported. Topic 2 came closest, with quantified health impact estimates from Barcelona-based modeling studies, but these were concentrated in a small number of abstracts.

## Limitations

- **Hallucination risk:** Constrained prompting produced no fabricated claims in the audit, but the model sometimes described an outcome as characteristic of a whole cluster when it appeared in only one representative abstract, and misattributed PMIDs within clusters. Future iterations should pass abstracts to the LLM individually rather than as grouped cluster input.
- **Corpus size and coverage:** The final corpus of 296 abstracts fell below the proposal's 500–1,000 target. Because PubMed primarily indexes biomedical and clinical journals, engineering, urban planning, and environmental science literature on NBS cooling effectiveness is likely underrepresented. Broader search terms and supplementary databases such as Web of Science or Scopus could expand coverage, though they may yield fewer public health–specific results.
- **Equity and generalizability:** Equity-focused abstracts were a small subset of Topic 3 and were not consistently paired with empirical heat or health outcome data. These gaps should be explicitly named in any evidence brief or policy recommendation derived from this work.

## Public Health Implications

LLM-assisted topic modeling can organize a heterogeneous public health corpus into interpretable themes within a single analytical pipeline. This approach could accelerate the scoping phase of evidence synthesis, such as identifying NBS interventions with a large evidence base for improving heat-related health outcomes, and inform research prioritization, intervention planning, and policy around urban heat adaptation. For this topic, a larger corpus, broader search terms, supplementary databases, and an extended date range are needed to generate more stable and actionable findings.

## Repository Contents

| File | Description |
|---|---|
| `murrayeileen_finalprojectepid695.py` | Full pipeline: PubMed retrieval, preprocessing, TF-IDF baseline, BERTopic, LLM summaries |
| `epid695_AUDIT.csv` | Manual quality audit of LLM summaries |
| `README.md` | Project overview |
|`murrayeileen_epid695_COPY.pptx`| Final presentation |
|`MurrayEileen_FINALPROJECT_EPID695.docx`| Full paper|

## AI-Use Disclosure

The initial code was adapted from a course lab and edited by me. ChatGPT and Google Gemini were used to help debug and explain code errors, and Claude was used to help organize and clean up the code and to edit the writing of the project report.

## References

- Grootendorst M. BERTopic. https://bertopic.com/
- National Center for Biotechnology Information. About PubMed. https://pubmed.ncbi.nlm.nih.gov/about/

## Author

**Eileen M. Murray, MPH**
Epidemiology & Biostatistics, CUNY Graduate School of Public Health and Health Policy
[ORCID: 0009-0007-5849-1761](https://orcid.org/0009-0007-5849-1761)
