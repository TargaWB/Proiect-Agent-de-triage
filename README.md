# Agent de Triaj ITSM

Propunere de proiect pentru un agent care ajută la triajul tichetelor IT: clasificare
pe prioritate, categorie și echipă țintă, cu un pas de aprobare umană pentru cazurile
riscante. Arhitectura tehnică completă e în `docs/architecture.md`, iar schema de date
și strategia de RAG în `docs/data-design.md`.

---

## Problema

Când vine un tichet nou într-un sistem ITSM, cineva trebuie să-l citească, să-i dea o
prioritate (P1-P4), să-l încadreze într-o categorie și să-l trimită la echipa potrivită.
În practică, asta se face manual, de un dispecer — și are câteva probleme cronice:

- **Inconsistență**: doi oameni pot clasifica diferit tichete foarte asemănătoare.
- **Blocaje la volum mare**: în vârfuri de trafic, tichetele stau în coadă înainte să
  ajungă la cineva care să le trieze.
- **Cunoaștere care nu circulă**: experiența de "am mai văzut așa ceva" rămâne în capul
  dispecerilor cu vechime, nu e ușor transferabilă restul echipei.
- **Nimic auditabil**: rar se scrie undeva *de ce* s-a ales o anumită prioritate, deci e
  greu de urmărit sau de îmbunătățit procesul.

## Ce propunem

Un pas de clasificare asistată de AI, inserat între deschiderea tichetului și rutarea
finală. Sistemul caută tichete similare din istoric și articole relevante dintr-o bază
de cunoștințe (RAG), propune o clasificare motivată și citată, iar apoi:

- dacă riscul e scăzut, rutează automat;
- dacă nu, trece obligatoriu prin aprobare umană, cu tot contextul deja pregătit.

```
Tichet nou → retrieval (istoric + KB) → clasificare AI → validare structurată
    → auto-rutare (risc scăzut) SAU aprobare umană (risc ridicat)
    → rutare finală, cu jurnal complet al deciziei
```

Diferența față de procesul actual: dispecerul nu mai citește fiecare tichet de la zero.
Intervine doar unde sistemul semnalează incertitudine sau risc — cu o propunere deja
motivată în față, nu cu o pagină goală.

## Cum e împărțită munca între AI și cod

LLM-ul nu execută nimic direct — doar propune o clasificare, sub forma unui obiect
structurat (validat cu Pydantic). Tot ce urmează după — validare, decizie, rutare — e
cod determinist, fără AI implicat:

- **Reasoning** (LLM): pe baza tichetului și a contextului recuperat prin RAG, propune
  prioritate, categorie și echipă.
- **Decizie** (cod): verifică dacă propunerea respectă schema așteptată și aplică
  pragurile de risc de mai jos, ca să stabilească dacă merge mai departe automat sau are
  nevoie de om.
- **Execuție** (tool-uri): rutarea efectivă. LLM-ul nu are niciodată acces direct la
  sistemele care execută acțiuni.

*(Identificarea explicită a agenților și a protocoalelor de handoff se detaliază separat,
în arhitectura tehnică — aici e doar conceptul de separare reasoning/execuție.)*

## Praguri de decizie — unde intervine omul

| Condiție | Ce se întâmplă | De ce |
|---|---|---|
| prioritate P1 | aprobare umană, mereu | costul unei greșeli e prea mare ca să fie lăsat pe automat |
| confidence sub 0.70 | aprobare umană | sub acest prag, rata de eroare crește vizibil (calibrat pe setul de validare) |
| echipă țintă necunoscută | aprobare umană | risc de rutare complet greșită |
| orice altceva | auto-rutare | risc scăzut, ușor de corectat ulterior dacă greșește |

## Ce ar aduce în plus

- **Timp recâștigat**: dispecerii nu mai pierd timp pe triajul repetitiv al cazurilor
  clare — atenția lor se duce spre ce chiar contează.
- **Consistență**: clasificarea nu mai depinde de cine a preluat tichetul, iar motivarea
  vizibilă (citări din istoric/KB) crește încrederea în decizii.
- **Urmă auditabilă**: fiecare decizie, automată sau aprobată de om, rămâne loggată —
  util atât pentru raportare, cât și pentru îmbunătățirea sistemului în timp.
- **Relevanță mai largă**: triajul e primul pas dintr-un lanț mai lung (diagnosticare,
  remediere, escaladare) — un triaj mai bun reduce eroarea care s-ar propaga în restul
  procesului, chiar dacă restul lanțului rămâne manual aici.

## Cum se măsoară dacă funcționează

Două lucruri simplu de urmărit:

1. **Acuratețea clasificării** — cât de des coincide prioritatea/categoria propusă de
   agent cu eticheta reală, pe un set de test separat de datele folosite pentru context.
2. **Procentul de intervenție umană** — câte tichete ajung la aprobare, din total.

Ambele se compară cu un baseline manual (etichetarea deja existentă în datele istorice),
pe aceleași tichete și aceleași metrici — nu e nevoie de estimări sau demo la etapa asta,
doar de metoda clară de verificare.

## Ce nu face

Ca să fie clar unde se opresc lucrurile: agentul nu diagnostichează tehnic incidentul, nu
face remediere automată, nu orchestrează incidente majore și nu integrează cu un ITSM
real în producție — lucrează pe date mock, pe partea de triaj și rutare.

## Structura repo

```
README.md                          ← acest fișier
docs/
├── architecture.md                ← arhitectură tehnică completă (roluri, tool-uri, diagrame)
└── data-design.md                 ← schema de date și strategia de RAG
diagrams-drawio/
├── 1-arhitectura-componente.drawio
├── 2-flux-to-be.drawio
└── 3-state-machine.drawio
slides/
└── triaj-agent-proposal.pptx      ← prezentare
```
