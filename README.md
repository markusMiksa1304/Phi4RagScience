# Phi4RagScience
Ein modulares, LLM- basiertes  CLI-System für wissenschaftliche Recherche (pdf) und Dialoganalyse.
# Science Dialog Ensemble CLI

Ein modulares CLI-System für wissenschaftliche Recherche und Dialoganalyse.

## Features
- **Dialog-CLI** mit Short-Term-Memory (STM) und fortlaufendem **Plan** (Goals, Hypothesen, To-Dos).
- **Retrieval-Lanes**:
  - **Short**: Google (SerpAPI) → schnelle Hinweise.
  - **Medium**: Wikipedia → kompakte Primer.
  - **Slow**: arXiv dynamisch → Abstracts + optionale **PDF-Chunking-Indizierung**.
- **Chunking-Plattform**: struktur-aware Chunking + Hybrid-Retrieval (BM25 + FAISS + RRF + MMR).
- **Understanding-Layer** (Mini-LLM): Facetten/Synonyme/Variablen → Subqueries; Vetting (Coverage, Semantik, optional NLI, optional Mini-LLM-Rescore).
- **Tandem-Antwort**: **Phi-4 Mini lokal** + **OpenAI** (konfigurierbar: phi4_only, openai_only, parallel).
- **Claim→Evidence Filter** + **Metriken** (Support-Rate, Wilson-Lower-Bound).


Hier ist das vollständige, abgestimmte, lauffähige Gesamt-System mit Doppel-LLM, Dialog-CLI, STM, Plan, Lanes (Short/Medium/Slow), Chunking-Plattform, Understanding-Layer, Orchestrator und Haupt-CLI.


## Installation
pip install -r requirements.txt
OpenAI-Key: als Umgebungsvariable OPENAI_API_KEY (oder .env).


# 1) Dialog (STM + Plan)
python science_cli_ensemble.py dialog

# 2) Einzelschuss-Chat (ohne Plan, ein Turn)
python science_cli_ensemble.py chat --query "Vergleich Self-Attention-Varianten 2024/25"

# 3) arXiv-Suche + optional PDF-Chunking
python science_cli_ensemble.py search --query "graph neural networks" --download-top 2

# 4) Indizes ansehen / neu bauen
python science_cli_ensemble.py show-index
python science_cli_ensemble.py reindex

# 5) Smoke-Test
python science_cli_ensemble.py selftest



#Zusammenfassung
Du hast jetzt ein vollständiges, lauffähiges System mit:
Dialog-CLI (STM + Plan),
Short/Medium/Slow-Lanes inkl. dynamischer arXiv + PDF-Chunking,
Chunking-Plattform (strukturbewusst, Hybrid-Retrieval),
Understanding-Layer (Plan, Subqueries, Vetting),
Orchestrator,
Doppel-LLM (Phi-4 lokal + OpenAI),
Claim→Evidence-Filter und Metriken,
Entry-Point science_cli_ensemble.py.

#Hinweise:

Phi-4 lokal: Lädt ohne 4-bit, damit keine zusätzliche Abhängigkeit (bitsandbytes) nötig ist. Auf CPU langsamer; wenn’s kneift, stell in config.json mode: "openai_only".


SerpAPI ist optional (Short-Lane). Wikipedia ist standardmäßig an.
PDF-Chunking ist standardmäßig an (policy: "chunked"). Du kannst es abschalten oder auf „full“ stellen (nicht empfohlen auf 8 GB).
Wenn du magst, kann ich dir noch eine minimale config.json hier ablegen – sag Bescheid, dann pack ich’s sofort dazu.

---

# 📄 requirements.txt
sentence-transformers
faiss-cpu
rank-bm25
feedparser
requests
wikipedia
openai>=1.0.0
pdfplumber
torch
transformers>=4.40.0
python-dotenv

#pip install -r requirements.txt
#Ordner (werden bei Bedarf automatisch angelegt):
./abstracts   ./pdfs   ./chunks

#Erster Test

# Dynamische arXiv-Suche + 1 PDF indizieren
python science_cli_ensemble.py search --query "graph neural networks" --download-top 1

# Einzelschuss
python science_cli_ensemble.py chat --query "Erkläre kurz Backpropagation."

# Dialog (STM + Plan)
python science_cli_ensemble.py dialog

#Nützliche Umschalter (falls’s hakt)

Nur OpenAI (schnell, wenig RAM): "mode": "openai_only"
Kein PDF-Download: "pdf": { "policy": "chunked", "top_k": 0 }
Wikipedia aus: "lanes": { "wikipedia": false, ... }
NLI präziser machen: "understanding": { "nli": true } (langsamer)

#Wenn du willst, geb’ ich dir noch eine zweite config.openai-only.json und config.cpu.json (konservativ für 8 GB) dazu.

#config.openai-only.json
{
  "ensemble": {
    "mode": "openai_only",
    "openai_model": "gpt-4o-mini",
    "phi4_model": "microsoft/Phi-4-mini-instruct",
    "max_new_tokens": 220
  },
  "lanes": {
    "wikipedia": true,
    "serpapi": false,
    "serpapi_key": ""
  },
  "rag": {
    "threshold": 0.93,
    "max_results": 12
  },
  "pdf": {
    "policy": "chunked",
    "top_k": 1
  },
  "understanding": {
    "enable": true,
    "nli": false,
    "llm_rescore_top": 2
  },
  "context": {
    "top_chunks": 8
  },
  "paths": {
    "abstracts": "./abstracts",
    "pdfs": "./pdfs",
    "chunks": "./chunks"
  }
}

#config.cpu-conservative.json (8-GB-Laptop, maximal sparsam)
{
  "ensemble": {
    "mode": "openai_only",
    "openai_model": "gpt-4o-mini",
    "phi4_model": "microsoft/Phi-4-mini-instruct",
    "max_new_tokens": 180
  },
  "lanes": {
    "wikipedia": true,
    "serpapi": false,
    "serpapi_key": ""
  },
  "rag": {
    "threshold": 0.94,
    "max_results": 10
  },
  "pdf": {
    "policy": "chunked",
    "top_k": 0
  },
  "understanding": {
    "enable": true,
    "nli": false,
    "llm_rescore_top": 1
  },
  "context": {
    "top_chunks": 6
  },
  "paths": {
    "abstracts": "./abstracts",
    "pdfs": "./pdfs",
    "chunks": "./chunks"
  }
}

#Wenn du später Phi-4 lokal zuschalten willst: in config.json ensemble.mode auf "parallel" stellen.

Für SerpAPI-Short-Lane: lanes.serpapi=true und lanes.serpapi_key setzen.
Wenn Antworten zu lang werden: ensemble.max_new_tokens weiter runter (150–180).

#1. Einzelschuss-Modus (wie `chat --query "...")

Jede Frage wird per CLI-Argument übergeben.
Kein Short-Term-Memory, kein Plan.
Praktisch für schnelle Tests oder Skripte.
#Beispiel:
python science_cli_ensemble.py chat --query "Was ist Backpropagation?"

#2. Dialog-Modus (Subcommand dialog)

Startest eine REPL (= Read-Eval-Print Loop).
Du bleibst im Prompt, tippst nacheinander deine Fragen.
System merkt sich den STM (z. B. letzte 6 Turns) und pflegt den Plan (plan.json).
Du kannst zwischendurch Kommandos nutzen wie:

:plan → aktuellen Plan anzeigen
:ctx → aktuellen Kurzzeit-Kontext ansehen
:save / :load → Plan persistieren oder laden
:reset → Dialog-Kontext löschen

#Beispiel:

python science_cli_ensemble.py dialog

#Dann:

> Erklär mir kurz Self-Attention
[Antwort]

> Und was ist die Rechenkomplexität?
[Antwort]

> :plan
[Zeigt Goals/Hypothesen/Todos]

Fazit

👉 Ein echter Dialog ist möglich, wenn du mit dialog startest.
👉 Wenn du nur chat --query nutzt, musst du jede Frage separat aufrufen.

python science_cli_ensemble.py dialog

#Was du im Dialog machen kannst

Einfach Fragen tippen – STM (Kurzzeitgedächtnis) + Plan laufen mit.
Befehle:
:plan – aktuellen Forschungs-Plan zeigen (goals/hypotheses/todos/notes/sources)
:ctx – Kurzzeit-Kontext (letzte ~6 Turns) anzeigen
:save / :load – Plan speichern/laden (./plan.json)
:reset – STM & Plan zurücksetzen
Ctrl+C – beenden

Vorher kurz prüfen
config.json passt (z. B. ensemble.mode: "parallel" oder "openai_only").
OPENAI_API_KEY gesetzt (für die OpenAI-Lane).
Optional: lanes.serpapi=true + lanes.serpapi_key für die Short-Lane.

#Wie du’s nutzt:

Konsole + Datei-Log:
python science_cli_ensemble.py --log-level DEBUG --log-file sci.log dialog
→ Konsole: kompakt. Datei sci.log: detailliert (mit Datei/Zeile).

#test-stapel
py hello_session.py
# → schreibt detailierten Log nach hello.log und zeigt Antworten/Metriken an

Wenn du später noch feinere Traces auf Modul-Ebene willst, kann ich zusätzlich in orchestrator.py, lanes.py, chunking_platform.py und understanding_layer.py interne log.debug(...)-Statements einziehen (z. B. pro Subquery/Ranking/Score). Für jetzt bekommst du den vollen Ablauf sichtbar – ohne die anderen Module zu verbiegen.
