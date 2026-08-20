# Data Design & RAG Thinking

## Ce date sunt necesare

Două categorii de date, cu roluri diferite:

1. **Tichete mock** (pentru antrenare/testare a clasificatorului) — 200-500 înregistrări.
2. **Bază de cunoștințe (KB)** pentru retrieval — articole de rezolvare, playbook-uri de
   rutare, nu neapărat tichete individuale.

## Entități / schemă aproximativă

### Tichet (mock data)

| Câmp | Tip | Descriere |
|---|---|---|
| `id` | string | identificator unic |
| `timestamp` | datetime | momentul deschiderii |
| `subject` | text scurt | titlul tichetului |
| `description` | text liber | descrierea completă a problemei |
| `reporter_role` | categorie | ex. end-user, sysadmin, monitoring-system |
| `system_affected` | categorie | ex. email, VPN, ERP, network, hardware |
| `true_priority` | enum P1-P4 | eticheta de referință (ground truth) |
| `true_category` | categorie | eticheta de referință |
| `true_team` | categorie | echipa corectă (ground truth) |

### Document KB (pentru retrieval)

| Câmp | Tip | Descriere |
|---|---|---|
| `doc_id` | string | identificator unic (folosit ca citare) |
| `title` | text | titlul articolului/playbook-ului |
| `content` | text | conținut (simptome tipice, pași, echipă responsabilă) |
| `category` | categorie | pentru filtrare rapidă |
| `embedding` | vector | generat la indexare, stocat în ChromaDB |

## Strategia de date mock

- Generare asistată de LLM (Groq, gratis), în batch-uri, cu prompt-uri variate pentru
  a acoperi diversitate reală: incidente hardware, acces/permisiuni, rețea, aplicații
  business, probleme de performanță.
- Distribuție realistă a priorităților — P1 rar (~5%), P2 puțin mai frecvent (~15%),
  P3/P4 majoritatea (~80%) — reflectă realitatea unui helpdesk (majoritatea tichetelor
  nu sunt urgențe).
- Split explicit: un subset (~15-20%) rezervat exclusiv ca **set de test**, nefolosit
  nici pentru retrieval, nici pentru eventual fine-tuning — pentru evaluare corectă.

## Ce informație are nevoie de retrieval/căutare

- **Tichete istorice similare**: pentru a da context LLM-ului ("tichete de acest fel au
  fost de obicei P2, rutate la echipa Network").
- **Playbook-uri/articole KB**: pentru cazuri unde există o procedură documentată
  (ex. "resetare parolă VPN" → echipă + prioritate standard cunoscute).

Ce **nu** are nevoie de retrieval: tichete complet noi/unice, fără precedent — acestea
rămân cazuri de graniță, candidate naturale pentru aprobare umană (confidence scăzut =
puține potriviri relevante găsite în retrieval).

## Utilizare ChromaDB

- O singură colecție (sau două separate: `tickets_history` și `kb_articles`), fiecare
  document cu metadata (`category`, `true_team` dacă e disponibil) pentru filtrare
  hibridă (semantic + filtru pe metadata).
- La query: embedding al tichetului nou → top-k (ex. k=5) cele mai similare din ambele
  colecții → documentele + scorurile devin parte din promptul trimis LLM-ului, iar
  `doc_id`-urile devin citările din output-ul final (`TicketClassification.citations`).
- Evaluare RAG separată cu RAGAS (faithfulness, context precision) pe un set mic de
  perechi întrebare/răspuns de referință, distinct de setul de test al clasificării.
