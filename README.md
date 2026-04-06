# Reducing False Positives in Financial Compliance Screening: Better Entity-Matching

<img width="974" height="1706" alt="image" src="https://github.com/user-attachments/assets/cf0021a6-3329-4ea8-b946-5ea00f5a786c" />


---

## Note

This research was conducted (in 2024) while I was working for a payment facilitator serving Muslim communities — a context where Arabic and Muslim naming conventions produce disproportionately high false positive rates in sanctions screening and adverse media searches. The report maps the full problem: the scale of false positives across the industry, the cost structure, the root cause (name matching without entity resolution), and the specific mechanics of why Arabic naming conventions — patronyms, high-frequency given names, transliteration variability — break existing screening systems. 

The key solution proposed in the report was multi-attribute entity matching on structured KYC data, with Fellegi-Sunter probabilistic weighting. On unstructured adverse media, NER-based entity extraction is proposed. Prototypes were created to illustrate the shift from string matching to entity resolution. 

Since this research was conducted, the compliance industry has moved aggressively in the direction this analysis pointed toward. Entity resolution is now the standard framing, but using far more data points than this research imagined. More significantly, the industry has leapfrogged rule-based and classical ML approaches entirely in favor of agentic AI systems: autonomous agents that conduct full alert investigations, write disposition narratives, and clear false positives with explainable audit trails. The prototypes here are not competitive with these commercial solutions. They were never intended to be. They demonstrate that the problem facing the compliance industry and customers with Arabic/Muslim names that were debanked, could be solved with the right technical architecture.



---

## 1. The Problem

Financial compliance — sanctions screening, AML monitoring, and fraud prevention — is plagued by extraordinarily high false positive rates. The European Central Bank reported in 2020 that **98–99% of compliance alerts are false positives**. Oracle and LexisNexis corroborate this, citing rates as high as 95% across financial crimes monitoring. In fraud prevention, up to 35% of rejected orders turn out to be legitimate, trending upward year over year (JPMorgan).

The root cause is deceptively simple: the industry's screening tools are optimized for **name matching**, not **entity matching**. A name is a string. An entity is a person — with a date of birth, a location, a transaction history, an employment record. Screening systems that cannot distinguish between two people who happen to share a name will, by construction, generate false positives proportional to the frequency of that name in the population.

This problem is not distributed evenly. Arabic and Muslim naming conventions — high-frequency given names (Muhammad, Abdullah), patronymic structures, tribal lineage markers, and transliteration variability across scripts — produce false positive rates far exceeding those for Western European names. One payment platform reported that after migrating to a large bank's compliance software, **50% of its UK donors with Muslim or Arabic names were rejected**.

---

## 2. Costs of False Positives

False positives are more expensive than actual fraud. In the US, an estimated **$118 billion** in sales were wrongly declined due to false positives — compared to $9 billion lost to actual card fraud. The costs cascade across several dimensions:

### 2.1 Labor

Every flagged alert requires manual review by AML compliance specialists. Global labor costs for financial crime compliance are estimated at **$274.1 billion** (up from $180.9B in 2019), with $61B in North America alone (LexisNexis & Forrester, 2023). For banks, managing a single blocked transaction costs €1.50–€5.00.

### 2.2 Customer Friction

33% of consumers said they would not return to a business after a false decline (Stripe). 61% of users who experienced a false positive blocking reduced their card usage or stopped entirely. This directly impacts straight-through processing rates — a key competitive metric for payment processors.

### 2.3 Regulatory Scrutiny

Regulators view high false positive rates not as evidence of caution, but as evidence of **failure**. ACAMS states plainly: "High false positive rates are not an indicator of extremely cautious screening: they are a warning signal of poor technology and potentially greater risks." Under the Bank Secrecy Act, payment processors must file Suspicious Activity Reports with FinCEN — excessive filings trigger regulatory scrutiny.

### 2.4 Analyst Fatigue

A process that over-cultivates false positives also misses true positives and false negatives. AML specialists consider over-cultivation a weakness of due diligence, not a strength. Alert fatigue degrades the quality of review across the board.

---

## 3. Causes of False Positives

### 3.1 Asymmetric Penalty Structure

Missing a true positive carries catastrophic risk — nearly **$5 billion** in fines were levied for AML non-compliance in 2022 alone (LexisNexis). HSBC was fined $1.9B in part for failing to flag transactions involving drug cartel members on blacklists. This asymmetry rationally drives institutions toward over-screening, but the resulting imprecision is self-defeating: the same blunt process that generates false positives also misses true positives.

### 3.2 Rules-Based Legacy Systems

Most banks and payment processors still rely on deterministic, rules-based transaction monitoring. A typical rule: flag if (customer risk profile = high) AND/OR (transaction amount > threshold) AND/OR (new merchant). These conjunctive/disjunctive rules are brittle, context-blind, and generate high volumes of alerts by design.

### 3.3 Name Matching Without Entity Resolution

This is the core technical failure. Current screening tools match **strings**, not **entities**. They use deterministic matching (exact match on specified fields) or simple fuzzy matching (Levenshtein distance, Soundex). These work passably for small, homogeneous datasets but break down when:

- **Name frequency is high**: "Muhammad" appears on sanctions lists and also belongs to millions of legitimate customers
- **Patronymic conventions create structural similarity**: "Abdullah bin Muhammad" and "Muhammad bin Abdullah" share all the same tokens
- **Transliteration is non-deterministic**: عبدالله can be rendered as Abdullah, Abdallah, Abd Allah, Abdulla — each a valid romanization
- **Entity types are conflated**: "Carmela" is both a common given name and a vessel owned by the Islamic Republic of Iran Shipping Lines; legacy systems do not distinguish between person and vessel entity types

### 3.4 Poor Data Quality and Structure

Transaction messages do not always provide structured entity information. Names arrive as single undelimited strings. Secondary identifiers (DOB, nationality, passport number) may be missing, inconsistent, or formatted heterogeneously. The problem is not too little data — it is an **overabundance of unstructured data** and an inability to process it effectively.

### 3.5 Arabic and Muslim Naming Conventions

Arabic names present specific challenges that compound the above:

| Feature | Effect on Screening |
|---|---|
| High-frequency given names (Muhammad, Ahmad, Ali) | Exponentially more partial matches against sanctions lists |
| Patronyms (bin/ibn + father's name) | Structural similarity across unrelated individuals |
| Tribal/clan names (al-Rashidi, al-Tamimi) | Shared surname components across large populations |
| Nasab chains (multi-generational lineage) | Long names with many matchable substrings |
| Transliteration variability | Same person may appear under multiple spellings |
| Honorifics and religious titles | Additional tokens that may or may not appear on lists |

The issue is not that Arabic names are harder to spell. It is that Arabic naming conventions produce a **higher collision rate** in any name-matching system that lacks entity resolution.

---

## 4. Solutions

### 4.1 Better Name-Matching Algorithms

Fuzzy matching (Jaro-Winkler, Levenshtein) and phonetic encoding (Soundex, Double Metaphone) improve recall but can worsen precision — they generate more partial matches, increasing both true hits and false positives. Better name-matching is necessary but insufficient.

### 4.2 Multiple-Attribute Matching

The key insight: move from matching **names** to matching **entities** by incorporating secondary identifiers. OFAC's SDN list already provides rich secondary data — name variations, DOB, POB, nationality, passport numbers. Effective screening must leverage all of these.

See `prototypes/multi_attribute_matching.py` for a working implementation.

### 4.3 Weighted Probabilistic Matching (Fellegi-Sunter)

Rather than binary match/no-match, compare multiple field values and assign each a weight reflecting its discriminating power. The sum of weights yields a match probability. High-frequency names like "Muhammad" receive lower weight (less discriminating); rare passport numbers receive higher weight.

See `prototypes/fellegi_sunter.py` for a working implementation.

### 4.4 ISO 20022 Structured Payment Data

The migration to ISO 20022 messaging provides a significant opportunity. Unlike legacy SWIFT MT messages (unstructured, 35-character name fields), ISO 20022 pacs.008 messages carry structured party identification:

```xml
<Dbtr>
  <Nm>Muhammad Ahmad Al-Rashidi</Nm>
  <PstlAdr>
    <Ctry>CA</Ctry>
    <AdrLine>123 Main St, Toronto</AdrLine>
  </PstlAdr>
  <Id>
    <PrvtId>
      <DtAndPlcOfBirth>
        <BirthDt>1985-03-15</BirthDt>
        <CityOfBirth>Toronto</CityOfBirth>
        <CtryOfBirth>CA</CtryOfBirth>
      </DtAndPlcOfBirth>
    </PrvtId>
  </Id>
</Id>
```

This structured data enables multi-attribute matching at the transaction level, reducing dependence on name-only screening.

See `prototypes/iso20022_screening.py` for a working implementation.

### 4.5 Named Entity Recognition for Entity Resolution

NER models (spaCy, HuggingFace transformers) can extract structured entity information from unstructured adverse media text, enabling entity resolution rather than name matching. A BERT-based NER model fine-tuned on financial compliance data can identify not just names but their associated attributes — nationality, role, organization, location — and use these to disambiguate.

See `prototypes/ner_entity_resolution.py` for a working implementation.

### 4.6 Risk-Tiered Screening

Assign customers to risk tiers based on product type, transaction geography, behavioral patterns, and PEP status. Apply different screening thresholds and list configurations per tier. Monitor high-risk customers more closely while reducing friction for verified low-risk customers.

### 4.7 Verified Customer Whitelists

After initial screening and manual clearance, maintain a whitelist of verified customers to prevent re-flagging on subsequent transactions. This requires ongoing monitoring for changes in sanctions lists and customer status.

---

## 5. Existing Solutions Landscape

### Data Providers
- **Thomson Reuters CLEAR**: 20M+ adverse media sources across 120+ countries, 900+ sanctions lists, 1M+ PEPs
- **Refinitiv World-Check**: Global PEP and sanctions data
- **Dow Jones Risk & Compliance**: Sanctions and adverse media API
- **ComplyAdvantage**: AML data and screening

### Screening Platforms
- **GBG**: Compliance platform with ML-driven matching across 350 watchlists; notable for explicit handling of Arabic name patterns
- **LexisNexis Bridger Insight**: Entity resolution with risk scoring
- **Quantexa**: Decision Intelligence platform (used by Standard Chartered, HSBC)
- **Socure**: Claims 30% reduction in false positives, 75% reduction in manual review time vs. legacy

### Case Studies
- **Danske Bank**: Reduced false positives by 60% and increased true positives by ~50% using a custom AI solution built on Stanford NER
- **Standard Chartered & HSBC**: Deployed Quantexa's Decision Intelligence Platform

---

## 6. Regulatory Framework

| Year | Framework | Relevance |
|---|---|---|
| 1970 | Bank Secrecy Act | CTR filing, SAR requirements |
| 1989 | Financial Action Task Force (FATF) | 40 recommendations on AML/CTF |
| 2001 | USA PATRIOT Act | 50+ amendments to BSA |
| 2018 | FinCEN CDD Rule | Customer due diligence requirements |
| 2021 | EU 6th AML Directive | 22 predicate offenses, adverse media requirements for high-risk regions |

Key statutes governing sanctions violations: Bank Secrecy Act (BSA), International Emergency Economic Powers Act (IEEPA, penalties up to $250K/violation), Trading with the Enemy Act (TWEA). Intentional violations carry criminal penalties up to $10M and 30 years imprisonment.

---

## 7. Prototypes

This repository includes working prototypes demonstrating the solutions described above:

| Prototype | Description |
|---|---|
| `prototypes/multi_attribute_matching.py` | Multi-attribute entity matching using structured KYC data and ISO 20022 fields |
| `prototypes/fellegi_sunter.py` | Probabilistic record linkage using the Fellegi-Sunter model with frequency-aware weights |
| `prototypes/iso20022_screening.py` | Sanctions screening against OFAC SDN list using structured ISO 20022 payment data |
| `prototypes/ner_entity_resolution.py` | NER-based entity extraction from adverse media, with entity resolution for Arabic names |
| `prototypes/demo.py` | End-to-end demonstration combining all approaches |

### Running the Prototypes

```bash
# Core prototypes (no dependencies required):
python prototypes/demo.py

# For full NER capability (optional):
pip install spacy transformers torch
python -m spacy download en_core_web_sm
```

---

## Sources

1. EY, *Sanctions Screening Industry Survey*, 2021
2. Thomson Reuters, *False Positives and False Negatives* (white paper)
3. Vorbyev & Krivitskaya, "Reducing false positives in bank anti-fraud systems," *Computers & Security*, 2022
4. Cognizant, *OFAC Name Matching and False-Positive Reduction Techniques*, 2014
5. JPMorgan, "When false positives spiked, company abandoned fraud prevention tools," 2023
6. LexisNexis & Forrester, *True Cost of Financial Crime Compliance Global Study*, 2023
7. KPMG, *Forensic Sanctions Survey*, 2023
8. GBG, *Global Fraud Report*, 2024
9. ACAMS, certification and industry guidance
10. Wolfsberg Group, AML/KYC/CTF frameworks
