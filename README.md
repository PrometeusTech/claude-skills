# claude-skills

Skill-uri personale pentru Claude Code, pentru proiecte Rails + React, cu fișiere pe BunnyNet sau
pe disc local, și pentru aplicațiile lor mobile în React Native / Expo. Repo-ul e un
**marketplace de plugin-uri** Claude Code cu două plugin-uri:

- `dev-skills` — backend și web; se instalează în orice proiect;
- `mobile-skills` — aplicația mobilă; se instalează, lângă `dev-skills`, doar în proiectele care
  au o aplicație mobilă (sau un backend folosit de una).

## Ce conține

### `dev-skills`

| Skill | Tip | Când se folosește |
|---|---|---|
| `file-uploads-cdn` | subiect | upload-uri, atașamente, descărcări, CDN / storage, validatoare de fișiere |
| `authz-multitenancy` | subiect | endpoint-uri, policies, roluri, serializere, guard-uri, date per tenant |
| `db-performance` | subiect | liste, paginare, N+1, indecși, căutare, migrații, liste mari în UI |
| `text-input-hardening` | subiect | câmpuri text (titluri, tag-uri, nume de fișier): caractere invizibile, lungimi, unicitate după colația DB, text ostil în UI |
| `idempotency-retries` | subiect | create cu `Idempotency-Key`, retry-uri, cereri concurente, indecși unici, timeout-uri pe clienți HTTP externi |
| `web-security` | subiect | XSS, params permise, redirect-uri, SSRF, CORS/CSRF, secrete și date personale în loguri / analytics, endpoint-uri publice, erori silențioase, ciclul de viață al datelor personale |
| `frontend-quality` | subiect | accesibilitate (axe, tastatură), traduceri complete, date proaspete după modificări, deep link / refresh / back, stări, CSP, cost per pagină |
| `deploy-safety` | subiect | migrații, expand / contract, compatibilitate API ↔ frontend vechi, variabile de mediu, job-uri și scheduler-e, dependențe, rollback |
| `domain-integrity` | subiect | state machines, sume și rotunjiri, date / ore / fus orar, rezervări și suprapuneri, contoare și invarianți, soft delete, audit trail |
| `notifications-integrations` | subiect | e-mail / push / WhatsApp: după commit, o singură dată, destinatarii corecți, traduceri, volum; webhook-uri primite (semnătură, replay, ordine); clienți de provideri |
| `pre-pr-check` | flux | verificare light pe diff la finalul unei schimbări, înainte de PR: gate-ul local, `/code-review`, `/security-review`, checklist-urile pe fișierele schimbate, probe ca teste; scrie secțiunea „Pre-PR check” |
| `implement-pr` | flux | implementarea unei schimbări ca un PR: context, teste întâi, verificare locală, raport |
| `feature-judge` | flux | review independent, cu dovezi, pe unul sau mai multe PR-uri, cu prompt de fix |

### `mobile-skills`

| Skill | Tip | Când se folosește |
|---|---|---|
| `mobile-release-compat` | subiect | build-uri vechi instalate luni de zile vs API-ul nou, versiune minimă și update forțat, OTA vs build nou în store (runtime version, schimbări native), rollout treptat, configurare per mediu, regulile App Store / Google Play (ștergerea contului, privacy manifest, data safety, permisiuni) |
| `mobile-offline-lifecycle` | subiect | timeout-uri și retry-uri sigure, scrieri întrerupte și cozi offline, cache salvat local (versiune, ștergere la logout), revenire din fundal, aplicația omorâtă de OS, task-uri în fundal, baze locale cu migrări |
| `mobile-security` | subiect | clientul nu e de încredere, token-uri în Keychain / Keystore, sesiune și refresh concurent, logout care șterge tot, deep link-uri ca input ostil, WebView, „tot ce e în bundle e public”, loguri / crash reports, ecrane sensibile în app switcher |
| `mobile-push-deeplinks` | subiect | permisiunea pentru notificări, token-uri per utilizator și dispozitiv, conținut pe lock screen, canale Android, tap cu aplicația închisă / în fundal / deschisă, universal links / App Links și fișierele de verificare |
| `mobile-ui-quality` | subiect | VoiceOver / TalkBack, text mărit din setări, ținte de atingere, safe areas, tastatura, back pe Android, ecrane mici și tablete, dark mode, stări, performanța listelor și a imaginilor |

Extensii în `dev-skills` pentru mobil: `file-uploads-cdn/references/react-native.md` (poze din
cameră, HEIC, compresie, EXIF GPS, upload întrerupt), o secțiune despre push în
`notifications-integrations`, build-urile mobile în `deploy-safety`, iar `implement-pr`,
`pre-pr-check` și `feature-judge` încarcă skill-urile mobile când diff-ul atinge aplicația. Ce cere
un simulator sau un telefon (screen reader, tastatură, push, link-uri, offline) apare ca „needs a
device” — nu se poate rula într-un container cloud și nu se raportează ca verificat.

Skill-urile de subiect au **reguli de implementare și listă de verificare** în același loc, deci
se folosesc atât la construire, cât și la review. Specificul de framework e în `references/`
(Rails, React, MySQL, BunnyNet, React Native / Expo), citit doar când e nevoie.

Specificul unui proiect anume (roluri, decizii, scripturi) **nu** stă aici: rămâne în
`AGENTS.md` / `CLAUDE.md` / `.claude/skills/` din repo-ul proiectului.

## Instalare

### Într-un proiect (recomandat)

Din rădăcina proiectului, o singură dată:

```bash
claude plugin marketplace add PrometeusTech/claude-skills --scope project
claude plugin install dev-skills@claude-skills --scope project
# doar dacă proiectul are (sau servește) o aplicație mobilă:
claude plugin install mobile-skills@claude-skills --scope project
```

Fă commit la `.claude/settings.json`. Orice sesiune **locală** (terminal, IDE) din proiect are apoi
skill-urile.

**Sesiunile cloud (claude.ai/code) nu instalează pluginurile declarate de un repo**, dar încarcă
`.claude/skills/` din repo. Pentru ele, copiază skill-urile în proiect cu un script care le ia din
acest repo la un commit fixat (`plugins/dev-skills/skills/*`, plus `plugins/mobile-skills/skills/*`
în proiectele cu aplicație mobilă → `.claude/skills/`) și notează
commit-ul într-un fișier lock; nu edita copiile, schimbă-le aici și resincronizează. Regulile
proprii proiectului stau lângă ele, într-un skill `<proiect>-checks`, pe care `pre-pr-check`,
`implement-pr` și `feature-judge` îl citesc primul.

### Doar pentru tine, în toate proiectele de pe mașină

```bash
claude plugin marketplace add PrometeusTech/claude-skills
claude plugin install dev-skills@claude-skills
claude plugin install mobile-skills@claude-skills   # opțional
```

### Update-uri

Pluginul nu are `version`, deci urmează ultimul commit. Update-ul automat e oprit implicit:
pornește-l din `/plugin` → Marketplaces → `claude-skills` → Enable auto-update, sau rulează
`/plugin marketplace update claude-skills`.

## Folosire

Skill-urile de subiect se încarcă singure când task-ul se potrivește cu descrierea lor. Cele de
flux se pot cere explicit:

- `/dev-skills:implement-pr` + ce ai de implementat
- `/dev-skills:pre-pr-check` — la finalul schimbărilor, înainte de PR (`implement-pr` îl rulează singur)
- `/dev-skills:feature-judge` + PR-urile sau intervalul de commit-uri de verificat

### Fluxul recomandat

| Nivel | Când | Ce | Cost |
|---|---|---|---|
| Gate local | la fiecare push (hook `pre-push`, opțional) | scriptul de CI local al proiectului | doar timp |
| `pre-pr-check` | o dată, la finalul implementării, înainte de PR | diff-only: gate, `/code-review`, `/security-review`, checklist-uri, probe ca teste; scrie „Pre-PR check” în PR | ~30–60k tokeni |
| `feature-judge` | o dată pe feature, **pe PR-ul deschis, înainte de merge** | aplicația pornită, roluri, concurență, volum, cod stricat intenționat; pornește de la „left for the judge” din Pre-PR check | ~300–400k tokeni |

`feature-judge` rulează ca **sub-agent separat** (`context: fork`) pe modelul `opus`, în prim-plan
(`background: false`, ca să aibă toate tool-urile). Sub-agentul nu vede conversația, deci scopul
se dă complet în argumente (repo-uri, PR-uri sau commit-uri, planul de referință). Raportul se
întoarce ca text în conversație.

## Cum le îmbunătățim

Când un judge sau un review găsește o categorie nouă de problemă, adaug-o în lista de verificare
a skill-ului de subiect potrivit (sau într-un skill nou), printr-un PR aici. Așa lecția ajunge în
toate proiectele.

### Ce am preluat din pluginurile oficiale Anthropic

Câteva idei din [`claude-plugins-official`](https://github.com/anthropics/claude-plugins-official)
sunt integrate direct în skill-uri, fără dependență de pluginuri:
- `code-review`: istoricul git și comentariile din PR-urile anterioare pe aceleași fișiere;
  respectarea comentariilor din cod;
- `pr-review-toolkit`: erorile silențioase (`silent-failure-hunter`), testele care nu prind
  regresii (`pr-test-analyzer`);
- `claude-security`: fiecare problemă e contestată pe accesibilitate, impact și apărări
  existente, iar problemele care existau înainte de PR sunt separate.

`claude-security` poate fi rulat și separat, ca scanare de securitate dedicată pe un diff; are
nevoie de tool-ul Workflow.

Context: Opus 5.5 și Sonnet 5.5 rulează nativ cu fereastră de 1M tokeni, inclusiv în sub-agentul
judge-ului, deci un singur agent poate citi tot diff-ul unui feature. Consumul real al unui judge
e de ordinul 250–400k tokeni.

Rezultatele evaluărilor (cu skill vs fără skill) sunt în `evals/`, fără detalii din proiectele
pe care s-au rulat.

Verificare locală înainte de push:

```bash
claude plugin validate .
```
