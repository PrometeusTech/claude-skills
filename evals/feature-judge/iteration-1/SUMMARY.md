# feature-judge — iteration 1

**Data:** 2026-09-30 / 10-01 · **Model:** Opus 5.5, effort high · **Skill-uri:** versiunea din
commit-ul `986db60`.

## Ce s-a testat

Un feature real, deja merged, dintr-un proiect Rails + React: o bibliotecă de documente per tenant
(upload PDF / Word de către admin, listă cu căutare și tag-uri, descărcare prin URL CDN semnat),
două PR-uri (backend + frontend). Referința („ground truth”) a fost un judge manual anterior, cu
prompt scris de mână: 8 probleme confirmate și 13 observații.

Două rulări independente, aceeași cerere:
- **cu skill:** `feature-judge` + skill-urile de subiect;
- **fără skill:** aceeași cerere, fără skill-uri.

Detaliile proiectului (căi, PR-uri, cod) nu sunt incluse aici; o parte din probleme erau încă în
curs de reparare.

## Consum

| | Tokeni | Timp | Tool calls | Findings |
|---|---:|---:|---:|---|
| Cu skill | 266.189 | 27,2 min | 96 | 2 Mediu, 4 Mic, 2 Notă |
| Fără skill | 228.452 | 18,8 min | 81 | 2 Mediu, 6 Mic, 3 Info |

## Calitate

| Criteriu | Cu skill | Fără skill |
|---|---|---|
| Fiecare finding are dovadă | ✅ | ✅ |
| Suspiciunile neconfirmate sunt separate | ✅ | ❌ |
| Prompt de fix la final | ✅ | ❌ |
| Performanța listei măsurată pe volum (query count / EXPLAIN) | ✅ | ❌ |
| Regresii pe celelalte fluxuri care folosesc pipeline-ul comun de upload | parțial | — |
| Recall pe problemele confirmate ale judge-ului manual | 4/8 | 3/8 |
| Recall pe observații | 3,5/11 | 3/11 |
| Probleme noi, reale, negăsite de judge-ul manual | 3 | 5 |

Probleme noi găsite **cu skill**: cheie de idempotență mai lungă decât coloana (efectul are loc,
cheia nu se salvează, apoi răspunde cu o eroare generică); două cereri concurente cu aceeași cheie
dau eroare de unicitate (cheia se salvează după efect); poliglot PDF/HTML acceptat ca PDF.

Probleme noi găsite **fără skill**: XLS care conține textul `WordDocument` acceptat ca DOC;
descărcarea unui document șters spune „nu ai dreptul”; un buton flotant acoperă ultimul
„Descarcă”; extensie dublată la descărcare; fișierul de 10 MB copiat de 3 ori în memorie.

## De ce au scăpat probleme

| Cauză | Exemple (din referință) | Ce s-a schimbat |
|---|---|---|
| Lipsă din checklist | NUL în numele fișierului (500 + obiect orfan în CDN); deduplicare de tag-uri fără diacritice, deși colația DB le ignoră; chip-uri lungi/multe pe mobil; replay de idempotență după ștergere; cuplaj cu store-ul altui domeniu; anti-pattern React | skill nou `text-input-hardening`, skill nou `idempotency-retries`; mentenabilitate și UI ostil în verificările mereu active din `feature-judge` |
| Sărite de agent | memorie per upload; mesaj greșit la nume de fișier lung; diferența de bundle | matrice de acoperire obligatorie în raport: fiecare punct e verificat, finding sau sărit cu motiv |
| Verificate greșit | numele de la descărcare trecut „OK” după câmpul `filename` din API, nu după numele salvat de browser prin CDN; coduri de eroare documentate pe o operație care nu le poate întoarce | regula „verifică efectul, nu câmpul”; coduri de eroare verificate per operație, în ambele sensuri |

Sniffer-ul OOXML/OLE (verificat după numele intrărilor, nu după content types / directorul OLE)
a fost prins doar parțial; `file-uploads-cdn` cere acum explicit content type-ul părții principale
și parsarea directorului OLE, cu probele care le ocolesc.

## Decizii

- **`context: fork` + `agent: general-purpose`:** da. Judge-ul nu are nevoie de conversație, doar
  de scop, iar contextul principal rămâne curat.
- **`background: false`:** un sub-agent în fundal primește un set mai mic de tool-uri; în test,
  scrierea raportului într-un fișier a fost refuzată. Rulat în prim-plan, are toate tool-urile.
  Raportul se întoarce oricum ca text.
- **`model: opus`:** păstrat; diferențele din test au venit din checklist, nu din model (ambele
  rulări au folosit același model). Nu e măsurat față de alt model.
- **Baseline CI pe base:** doar când un pas pică pe head (economie de timp față de iterația 1).

## Următoarea iterație

Același feature, după merge-ul fix-urilor: rulare cu skill-urile noi, aceleași criterii plus
„matricea de acoperire acoperă toate punctele skill-urilor încărcate”. Ținta: ≥ 7/8 pe problemele
confirmate, fără creșterea consumului peste ~300k tokeni.
