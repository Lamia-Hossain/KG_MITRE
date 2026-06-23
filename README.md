# MITRE ATT&CK Knowledge Graph — v2.0

A full-pipeline Neo4j knowledge graph linking MITRE ATT&CK, CAPEC, CWE, and CVE data into a single traversable graph.

---

## Graph Schema

```
(CVE)-[:EXPLOITS_WEAKNESS]->(CWE)-[:RELATED_TO]->(CAPEC)-[:MAPS_TO_ATTACK]->(Technique)-[:BELONGS_TO]->(Tactic)
(Group)-[:USES]->(Software)-[:EXECUTED]->(Procedure)-[:TARGETS]->(Technique)
(Technique)-[:HAS_SUBTECHNIQUE]->(Technique)
(CWE)-[:CHILD_OF]->(CWE)
```

---

## Prerequisites

- Python 3.8+
- Neo4j 5.x running locally on `bolt://localhost:7687`
- Default credentials: `kg_mitre_v1.1`

```bash
pip install pandas tqdm requests neo4j stix2
```

---

## Data Sources

| Layer  | Source                                                                     | Format           |
| ------ | -------------------------------------------------------------------------- | ---------------- |
| ATT&CK | [enterprise-attack.json](https://github.com/mitre/cti) — download manually | STIX 2.1 bundle  |
| CWE    | Auto-downloaded from `cwe.mitre.org`                                       | XML zip          |
| CAPEC  | Auto-downloaded from `capec.mitre.org`                                     | CSV zip          |
| CVE    | Auto-downloaded from `nvd.nist.gov` (2002–2026)                            | JSON gz per year |

---

## Pipeline

Run each cell in order inside `MITRE_CAPEC_CWE_CVE.ipynb`.

| Cell           | What it does                                                                                                                                        |
| -------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| **1 — Config** | Connects to Neo4j, creates uniqueness constraints and indexes                                                                                       |
| **2 — ATT&CK** | Ingests Tactics, Techniques, Sub-techniques, Groups, Software, Procedures from local `enterprise-attack.json`                                       |
| **3 — CWE**    | Downloads and parses CWE XML; ingests nodes with description, exploit likelihood, consequences; builds `CHILD_OF` and `RELATED_TO` (CAPEC) edges    |
| **4 — CAPEC**  | Downloads CAPEC CSV; enriches nodes with severity and likelihood; maps to ATT&CK Techniques via `MAPS_TO_ATTACK`; backfills missing CWE↔CAPEC links |
| **5 — CVE**    | Auto-downloads NVD annual feeds (2002–2026); parses CWE mappings (Primary → Secondary fallback); links CVEs to CWE nodes via `EXPLOITS_WEAKNESS`    |

---

## Output

~250,000+ nodes. Example traversal — find ATT&CK techniques reachable from a CVE:

```cypher
MATCH (v:CVE {id: "CVE-2021-44228"})-[:EXPLOITS_WEAKNESS]->(w:CWE)
      -[:RELATED_TO]->(c:CAPEC)-[:MAPS_TO_ATTACK]->(t:Technique)
RETURN v.id, w.name, c.name, t.name
```
