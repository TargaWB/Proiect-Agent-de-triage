# Agent de Triaj ITSM — Propunere de proiect

Deliverable de tip propunere (nu implementare completă) — evaluat pe 7 criterii, 100p total.
Vezi `docs/architecture.md` pentru arhitectura tehnică detaliată și `docs/data-design.md`
pentru schema de date și strategia RAG.

---

## 0. Rezumat — problemă, abordare, rezultat, impact

**Problema**: triajul manual al tichetelor IT (citire, evaluare, decizie de prioritate și
rutare) e inconsistent între operatori și devine blocaj în vârfuri de volum — vezi
secțiunea 1 și 2 pentru detalii.

**Abordarea**: un pas de clasificare asistată de AI, cu RAG pe istoricul de tichete și
KB, care propune prioritate/categorie/echipă motivat și citat, dar care nu execută
nimic direct — decizia trece printr-un strat determinist de validare, iar cazurile
riscante ajung obligatoriu la aprobare umană (vezi secțiunea 3, 4, 6).

**Rezultatul așteptat**: triaj mai rapid și mai consistent pentru cazurile clare (auto-rutare),
cu omul concentrat exact pe cazurile care merită atenția lui — nu pe toate. Verificabil
prin acuratețea clasificării față de un baseline manual și prin procentul de cazuri care
chiar au nevoie de intervenție umană (secțiunea 7).

**Impact / valoare de business**:
- **Timp recâștigat pentru echipa de suport**: dispecerii nu mai citesc fiecare tichet de
  la zero — timpul lor se mută spre cazurile ambigue/riscante, unde judecata lor chiar
  contează, nu spre triaj repetitiv.
- **Consistență ca produs**: clasificarea nu mai depinde de care operator a preluat
  tichetul — un standard aplicat uniform, cu motivare vizibilă (citări din KB/istoric),
  ceea ce crește încrederea echipei și a managementului în decizii.
- **Urmă auditabilă**: fiecare decizie (automată sau aprobată de om) e loggată — util
  atât pentru îmbunătățirea continuă a sistemului, cât și pentru raportare către
  management (cât se auto-rezolvă, cât necesită om, unde apar erorile).
- **Relevanță pentru domeniu (ITSM/AIOps)**: triajul e primul pas dintr-un lanț mai lung
  (diagnosticare, remediere, escaladare) — un triaj mai bun reduce eroarea propagată în
  tot restul procesului, chiar dacă restul lanțului rămâne manual în acest proiect.

**Praguri de decizie (human-in-the-loop) — fixate explicit**:

| Condiție | Acțiune | Justificare |
|---|---|---|
| `priority == "P1"` | întotdeauna aprobare umană | impact major, cost al greșelii foarte mare |
| `confidence < 0.70` | aprobare umană | sub prag, rata de eroare empirică crește semnificativ (se calibrează pe setul de validare) |
| `suggested_team` necunoscută | aprobare umană | risc de rutare complet greșită |
| altfel | auto-rutare | risc scăzut, ușor reversibil prin re-rutare manuală ulterioară |

---

## 1. Problem definition & scope (15p)

**Problema de business/IT**: tichetele de incident IT primite prin ITSM (ex. ServiceNow-style)
sunt triate manual de un dispecer/agent nivel-1: citește descrierea, decide prioritatea
(P1-P4) și echipa țintă, apoi rutează. Procesul e lent, inconsistent între operatori
diferiți, și devine blocaj când volumul de tichete crește (ex. incident major, vârf de
solicitări).

**Obiectiv**: un agent care propune automat clasificarea (prioritate + categorie + echipă)
pe baza descrierii tichetului și a istoricului similar, cu decizie finală automată pentru
cazurile clare și escaladare la om pentru cazurile riscante sau ambigue.

**Scope**:
- IN: clasificare (P1-P4), categorizare, sugestie de echipă țintă, rutare, explicație a
  deciziei (motiv + surse folosite).
- OUT (nu face parte din acest proiect): diagnosticare tehnică a incidentului, remediere
  automată, orchestrare de incidente majore, dezvoltarea unui KEDB nou.

**Asumpții**:
- Tichetele sunt text liber în limba română/engleză, structură minimă (subiect + descriere).
- Există un istoric de tichete rezolvate anterior, suficient pentru retrieval (RAG).
- Taxonomia de echipe/categorii e fixă și cunoscută dinainte.

**Excluderi**:
- Nu se integrează cu un ITSM real în producție (se lucrează pe date mock).
- Nu se ocupă de SLA tracking sau escaladare automată pe timp (doar pe conținut).

---

## 2. Understanding of the process (AS-IS) (15p)

**Metoda tradițională**:
1. Utilizator/sistem monitorizare deschide tichet cu descriere liberă.
2. Un dispecer (om) citește tichetul, evaluează impactul și urgența, decide prioritatea.
3. Dispecerul caută manual în cunoștințele proprii sau într-o bază KB dacă a mai văzut
   ceva similar.
4. Rutează tichetul către echipa pe care o crede potrivită.
5. Dacă greșește ruta, tichetul se întoarce, se re-rutează — timp pierdut.

**Blocaje identificate**:
- **Inconsistență**: doi dispeceri pot clasifica diferit același tip de tichet.
- **Timp de răspuns**: în vârfuri de volum, tichetele stau în coadă înainte să fie triate.
- **Cunoaștere tacită**: experiența de a recunoaște tipare similare stă doar în capul
  dispecerilor experimentați — nu e reutilizabilă ușor de restul echipei.
- **Fără urmă auditabilă**: rareori se documentează *de ce* s-a ales o anumită prioritate.

**Ce poate îmbunătăți AI-ul**:
- Recuperarea automată a tichetelor istorice similare (RAG) — pune la dispoziție "memoria"
  echipei, nu doar a unei singure persoane.
- Clasificare consistentă, cu motivare explicită și citări către sursele folosite.
- Timp de triaj redus pentru cazurile clare (auto-rutare), păstrând omul în buclă exact
  acolo unde decizia e riscantă sau ambiguă.

---

## 3. Proposed solution / TO-BE flow (15p)

Sistemul propus introduce un pas de clasificare asistată de AI **între** deschiderea
tichetului și rutarea finală, cu punct explicit de aprobare umană pentru cazurile riscante.

```
TO-BE:
Tichet nou → Retrieval (tichete similare + KB) → Clasificare AI (prioritate, categorie, echipă)
    → validare structurată → decizie automată (risc scăzut) SAU aprobare umană (risc ridicat)
    → rutare finală + jurnal auditabil (motiv, surse, cine a aprobat)
```

Diferența față de AS-IS: dispecerul uman nu mai citește fiecare tichet de la zero — el
intervine doar acolo unde sistemul semnalează incertitudine sau risc, cu context deja
pregătit (clasificare propusă + motivare + surse similare).

Diagrama de flux detaliată (state machine + sequence user↔AI) e în `docs/architecture.md`,
secțiunile 5 și 8.

---

## 6. Reasoning / decision / execution concept (10p)

La nivel general (fără a intra în detaliile de implementare a agenților, care se discută
în modulul următor):

- **Reasoning** (LLM): propune clasificarea pe baza tichetului + context recuperat prin RAG.
  Rezultatul e un obiect structurat (JSON), nu o acțiune.
- **Decizie** (cod determinist): validează structura propusă și aplică praguri de risc
  fixe pentru a decide dacă acțiunea se execută automat sau necesită aprobare.
- **Execuție** (tool-uri deterministe): rutarea efectivă a tichetului, complet separată de
  LLM — LLM-ul nu are acces direct la sistemele de execuție.

Separarea asta e intenționată: LLM-ul nu execută niciodată direct — doar propune, iar
execuția trece mereu prin validare + (dacă e cazul) aprobare umană.

---

## 7. KPIs & success criteria (10p)

Nu se cere estimare sau demo la această etapă — doar metoda de verificare a succesului.

1. **Acuratețe clasificare**: proporția de tichete unde prioritatea/categoria propusă de
   agent coincide cu eticheta de referință (ground truth), măsurată pe un set de test
   separat de datele folosite pentru context/retrieval.
2. **% intervenție umană**: proporția de tichete care ajung la aprobare umană (risc ridicat)
   din totalul de tichete procesate — un indicator indirect al cât de mult reduce sistemul
   volumul de decizii manuale.

Metoda de verificare: se compară performanța agentului pe setul de test cu un baseline
manual (etichetare umană existentă în datele istorice, folosită ca ground truth), pe
aceleași tichete, folosind aceleași metrici.

---

## Structura repo

```
triaj-agent-proposal/
├── README.md               ← acest fișier (rezumat + criteriile 1, 2, 3, 6, 7)
├── docs/
│   ├── architecture.md     ← criteriul 4 (arhitectură + diagrame tehnice)
│   └── data-design.md      ← criteriul 5 (date & RAG)
└── slides/
    └── triaj-agent-proposal.pptx  ← prezentare sumar, toate criteriile
```
