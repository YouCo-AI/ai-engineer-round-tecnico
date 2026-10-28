# Da agente testuale ad assistente vocale real-time con avatar

> **System Design Document** · Proposta architetturale per trasformare un agente ReAct con tool calling in un assistente vocale real-time con avatar.

**In una frase**: una **pipeline cascade in streaming su WebRTC** (VAD → STT → LLM → TTS) con un **orchestratore ReAct proprietario**, che offre controllo deterministico su tool, chain of thought, turni e interruzioni entro un budget voice-to-voice di **~700 ms (p50)**. Intorno alla pipeline ci sono un ramo emotivo parallelo e un avatar sincronizzato sul clock audio del client.

### Indice
0. [Overview architetturale e decisione (ADR-001)](#0-overview-architetturale-e-decisione-adr-001)
1. [La voce](#1-la-voce)
2. [I turni di parola](#2-i-turni-di-parola)
3. [Solo la risposta](#3-solo-la-risposta)
4. [Le interruzioni](#4-le-interruzioni)
5. [Le emozioni](#5-le-emozioni)
6. [L'avatar](#6-lavatar)
7. [Labiale ed espressioni](#7-labiale-ed-espressioni)
- [Sintesi delle decisioni](#sintesi-delle-decisioni)
- [Traccia di lavoro](#traccia-di-lavoro)

---

## 0. Overview architetturale e decisione (ADR-001)

### Contesto
L'agente di partenza segue un ciclo ReAct multi-step: il modello alterna ragionamento, chiamate a tool e risposta finale. Per passare alla voce le strade sono due:

| | **Cascade in streaming** (VAD → STT → LLM → TTS) | **Speech-to-speech nativo** (es. OpenAI Realtime) |
|---|---|---|
| Controllo sul ciclo ReAct | **Completo**: l'orchestratore gestisce ogni passo, i timeout, i retry e i limiti sul numero di step | Parziale: il ciclo vive dentro la sessione del provider |
| Filtraggio di ragionamenti e tool call | **Deterministico**, sul testo e prima del TTS | Il modello pensa e parla nello stesso flusso, quindi il filtraggio è limitato |
| Scelta dei modelli | Ogni stadio si sostituisce da solo (STT per lingua, LLM per qualità sui tool, TTS per voce e costo) | Un solo fornitore per tutto |
| Osservabilità | Trascrizioni, prompt e output sono testo: si possono loggare, valutare e rigiocare | Più opaca |
| Latenza | Più stadi: serve uno streaming end-to-end aggressivo | Bassa per costruzione |
| Informazione paralinguistica | Lo STT perde tono ed emozione (si recupera con un ramo audio parallelo, vedi punto 5) | Conservata |

### Decisione
**Pipeline cascade in streaming, trasportata su WebRTC** (es. LiveKit o Pipecat come framework agente, con un SFU/media server). Il ciclo ReAct resta **invariato nel suo nucleo**: si aggiungono degli *adattatori di confine* in ingresso e in uscita e un **orchestratore che conosce lo stato del parlato**.

### Conseguenze
- La latenza diventa un **requisito di progetto** con un budget esplicito per ogni stadio (§1.1), non un risultato da sperare.
- Serve un livello di **post-processing dell'output** (parser, normalizzatore, chunker) tra LLM e TTS: è proprio dove mettiamo il controllo deterministico.
- **Quando riconsiderare la scelta**: con un agente che usa pochi tool e semplici e punta su empatia e prosodia, oppure se i modelli STS esporranno un controllo fine sul ciclo dei tool. È possibile anche un ibrido: STS per il chit-chat e cascade per i task.

### Architettura di riferimento

```
┌────────────────────────────── CLIENT (browser / app) ──────────────────────────────┐
│                                                                                    │
│  mic ─▶ AEC/NS/AGC (WebRTC APM) ─▶ Opus ──────────────────────────┐                │
│                                                                   │                │
│  speaker ◀── AudioContext (master clock) ◀── audio agente ◀───────┼─────┐          │
│                    │ currentTime                                  │     │          │
│                    ▼                                              │     │          │
│  Avatar runtime: L1 idle · L2 stato · L3 expr + visemi (§6-7) ◀───┼─────┤ eventi   │
│                                                                   │     │          │
└───────────────────────────────────────────────────────────────────┼─────┼──────────┘
                                   WebRTC: audio RTP + data channel │     │
┌──────────────────────────────── AGENT WORKER ─────────────────────▼─────┴──────────┐
│                                                                                    │
│  [VAD] ─▶ [STT streaming] ─▶ [Turn Manager: VAD + EOU semantico + barge-in]        │
│    │                                      │ turno confermato / barge_in  (§2, §4)  │
│    └─▶ [Ramo emotivo] ─▶ Affect State     ▼                                        │
│        (§5, parallelo)     │     ┌──────────────────────────┐                      │
│                            └───▶ │   Orchestratore ReAct    │ ◀──▶ Tool registry   │
│                                  │ stato conv. + parlato,   │      + executor      │
│                                  │ TurnScope (cancel tree)  │      (§1.3, §4.3)    │
│                                  └────────────┬─────────────┘                      │
│                                               │ stream di eventi LLM               │
│                                               ▼                                    │
│  [Router a blocchi] ─▶ [Parser XML a stati] ─▶ [Chunker] ─▶ [Normalizer] ─▶ [TTS]  │
│   thinking│tool_use     say│answer│expr (§3)              offset map (§4.2)   │    │
│                                                                               │    │
│  [Playout ledger + pacer] ◀── audio + timestamp parole/visemi ────────────────┘    │
│  [Avatar event builder]   ──────────────────────▶ audio + eventi verso il client   │
│                                                                                    │
└────────────────────────────────────────────────────────────────────────────────────┘
   Opzione §6-B: [Neural renderer su GPU] ─▶ traccia video WebRTC al posto dell'avatar client
```

**Il percorso di un turno** (riferimenti alle sezioni):
1. L'audio dell'utente passa per AEC e noise suppression sul client e arriva al worker via WebRTC (§1.4).
2. VAD e STT lavorano in streaming. Il **Turn Manager** combina il VAD con l'endpoint semantico e conferma il turno (§2), mentre il ramo emotivo aggiorna l'Affect State in parallelo (§5).
3. L'**orchestratore** esegue il ciclo ReAct: chiamate LLM, tool con politiche per classe, filler durante l'attesa (§1.3).
4. L'output dell'LLM passa per il router a blocchi e il parser a stati: solo `<say>` e `<answer>` arrivano al TTS (§3). I tag `<expr/>` diventano eventi per l'avatar (§7.2).
5. Chunker, normalizzatore e TTS producono audio con timestamp. Il playout ledger registra cosa viene effettivamente riprodotto (§4.2).
6. Il client riproduce l'audio sull'`AudioContext` e anima l'avatar sullo stesso clock (§7.1).
7. Se l'utente interrompe, il `barge_in` cancella il `TurnScope`: si fermano audio, LLM e TTS, e la storia viene troncata a ciò che l'utente ha sentito (§4).

---

## 1. La voce

> **Domanda**: STT in ingresso e TTS in uscita bastano? **No.** Mancano quattro cose: un budget di latenza end-to-end, un registro adatto al parlato, la gestione dei silenzi mentre girano i tool e la soppressione dell'eco.

### 1.1 Latenza cumulativa

**Metrica primaria: voice-to-voice latency (V2V)**, cioè il tempo che passa tra la fine reale del parlato dell'utente e il primo campione audio dell'agente riprodotto sul client. Si misura lato client, sui percentili.

**SLO proposti**

| Metrica | Target p50 | Target p95 |
|---|---|---|
| V2V su turno senza tool | ≤ 700 ms | ≤ 1200 ms |
| Time-to-first-audio su turno con tool (preambolo o filler) | ≤ 800 ms | ≤ 1300 ms |
| Silenzio massimo senza feedback durante l'esecuzione di un tool | — | ≤ 2 s |

**Budget per stadio** (valori indicativi, servizi nella stessa region, connessioni calde):

| Stadio | Budget | Leva di ottimizzazione |
|---|---|---|
| Uplink WebRTC (frame Opus + rete) | 30–50 ms | Media server vicino all'utente, UDP |
| Hangover del VAD (silenzio minimo) | 200–300 ms | Abbassabile perché la decisione finale spetta al detector semantico (§2) |
| Inferenza turn detector | 10–50 ms | Modello piccolo su CPU, gira sui partial |
| Finalizzazione STT | 50–150 ms | **In parallelo** all'hangover (STT streaming, non batch) |
| LLM time-to-first-token | 200–350 ms | Prompt caching, modello "fast", contesto compatto, **generazione preventiva** |
| Prima clausola completa | 50–100 ms | Il chunker invia al TTS la prima clausola, non la frase intera |
| TTS time-to-first-byte | 80–150 ms | TTS streaming con input incrementale via websocket |
| Downlink + jitter buffer | 40–80 ms | Jitter buffer adattivo |
| **Somma seriale** | **~700–1200 ms** | |
| **Con le sovrapposizioni** | **~600–750 ms** | STT ∥ hangover, LLM preventivo ∥ conferma EOU |

**Principi di progetto per rispettare il budget**
1. **Streaming end to end**: nessuno stadio aspetta l'output completo di quello precedente. Lo STT emette i partial, l'LLM produce token, il chunker emette clausole, il TTS produce audio a blocchi.
2. **Generazione preventiva**: appena arriva l'evento "fine parlato candidata", l'orchestratore fa partire l'LLM sulla trascrizione corrente. Se l'utente riprende a parlare, la generazione si **annulla** (cancellazione della richiesta e svuotamento delle code). Vincolo deterministico: **in modalità preventiva non si eseguono tool con side-effect**. L'esecuzione si sblocca solo a turno confermato.
3. **Prima frase corta**: il prompt chiede di aprire con una frase breve, così la prima clausola arriva in pochi token.
4. **Co-locazione e connessioni persistenti** verso STT, LLM e TTS (niente handshake per ogni turno).
5. **Osservabilità**: ogni stadio emette timestamp sullo stesso `turn_id` (tracing distribuito), per sapere *dove* si consuma il budget.

### 1.2 Registro scritto vs parlato

Un agente nato per la chat produce markdown, elenchi, tabelle, URL, ID e numeri formattati. Letti da un TTS diventano rumore ("asterisco asterisco", "trattino"), oppure il TTS li pronuncia male.

Si interviene su **due livelli complementari**: il prompt riduce il problema alla fonte, la normalizzazione è la rete di sicurezza deterministica.

**A. Voice-first prompting** (sezione dedicata del system prompt)
- Niente markdown, elenchi, tabelle o emoji: solo prosa parlata.
- Risposte brevi, 1–3 frasi, poi offrire l'approfondimento ("Vuoi che ti dica anche gli orari del pomeriggio?").
- Al massimo 2–3 opzioni per volta, perché a voce l'utente non può "rileggere".
- Mai leggere URL, ID tecnici, JSON o output grezzi dei tool: vanno riassunti.
- **Consapevolezza dell'ingresso**: "Il messaggio dell'utente è una trascrizione automatica e può contenere errori, soprattutto su nomi propri e numeri. Se un dato è critico, confermalo prima di agire". Questa regola è collegata al gating dei tool con side-effect.

**B. Text normalization** (deterministica, tra parser e TTS)

| Categoria | Input | Output parlato |
|---|---|---|
| Valuta (locale IT) | `€ 1.250,50` | "milleduecentocinquanta euro e cinquanta" |
| Date e orari | `28/10/2026 14:30` | "ventotto ottobre, alle quattordici e trenta" |
| Unità | `5 km`, `20°C` | "cinque chilometri", "venti gradi" |
| Telefoni e codici | `3471234567` | Raggruppamento a cifre con pause ("tre quattro sette…") |
| Sigle | `INPS`, `API` | Lessico di pronuncia: sigla letta come parola o compitata |
| Markdown residuo | `**importante**`, `- voce` | Rimosso (strip) |
| URL ed email | `www.esempio.it/x?id=…` | Sostituiti con "ti ho mandato il link in chat" (canale testuale parallelo) |

- Si implementa con regole e grammatiche per locale (approccio WFST/regex) più un **dizionario di pronuncia** per brand e nomi di dominio (SSML `<phoneme>` o lessico del TTS, dove supportato).
- Va eseguita **sul singolo chunk** prima del TTS, quindi il chunker non deve spezzare entità come `3,5`, `Sig.` o `14:30`.
- **Simmetria in ingresso**: keyword boosting o lessico custom sullo STT per entità di dominio (nomi prodotto, città, codici).

### 1.3 Silenzi durante l'esecuzione dei tool

In chat, un tool da 3 secondi si vede come uno spinner. A voce, 3 secondi di silenzio sembrano un guasto. L'orchestratore applica una **strategia a livelli**, guidata dai metadati del tool registry:

```yaml
tool: search_availability
latency_class: slow        # fast (<500ms) | slow (0.5–5s) | async (>5s)
side_effect: false
filler_hint: "Controllo le disponibilità"
timeout_ms: 6000
```

| Livello | Quando | Meccanismo |
|---|---|---|
| **L0 – Niente** | Tool `fast` | Nessun riempitivo, perché parlare inutilmente peggiora l'esperienza |
| **L1 – Preambolo del modello** | Il modello lo emette prima della tool call | Tag `<say>Controllo subito il calendario.</say>`, l'unico testo intermedio ammesso al TTS (§3). Parte mentre l'LLM sta ancora generando gli argomenti del tool |
| **L2 – Filler deterministico** | Non c'è stato un preambolo e il tool non ha risposto entro `T_filler` (~700 ms) | Il nome del tool è noto **già all'inizio del blocco `tool_use`**, prima degli argomenti, quindi l'orchestratore sceglie subito un filler contestuale. Audio **pre-sintetizzato e in cache** (latenza ~0), ruotato per non ripetersi |
| **L3 – Aggiornamenti di avanzamento** | Tool `slow` o `async` oltre ~4 s | Messaggio periodico ("Ci sono quasi, sto ancora verificando…"), al massimo uno ogni N secondi. Con tool `async` l'utente può continuare a parlare |
| **L4 – Fallback** | Timeout o errore | Messaggio parlato onesto e opzione di retry, mai silenzio |

Regole trasversali:
- **Coda del parlato seriale**: la risposta finale si accoda al preambolo in riproduzione e non lo tronca.
- **Rate limit dei preamboli** nei cicli multi-step (al massimo uno per step, deduplicati), per evitare "Controllo… Verifico… Sto guardando…".
- **Tool indipendenti in parallelo**, così si riduce il tempo di attesa invece di limitarsi a coprirlo.
- Lo stato "sto lavorando" viene esposto anche come evento per l'avatar (espressione pensosa, vedi punto 6) e per eventuali earcon discreti.

### 1.4 Eco

**Problema**: il microfono capta l'audio dell'agente dagli altoparlanti. Lo STT trascrive l'agente stesso e il VAD scambia l'eco per parlato dell'utente. Ne nascono **auto-interruzioni** (falsi barge-in) e turni fantasma.

Difesa in profondità, dal segnale alla semantica:

| # | Livello | Misura |
|---|---|---|
| 1 | **Client, AEC** | `getUserMedia({echoCancellation, noiseSuppression, autoGainControl})`: WebRTC APM (AEC3) usa l'audio riprodotto come riferimento. È la prima linea e copre la maggior parte dei casi su browser e app |
| 2 | **Server, AEC con riferimento** | Per canali senza AEC affidabile (telefonia SIP, dispositivi embedded): il server conosce **esattamente** l'audio TTS inviato e lo usa come segnale di riferimento per un AEC lato server |
| 3 | **Isolamento della voce** | Modelli di noise e voice isolation (tipo Krisp/BVC) prima di VAD e STT: abbattono eco residua, TV e voci di sottofondo |
| 4 | **Gating adattivo durante la riproduzione** | Mentre l'agente parla: soglia VAD più alta e **durata minima del parlato** più lunga (es. ≥ 300 ms) per considerarlo barge-in. Il microfono **non si muta**, altrimenti l'utente non può interrompere (punto 4) |
| 5 | **Filtro semantico anti-eco** | Se la trascrizione arrivata durante la riproduzione è molto simile (fuzzy match) al testo appena inviato al TTS, viene scartata come eco |

Metriche: tasso di falsi barge-in per sessione e tasso di trascrizioni scartate dal filtro anti-eco (se alto, l'AEC a monte non sta funzionando).

---

## 2. I turni di parola

> **Obiettivo**: niente push-to-talk. L'agente deve capire *quando* l'utente ha finito, senza tagliarlo mentre pensa e senza attese innaturali.

### 2.1 Il limite della soglia di silenzio fissa

Un endpointing basato solo sul silenzio impone un trade-off che non si può risolvere:

| Soglia | Effetto |
|---|---|
| Bassa (~300 ms) | Reattivo, ma interrompe chi fa pause: *"Vorrei prenotare per… ehm… giovedì"* viene tagliato a "per" |
| Alta (~1.2 s) | Non interrompe, ma ogni turno paga più di un secondo di latenza: fuori SLO |

Il silenzio è un segnale **acustico**. Il fatto che una frase sia completa è un'informazione **linguistica**. Servono tutti e due.

### 2.2 Soluzione: endpointing ibrido a due stadi

```
audio ─▶ [VAD acustico]──"silenzio ≥ 200-300ms"──▶ CANDIDATO EOU
                                                      │
              trascrizione corrente + ultimi turni ───▶ [Turn detector semantico]
                                                      │  P(fine turno)
                                                      ▼
                                       attesa dinamica = f(P, stato dialogo)
                                                      │
                          ripresa parlato? ── sì ──▶ annulla e torna in ascolto
                                                      │ no, scaduta l'attesa
                                                      ▼
                                                TURNO CONFERMATO
```

**Stadio 1, VAD acustico** (es. Silero VAD): frame da ~30 ms, soglia di probabilità, durata minima del parlato (filtra click e rumori) e silenzio minimo **corto**, perché qui il VAD *propone* e non decide.

**Stadio 2, smart endpoint semantico** (es. LiveKit turn detector, modello linguistico piccolo, ~100–500M parametri, inferenza CPU in decine di ms): riceve la trascrizione parziale con il contesto degli ultimi turni, **inclusa l'ultima domanda dell'agente**, e stima P(fine turno).
- *"Vorrei prenotare per"* → P bassa (frase sintatticamente aperta)
- *"Vorrei prenotare per giovedì alle otto"* → P alta
- Dopo *"Mi confermi?"* dell'agente → *"Sì"* → P alta, anche se è una sola parola

**Politica di attesa dinamica**

| P(fine turno) | Attesa aggiuntiva dopo il candidato |
|---|---|
| ≥ 0.85 | 0–150 ms → commit quasi immediato |
| 0.5 – 0.85 | ~500 ms |
| < 0.5 | Fino a `max_endpoint_delay` (es. 2–3 s), poi **commit forzato** come rete di sicurezza |

I valori sono da tarare su dati reali. Se il parlato riprende durante l'attesa, il candidato viene annullato.

### 2.3 Raffinamenti per la produzione

- **Segnale prosodico**: un detector solo testuale non sente l'intonazione (una frase interrogativa e una affermativa hanno lo stesso testo). Come complemento si può usare un modello che lavora **direttamente sull'audio** (es. Pipecat Smart Turn) e fondere i due punteggi.
- **Hint di stato dall'orchestratore**: se l'agente ha appena chiesto un numero di telefono, un codice o un indirizzo, l'utente detterà a blocchi con pause. L'orchestratore imposta `expected_input = dictation`, che allunga l'attesa e alza la soglia di P. È un controllo deterministico che un modello STS non offre.
- **Integrazione con la generazione preventiva** (§1.1): sul candidato EOU l'LLM parte subito. Se il detector conferma, l'output è già pronto. Se l'utente riprende, si annulla. I tool con side-effect sono sempre bloccati fino al commit.
- **Lingua**: va verificata la copertura dell'italiano nel detector scelto (i modelli multilingue la includono, ma la qualità varia), con valutazione sul dominio specifico.
- **Alternative e fallback**: endpointing semantico offerto dal vendor STT (alcuni lo integrano). Si può usare come segnale aggiuntivo, ma la decisione finale resta nel Turn Manager: un solo punto di verità.
- **Confine col punto 4**: i backchannel durante il parlato dell'agente ("mhm", "ok", "sì sì") **non sono turni** e vanno classificati a parte. Il Turn Manager distingue `user_turn` da `backchannel` da `barge_in`.

### 2.4 Macchina a stati del Turn Manager

```
LISTENING ──vad_start──▶ USER_SPEAKING ──vad_stop──▶ PENDING_EOU
    ▲                          ▲                          │
    │                          └────────vad_start─────────┤ (annulla, cancel LLM preventivo)
    │                                                     │ timeout(P) scaduto
    │                                                     ▼
    │                                               TURN_COMMITTED ──▶ AGENT_THINKING
    │                                                                        │ primo chunk audio
    └─────────────────────────── fine riproduzione ◀──────────────── AGENT_SPEAKING
```

### 2.5 Metriche
- **Tasso di interruzioni premature** (turno confermato mentre l'utente non aveva finito): si misura su un dataset etichettato di conversazioni italiane e in produzione tramite proxy (l'utente riprende a parlare entro 1 s dal commit).
- **Endpoint latency** p50 e p95 (fine reale del parlato → commit).
- Le due metriche sono in tensione: si tarano insieme guardando una curva, non con un numero solo.

---

## 3. Solo la risposta

> **Obiettivo**: l'utente sente **solo** ciò che è destinato a lui (preamboli di cortesia e risposta finale), mai ragionamenti, tool call, argomenti JSON o output grezzi dei tool.

**Principio guida: default deny.** Il canale verso il TTS è una whitelist: viene detto solo il contenuto *esplicitamente* marcato come parlabile. Se qualcosa va storto, l'agente resta zitto invece di leggere il chain of thought. Il caso di silenzio è coperto da un fallback (§3.4).

### 3.1 Livello 1: separazione nativa delle API

I provider moderni restituiscono già blocchi tipizzati nello stream:

| Tipo di blocco | Esempi | Destinazione |
|---|---|---|
| Ragionamento | Blocchi `thinking` (Anthropic), item `reasoning` (OpenAI) | **Solo log, telemetria, UI di debug**. Mai TTS |
| Tool call | `tool_use`, `tool_calls` delta | **Esecutore dei tool**. L'evento di inizio blocco porta il nome del tool e fa scattare il filler (§1.3) |
| Testo | `text` delta | → Livello 2 (parser) |

Il routing avviene **a livello di blocco**, quindi è il filtro più robusto e meno costoso: un blocco di ragionamento non arriva mai al parser.

**Perché non basta**: in un ciclo ReAct il modello scrive spesso **testo di commento nel canale `text`** prima di una tool call ("Ora interrogo il database clienti filtrando per…"). Per l'API è testo legittimo e non si distingue dalla risposta finale. In più, il testo arriva *prima* del blocco `tool_use`: non si può aspettare la fine dello step per capire se era commento, senza perdere lo streaming. Serve quindi un protocollo esplicito all'interno del testo.

### 3.2 Livello 2: protocollo a tag XML e stream parser a stati

**Contratto di output** (nel system prompt, con esempi few-shot):

| Tag | Significato | TTS |
|---|---|---|
| `<say>…</say>` | Preambolo breve per l'utente prima di una tool call | ✅ (soggetto a rate limit) |
| `<answer>…</answer>` | Risposta finale parlata | ✅ |
| `<thinking>…</thinking>` | Ragionamento esplicito (per modelli senza reasoning nativo) | ❌ log |
| `<tool_call>…</tool_call>` | Tool call testuale (modelli open-weight che non hanno tool calling nativo) | ❌ → esecutore |
| Testo fuori dai tag | Violazione del protocollo | ❌ (default deny) |

**Il problema dello streaming**: i token arrivano in chunk arbitrari, quindi un tag può essere spezzato (`"<ans"` + `"wer>"`). Il testo può anche contenere un `<` legittimo (`"meno di 5 < 10"`). Un `replace` o una regex sul chunk non bastano: serve una **macchina a stati con buffer di lookahead**.

**Stati**: `OUTSIDE`, `IN_SAY`, `IN_ANSWER`, `IN_THINKING`, `IN_TOOL_CALL`, più un sotto-stato trasversale `TAG_PENDING` (buffer aperto su `<`).

**Transizioni**

| Stato | Input | Azione | Nuovo stato |
|---|---|---|---|
| qualsiasi | carattere `<` | Apre il buffer di lookahead | `TAG_PENDING` |
| `TAG_PENDING` | il buffer è prefisso di un tag noto | Continua ad accumulare | `TAG_PENDING` |
| `TAG_PENDING` | il buffer coincide con un tag di apertura noto | Svuota il buffer | Stato del tag |
| `TAG_PENDING` | il buffer coincide con la chiusura del tag corrente | Flush del chunker, emette `segment_end` | `OUTSIDE` |
| `TAG_PENDING` | il buffer non può più essere un tag noto | Riversa il buffer come contenuto dello stato precedente | stato precedente |
| `IN_SAY` / `IN_ANSWER` | testo | → sentence chunker → normalizer → TTS | invariato |
| `IN_THINKING` | testo | → log | invariato |
| `IN_TOOL_CALL` | testo | → buffer degli argomenti, parse JSON alla chiusura | invariato |
| `OUTSIDE` | testo | → log `protocol_violation` | invariato |
| fine stream | stato aperto | Chiusura implicita, flush e metrica | — |

**Pseudocodice essenziale**

```python
KNOWN = {"<say>", "</say>", "<answer>", "</answer>", "<thinking>", "</thinking>",
         "<tool_call>", "</tool_call>"}
MAX_TAG = max(map(len, KNOWN))

def feed(chunk):
    for ch in chunk:
        if pending is not None:
            pending += ch
            if pending in KNOWN:            transition(pending); pending = None
            elif not is_prefix_of_known(pending) or len(pending) > MAX_TAG:
                                            route(state, pending); pending = None
        elif ch == "<":                     pending = "<"
        else:                               route(state, ch)

def route(state, text):
    if state in (IN_SAY, IN_ANSWER):        chunker.push(text)   # → normalizer → TTS
    elif state == IN_THINKING:              trace.log(text)
    elif state == IN_TOOL_CALL:             tool_buf.append(text)
    else:                                   metrics.protocol_violation(text)
```

Proprietà:
- **Latenza aggiunta trascurabile**: il buffer trattiene al massimo `MAX_TAG` caratteri e solo dopo un `<`.
- **Tag sconosciuti o malformati** sono trattati come testo dello stato corrente: nessun crash e nessuna fuga dalla whitelist.
- **Componente puro e testabile**: unit test sulle frammentazioni (lo stesso output spezzato in ogni posizione possibile deve dare lo stesso risultato) e fuzzing.

### 3.3 Composizione dei due livelli

```
stream LLM ─▶ Router a blocchi ─┬─ thinking  ──────────────────────▶ trace
                                ├─ tool_use  ──────────────────────▶ executor (+ trigger filler)
                                └─ text ─▶ Parser XML ─┬─ say/answer ─▶ chunker ─▶ normalizer ─▶ TTS
                                                       ├─ thinking   ─▶ trace
                                                       ├─ tool_call  ─▶ executor
                                                       └─ outside    ─▶ violation log
```

- Con un modello dotato di reasoning e tool calling nativi, il livello 1 fa quasi tutto il lavoro. Il parser serve a distinguere `say` da `answer` e a bloccare i commenti non marcati.
- Con modelli open-weight o senza reasoning nativo, il parser copre da solo tutti i canali.
- **Lo stesso codice serve entrambi i casi**: si può cambiare modello senza toccare la pipeline vocale.

### 3.4 Casi limite e politiche

| Caso | Politica |
|---|---|
| Ultimo step (nessuna tool call) senza alcun contenuto taggato | **Fallback**: si prende il testo fuori dai tag, si applicano strip del markdown e normalizzazione e lo si pronuncia. Si registra la violazione e si segue il tasso nelle eval |
| `<answer>` emesso in uno step che poi contiene una tool call | L'orchestratore lo tratta come `<say>`. Se è lungo, lo tronca alla prima frase |
| Preamboli ripetuti nei cicli multi-step | Rate limit e deduplica (§1.3) |
| Risposta che contiene dati grezzi dei tool (JSON, ID) | Prompt (§1.2.A) più normalizer: URL e ID vanno sul canale testuale parallelo, non nel parlato |
| Storia della conversazione | Nel contesto LLM si conserva l'output completo con i tag, e si registra a parte **cosa è stato effettivamente pronunciato**. Serve per le interruzioni (punto 4): il contesto deve riflettere ciò che l'utente ha davvero sentito |

### 3.5 Metriche
- **Leak rate**: percentuale di turni in cui contenuto non destinato all'utente arriva al TTS. Target 0, verificato con eval offline su conversazioni sintetiche multi-tool.
- **Protocol violation rate** per modello e versione del prompt: è il segnale di regressione quando si cambia LLM.
- **Fallback rate** (§3.4).

---

## 4. Le interruzioni

> **Obiettivo**: l'utente prende la parola quando vuole e l'agente si comporta come una persona che ascolta. Si ferma subito se è interrotto davvero, ignora tosse e "mhm", e dopo sa esattamente cosa l'utente ha sentito e cosa è stato fatto nel mondo reale.

**Principio guida: l'interruzione è un evento di cancellazione che attraversa tutta la pipeline.** Ogni turno dell'agente vive dentro un **albero di cancellazione** (structured concurrency). Lo scope radice è il turno, e i figli sono lo stream LLM, lo stream TTS, la riproduzione audio e le esecuzioni dei tool. Un solo evento, `barge_in`, emesso dal Turn Manager (l'unica fonte di verità, §2.3), cancella lo scope. Ogni figlio reagisce secondo la propria politica.

```
TurnScope (turn_id=42) ── cancel(reason=barge_in, t=…)
 ├─ LLMStream        → abort della richiesta HTTP (smette anche di consumare token)
 ├─ TTSStream        → cancel del context sul websocket TTS
 ├─ Playout          → flush delle code audio + segnale al client
 └─ ToolExecutions[] → politica per classe di tool (§4.3)
```

### 4.1 Falsi positivi: tosse, rumori e backchannel

Durante `AGENT_SPEAKING` o `AGENT_THINKING`, non tutto il parlato dell'utente è un'interruzione:

| Classe | Esempi | Comportamento atteso |
|---|---|---|
| **Rumore non verbale** | Tosse, starnuto, risata breve, porta, tastiera | Ignorare |
| **Backchannel** | "mhm", "sì", "ok", "certo", "esatto", "ah, capito" | L'agente **continua** a parlare (non è un turno) |
| **Parlato laterale** | L'utente risponde a qualcun altro nella stanza | Ignorare (caso difficile, solo best effort) |
| **Barge-in vero** | "Aspetta", "no, intendevo venerdì", "stop", una nuova domanda | Fermarsi e cedere il turno |

**Il trade-off**: classificare con sicurezza richiede qualche centinaio di millisecondi di audio e almeno una o due parole trascritte. Se nel frattempo l'agente continua a parlare a pieno volume, lui e l'utente si sovrappongono. Se invece si ferma al primo suono, ogni colpo di tosse lo blocca.

**Soluzione: arresto in due fasi, soft pause e poi hard stop**

```
voce rilevata durante il playout
        │
        ▼
 [FASE 1 · SOFT PAUSE]  ~100–200 ms dall'inizio del parlato
   playout in pausa (o ducking −15 dB), buffer CONSERVATO
   LLM e TTS continuano a generare in coda (nessuna perdita)
        │
        ▼  classificazione a livelli (finestra max ~600 ms)
 ┌──────┴───────────────┬───────────────────────┐
 rumore / backchannel   barge-in                indeciso allo scadere
 │                      │                       │
 RESUME dal punto       [FASE 2 · HARD STOP]    → trattato come barge-in
 esatto di pausa        cancel dello scope      (meglio fermarsi che parlarci sopra)
```

**Classificazione a livelli**, dal più economico al più costoso:

| Livello | Segnale | Regola indicativa |
|---|---|---|
| **L1 · Acustico** | VAD con soglia più alta durante il playout (§1.4), durata minima del parlato, **voicing** (presenza di una f0 stabile) | Eventi < 250 ms o impulsivi senza f0 (tosse, colpi) → rumore. Opzionale: un piccolo classificatore di eventi audio (tosse, risata) |
| **L2 · Lessicale** | Partial STT | **Fast path per lo stop**: "aspetta", "stop", "basta", "fermati", "no" → hard stop immediato. Se l'enunciato è interamente contenuto nel **lessico di backchannel** ed è breve (≤ 2 parole) → backchannel. Se ci sono ≥ 3 parole fuori dal lessico → barge-in |
| **L3 · Contestuale** | Frase che l'agente stava dicendo e partial dell'utente | Un classificatore leggero risolve i casi ambigui: "sì" mentre l'agente elenca opzioni è un backchannel, "sì" dopo "Procedo con il pagamento?" è una risposta. In quel caso l'agente aveva comunque finito il turno |

Note:
- Il lessico di backchannel e quello di stop sono **configurazione per locale**, non codice.
- Il backchannel **non entra nella storia** come turno utente. Viene annotato come segnale di ascolto per il modulo emozioni (§5).
- Il resume parte dal campione esatto di pausa, grazie al registro di riproduzione descritto in §4.2.

### 4.2 Interruzione mentre l'agente parla

Fermare la generazione non basta. Due problemi vengono sistematicamente trascurati.

#### A. L'audio già in viaggio: svuotare i buffer

**Insidia**: il TTS genera audio **più veloce del tempo reale**, spesso 5–20 volte. Se il server spinge tutto appena è pronto, al momento dell'interruzione ci sono secondi di audio già accodati, e l'utente continua a sentire l'agente anche dopo lo "stop".

| Punto della catena | Misura |
|---|---|
| **Coda in uscita del server** | Pacing in tempo reale: il server invia audio al ritmo di riproduzione e tiene al massimo ~100–200 ms di anticipo. Al barge-in **svuota la coda** (`clear_queue` sulla sorgente audio) |
| **Rete e jitter buffer del client** | Con WebRTC il jitter buffer è piccolo (~40–80 ms) e si scarica da solo. In più si invia un evento `agent_audio_stop` sul **data channel**: il client azzera subito il volume della traccia e ferma il labiale dell'avatar (punti 6–7) |
| **Stream TTS e LLM** | Cancel del context TTS e abort della richiesta LLM, per non pagare token e audio che nessuno sentirà |

**SLO**: *stop latency* (inizio del parlato dell'utente → audio dell'agente silenzioso sul client) **p95 ≤ 300 ms**. Con la soft pause la percezione è ancora migliore, perché l'agente "fa silenzio" già in fase 1.

#### B. Context truncation: l'LLM deve sapere cosa l'utente ha *davvero* sentito

**Insidia**: nella storia della conversazione l'LLM ha scritto una risposta completa di 6 frasi, ma l'utente ne ha sentite una e mezza. Se la storia non viene corretta, al turno successivo l'agente dà per scontate informazioni mai ascoltate ("come ti dicevo, il volo delle 18 è pieno…").

**Meccanismo: registro di riproduzione allineato a livello di parola**

1. Il TTS restituisce **timestamp per parola** (o per carattere) insieme all'audio. Molti TTS streaming lo supportano tramite eventi di alignment o word boundary.
2. Per ogni chunk audio l'orchestratore registra nel **playout ledger** la mappa `campioni → parole`, con offset cumulativo dall'inizio della risposta.
3. La **posizione realmente riprodotta** è data dai campioni effettivamente inviati dal pacer, meno la latenza stimata di downlink e jitter buffer (~50–80 ms).
4. Al momento dello stop (`t_stop`) la parte ascoltata contiene le parole con `end_time ≤ t_stop`. La parola a metà si **esclude** (stima prudente).

**Insidia nell'insidia: il testo pronunciato non è il testo generato.** Il normalizzatore (§1.2) trasforma `€ 1.250` in "milleduecentocinquanta euro", e i timestamp del TTS si riferiscono al testo **normalizzato**. Chunker e normalizzatore devono quindi mantenere una **mappa degli offset** (span normalizzato → span sorgente). Così il taglio si riporta sul testo originale dell'LLM senza spezzare un'entità a metà.

**Riscrittura della storia**

| | Contenuto del messaggio assistant nella storia |
|---|---|
| Generato | `<answer>Ho trovato tre voli per Roma. Il primo parte alle 9:40 ed è il più economico, il secondo alle 13:15, il terzo alle 18 ma è quasi pieno.</answer>` |
| Ascoltato fino a `t_stop` | "Ho trovato tre voli per Roma. Il primo parte alle nove e quaranta ed è…" |
| **Salvato nella storia** | `Ho trovato tre voli per Roma. Il primo parte alle 9:40 ed è… [interrotto dall'utente]` |

Regole:
- Si tronca **solo il testo parlato**. I `tool_use` e `tool_result` già eseguiti restano integri, perché sono accaduti davvero.
- Il marker `[interrotto dall'utente]` (spiegato nel system prompt) segnala al modello che l'utente potrebbe voler correggere la rotta.
- Preamboli e filler deterministici (§1.3) non entrano nella storia, oppure entrano marcati come tali.
- Si modifica solo l'**ultimo** messaggio assistant, quindi il prefisso in cache del prompt resta valido (prompt caching preservato).

### 4.3 Interruzione mentre l'agente esegue un tool

Qui la posta in gioco non è più l'esperienza utente ma **la coerenza del mondo reale**. Interrompere una lettura costa al massimo una query sprecata. Interrompere una transazione a metà può lasciare uno **stato sconosciuto**: la prenotazione è stata fatta oppure no?

**Meccanismo**: ogni tool riceve un **token di cancellazione** nel proprio contesto di esecuzione, propagato fino all'I/O:

```python
async def run(self, args: dict, ctx: ToolContext) -> ToolResult:
    # ctx.cancel: CancellationToken (asyncio/anyio) → AbortSignal per fetch in JS,
    # deadline/cancel per gRPC, cancel della query sul DB, ecc.
```

**Classificazione nel tool registry** (estende i metadati di §1.3):

```yaml
tool: book_flight
side_effect: transactional    # none | idempotent | transactional
two_phase: true               # supporta reserve → confirm
compensation: cancel_booking  # azione di compensazione (saga)
requires_confirmation: true   # conferma vocale esplicita prima dell'esecuzione
```

**Politiche per classe e fase**

| Classe | Fase al momento del barge-in | Politica |
|---|---|---|
| **Sola lettura** (`none`): ricerca, SELECT, GET | Qualsiasi | **Abort immediato** tramite token. Il risultato parziale si scarta. Se la chiamata termina comunque, il risultato va in una **cache breve** indicizzata per argomenti: spesso la correzione dell'utente ("e venerdì?") riusa metà del lavoro |
| **Idempotente**: upsert con idempotency key | Qualsiasi | Abort consentito: un retry successivo con la stessa chiave non duplica |
| **Transazionale** | **Prima del dispatch** (in attesa di conferma o in coda) | **Annullare**: non è successo nulla. È il caso più frequente grazie al gate di conferma |
| **Transazionale** | **In volo** (richiesta già inviata) | **Mai abort cieco**: chiudere la connessione lato client *non* annulla il commit lato server. La chiamata viene **protetta dalla cancellazione** (sezione non cancellabile, es. `asyncio.shield`) e si attende l'esito con un timeout. Intanto l'agente cede comunque la parola: l'audio si ferma subito, e solo il tool prosegue in background |
| **Transazionale** | **Dopo il commit** | Il risultato entra nella storia. L'agente lo comunica in modo trasparente ("La prenotazione era già partita ed è confermata, vuoi che la annulli?"). La **compensazione** (saga) parte solo su richiesta esplicita dell'utente |
| **Transazionale a due fasi** | Tra `reserve` e `confirm` | Si salta il `confirm` e la riserva scade o viene rilasciata: è la soluzione più pulita, quando l'API la offre |

**Coerenza della storia dopo la cancellazione**: le API dei modelli richiedono che ogni `tool_use` abbia il suo `tool_result`. Per un tool annullato si inserisce un risultato sintetico esplicito, ad esempio `{"status": "cancelled_by_user"}` oppure `{"status": "completed_after_interrupt", ...}`, seguito dal nuovo turno utente. Senza questo passaggio la richiesta successiva fallisce, oppure il modello ripete la chiamata.

**Garanzie da testare** (chaos test: barge-in iniettato in ogni fase di ogni tool):
- **Zero side-effect orfani**: nessuna transazione completata senza che sia registrata nella storia e comunicata all'utente.
- **Zero duplicati**: idempotency key su ogni chiamata transazionale.
- Storia sempre valida per l'API: nessun `tool_use` senza risultato.

### 4.4 Metriche

| Metrica | Target |
|---|---|
| Stop latency p95 (inizio parlato → silenzio sul client) | ≤ 300 ms |
| Falsi barge-in (tosse, backchannel che fermano l'agente) | < 1 ogni 20 min di conversazione (da tarare) |
| Barge-in mancati (l'utente deve ripetere "stop") | ~0, è l'errore più frustrante |
| Accuratezza del troncamento (parole nella storia vs parole riprodotte) | ±1 parola |
| Side-effect orfani / duplicati | 0 |

---

## 5. Le emozioni

> **Obiettivo**: capire *come* l'utente sta parlando oltre a *cosa* dice, e usarlo per adattare in modo misurabile il comportamento dell'agente.

### 5.1 Perché il solo testo non basta

- **Lo STT è un collo di bottiglia per l'informazione**: trascrive le parole e scarta la prosodia. "Va bene" può essere sollievo, rassegnazione o rabbia trattenuta, ma la trascrizione è identica.
- **La frustrazione si sente prima di leggersi**: tono più alto, ritmo più rapido e voce più tesa precedono di diversi turni le parole esplicite ("non funziona niente!").
- **Ironia e sarcasmo** invertono la polarità del testo.
- È il costo dichiarato della scelta cascade (ADR-001), e questo modulo serve a **ripagarlo**.

### 5.2 Architettura: un ramo parallelo, fuori dal percorso critico

```
audio frames ─┬──────────────────────────────▶ VAD → STT → Turn Manager → … (percorso critico)
              │
              └─▶ [Feature extractor acustico] ─┐
                                                │
   trascrizione ─▶ [Classificatore lessicale] ──┼─▶ [Fusione + baseline utente] ─▶ Affect State
                                                │        (EMA + isteresi)              │
   eventi pipeline ─▶ [Segnali comportamentali] ┘                                     ▼
   (barge-in, ripetizioni, errori tool)                                        [Policy engine]
                                                                     ┌──────────┼───────────┐
                                                                     ▼          ▼           ▼
                                                                LLM (stile,   TTS        Escalation
                                                                max_tokens)   (velocità,  umana
                                                                              stile)
```

**Vincolo di latenza: zero millisecondi aggiunti al V2V.** Le feature si calcolano in modo incrementale *mentre l'utente parla*. Il modello sull'enunciato gira in parallelo alla finalizzazione dello STT. Se lo stato non è pronto al momento del commit si usa **quello del turno precedente**: il ramo emotivo non blocca mai la pipeline.

### 5.3 Le tre famiglie di segnali

| Famiglia | Feature | Strumenti | Cosa coglie bene |
|---|---|---|---|
| **Acustica e prosodica** | **Pitch** (f0: media, range, varianza), **energia** (RMS, loudness), **speech rate** (sillabe/s, o parole/s dai timestamp STT), rapporto pause/parlato, jitter e shimmer (tensione vocale) | Set di feature eGeMAPS (openSMILE) per l'interpretabilità; modelli SER pre-addestrati su embedding self-supervised (wav2vec2/HuBERT/emotion2vec) per la robustezza | **Arousal** (attivazione: agitazione vs calma), fretta |
| **Lessicale** | Sentiment, marcatori di frustrazione ("ti ho già detto", "assurdo"), marcatori di confusione ("cioè?", "non ho capito") | Classificatore piccolo sulla trascrizione | **Valenza** (positivo/negativo), confusione esplicita |
| **Comportamentale** | Ripetizione dello stesso intento, frequenza dei barge-in, riformulazioni, esitazioni lunghe, confidenza STT bassa, serie di errori dei tool | **Gratis**: sono eventi che la pipeline produce già (§2, §4) | Frustrazione e confusione *operative*, con alta precisione |

**Decisioni di progetto**
1. **Baseline per utente**: le feature acustiche si normalizzano (z-score) rispetto ai primi N secondi della sessione. C'è chi parla naturalmente forte e veloce, e senza baseline il sistema lo classificherebbe sempre come "agitato". È il singolo fattore più importante per ridurre i falsi positivi.
2. **Dimensioni continue invece di categorie**: i modelli stimano arousal e valenza, più robusti delle 6 emozioni di Ekman (che nei dataset sono spesso recitate). Lo spazio continuo viene poi mappato su **stati operativi**: classificare emozioni non è lo scopo, lo scopo è decidere azioni.
3. **Late fusion calibrata**: ogni ramo produce punteggi con la propria confidenza, e un combinatore semplice (pesato o un piccolo modello calibrato) produce lo stato. I segnali comportamentali pesano di più, perché sono i più affidabili.
4. **Smoothing temporale**: media mobile esponenziale (EMA) sui turni e **isteresi** sulle soglie, così lo stato non oscilla da un turno all'altro e l'agente non cambia "personalità" ogni frase.

### 5.4 Dagli stati alle azioni: il policy engine

| Stato operativo | Segnali tipici | **LLM** (concisione e stile) | **TTS** | Altre leve deterministiche |
|---|---|---|---|---|
| **Neutro** | — | Default | 1.0× | Default |
| **Frustrato** | Arousal alto e valenza negativa, ripetizioni, barge-in frequenti | Un solo riconoscimento breve e non stucchevole, poi dritto al punto. Proporre alternative concrete | Leggermente più lento (~0.95×) e tono calmo. **Non rispecchiare** l'agitazione, ma de-escalare | Punteggio di escalation in aumento |
| **Confuso** | Esitazioni, "cioè?", riformulazioni, confidenza STT bassa | Frasi semplici, **un'informazione per volta**, verifica della comprensione ("Fin qui ti torna?") | ~0.9×, pause più lunghe tra le frasi | **Endpointing più paziente** (§2.2): attesa massima aumentata |
| **Di fretta** | Speech rate alto, enunciati brevi, barge-in per saltare | **Massima concisione**: solo il dato richiesto, niente preamboli | ~1.1× | `max_tokens` ridotto, preamboli L1 e filler non essenziali soppressi (§1.3), endpointing più rapido |

**Come si applica ogni leva**
- **Leve deterministiche** (velocità TTS, `max_tokens`, politica dei filler, parametri di endpointing): le imposta direttamente l'orchestratore. Sono garantite e testabili.
- **Leve di prompt** (tono, struttura della risposta): un blocco dinamico inserito **in coda** al contesto, dopo il prefisso in cache, così il caching resta valido:

```xml
<user_state confidence="0.78">frustrated
  segnali: 3 ripetizioni della richiesta, 2 interruzioni, tono in aumento
  linee guida: riconosci brevemente, vai al punto, offri un'alternativa o un operatore
</user_state>
```

- **TTS**: parametro di velocità e, dove il provider lo supporta, controlli di stile ed emozione (oppure SSML `<prosody rate>`).

### 5.5 Escalation umana

**Punteggio di escalation** cumulativo nella sessione, con regole esplicite:

| Trigger | Tipo | Azione |
|---|---|---|
| Richiesta esplicita ("voglio parlare con una persona") | Lessicale e **deterministico** | Escalation **immediata**, senza negoziare |
| Frustrazione sostenuta per ≥ K turni **e** almeno un segnale operativo (stesso intento ripetuto ≥ 3 volte, errori dei tool, loop) | Combinato | **Proposta** all'utente: "Vuoi che ti passi a un collega?" |
| Solo segnale acustico, anche forte | — | **Mai da solo**: adatta lo stile ma non fa escalation, perché i falsi positivi costano in fiducia e in costo operativo |
| Dominio sensibile (reclami, pagamenti falliti) | Configurazione | Soglie più basse |

**Passaggio di consegne a caldo**: l'operatore entra nella stessa room WebRTC (o tramite transfer SIP) e riceve un **riassunto strutturato**: intento, dati già raccolti, tool eseguiti e relativi esiti, stato emotivo e motivo dell'escalation. L'utente non deve ripetere nulla, perché la ripetizione è la causa principale di frustrazione dopo un passaggio.

### 5.6 Rischi, compliance e bias

- **Regolamentazione**: il Regolamento (UE) 2024/1689 (AI Act) tratta in modo specifico i sistemi di riconoscimento delle emozioni: sono vietati in ambito lavorativo e negli istituti di istruzione salvo motivi medici o di sicurezza (**Art. 5, par. 1, lett. f**) e negli altri contesti prevedono obblighi di trasparenza esplicita verso le persone esposte (**Art. 50, par. 1 e 3**). Conseguenze di progetto: **informativa esplicita** all'utente, uso **solo in sessione** per adattare lo stile, **nessuna profilazione persistente** e nessuna decisione con effetti rilevanti basata sullo stato emotivo. Da validare con il legal.
- **Bias**: accento, età, genere e condizioni del canale (telefono vs app) spostano le feature acustiche. Servono valutazioni stratificate per gruppo, e la baseline per utente (§5.3) attenua il problema.
- **Dataset**: i corpora emotivi (anche italiani, come EMOVO) sono spesso **recitati**, quindi le emozioni sono esagerate rispetto a quelle spontanee. Il modello va calibrato su conversazioni reali del dominio annotate internamente.
- **Principio di prudenza**: in caso di dubbio lo stato resta "neutro". Un agente che risponde con empatia a qualcuno che non è frustrato risulta peggiore di uno semplicemente neutro.

### 5.7 Metriche

| Livello | Metrica |
|---|---|
| Offline | Concordanza (es. CCC su arousal e valenza, F1 sugli stati operativi) con annotazioni umane del dominio, stratificata per gruppo |
| Online | **A/B test della policy di adattamento**: task completion, durata della conversazione, tasso di abbandono, CSAT |
| Escalation | Tasso di escalation, **tasso di false escalation** (l'operatore la giudica non necessaria), tempo di risoluzione dopo il passaggio |

---

## 6. L'avatar

> **Obiettivo**: dare un volto all'agente **senza aggiungere latenza** al V2V (§1.1), restando sincronizzato con voce, turni e interruzioni.

**Principio guida: l'avatar consuma eventi, non decide nulla.** L'orchestratore emette un **Avatar Event Protocol** indipendente dal tipo di rendering, che contiene lo stato conversazionale, la timeline dei visemi, i tag di espressione e lo stato emotivo dell'utente. La scelta del renderer diventa così un dettaglio sostituibile.

### 6.1 I due approcci

| Dimensione | **A · Client-side 3D/WebGL** | **B · Server-side neural rendering via WebRTC** |
|---|---|---|
| Come funziona | Il modello 3D (GLB/VRM con blendshape ARKit, oppure 2D tipo Live2D/Rive) è renderizzato nel browser con Three.js o Babylon.js. Il server invia **audio + eventi** (visemi, espressioni) | Un modello generativo su GPU (talking head audio-driven, da vendor o open source) produce i frame del volto a partire dall'audio e li invia come **traccia video WebRTC** |
| Realismo | Stilizzato o "semi-realistico". Evita l'uncanny valley per scelta di design | **Fotorealistico**, anche il clone di una persona reale |
| Latenza aggiunta al V2V | **~0 ms**: il rendering locale gira in parallelo alla riproduzione | **+100–300 ms o più**: la generazione dei frame impone di **ritardare l'audio** per tenerlo sincronizzato con il video |
| Costo marginale per sessione | ~0 (calcola il dispositivo dell'utente) | **Alto**: GPU dedicata (o una frazione) per ogni sessione concorrente |
| Scalabilità | Lineare e gratuita sul server | Limitata dalla capacità GPU, va pianificata |
| Banda | Audio + pochi KB/s di eventi | Audio + video (~1–2 Mbps) |
| Requisiti del client | GPU o WebGL decente, consumo di batteria su mobile | Basta decodificare video: funziona su qualsiasi dispositivo (kiosk, smart TV, client leggeri) |
| Controllo delle espressioni | **Totale e deterministico**: parametri, blendshape, layer (§7) | Limitato a ciò che il modello espone (alcuni accettano condizionamento su emozione o stato) |
| Interruzioni (§4.2) | Stop istantaneo all'evento `agent_audio_stop` | Bisogna svuotare anche la pipeline video (frame già generati e in coda) |
| Personalizzazione del brand | Asset creato da un designer, modificabile in qualsiasi momento | Ripresa video di un attore, con **consenso**, gestione dei diritti d'immagine e rischio deepfake |
| Compliance | Trasparenza standard sull'uso di AI | Obbligo di dichiarare il contenuto sintetico realistico (AI Act, trasparenza), oltre al consenso della persona clonata |

### 6.2 Quando scegliere quale

| Scegli **A (client-side)** quando… | Scegli **B (neural server-side)** quando… |
|---|---|
| Il requisito dominante è la **latenza** (il nostro SLO ≤ 700 ms) | Il requisito dominante è la **fiducia percepita** (banca, sanità, vendita consulenziale, front desk) |
| Servono **molte sessioni concorrenti** con costo marginale ~0 (consumer, education, supporto di massa) | Le sessioni concorrenti sono **poche e di alto valore** e giustificano il costo GPU |
| Servono **espressività e controllo fine** (§7), anche su un personaggio stilizzato | Serve il **volto di una persona reale** o un brand ambassador fotorealistico |
| I client sono app o browser moderni con GPU | I client sono **leggeri** (kiosk, TV, hardware datato) |

**Raccomandazione**: **A come default**, coerente con l'ADR-001 (latenza e controllo deterministico). **B come tier premium** da attivare per verticali specifici, solo se il fornitore garantisce una latenza aggiunta ≤ ~150 ms con generazione in pipeline. Grazie all'Avatar Event Protocol il passaggio da A a B non tocca l'orchestratore.

### 6.3 Avatar Event Protocol (contratto comune)

Gli eventi viaggiano sul **data channel WebRTC** (affidabile e ordinato). I tempi sono espressi **nel dominio audio**, cioè in secondi dall'inizio del segmento di parlato (campioni / sample rate), **mai in orario di parete**:

```json
{ "type": "agent_state", "state": "thinking", "turn_id": 42 }

{ "type": "speech_chunk", "seg": 42, "chunk": 3, "t0": 2.000,
  "visemes": [ {"v": "PP", "t": 0.000}, {"v": "aa", "t": 0.062}, {"v": "E", "t": 0.131} ],
  "expr":    [ {"name": "happy", "i": 0.6, "t": 0.120, "dur": 1.5} ] }

{ "type": "agent_audio_stop", "seg": 42, "reason": "barge_in" }

{ "type": "user_affect", "state": "frustrated", "confidence": 0.78 }
```

Con l'approccio B lo stesso flusso va al renderer server-side invece che al client.

---

## 7. Labiale ed espressioni

> **Obiettivo**: una bocca che articola **esattamente** ciò che si sente, senza deriva su risposte lunghe, e un volto che esprime stati coerenti con la conversazione e con il contenuto.

### 7.1 Lip-sync: visemi con timestamp

**Visemi**: sono le forme della bocca corrispondenti a gruppi di fonemi visivamente indistinguibili (es. /p/, /b/, /m/ → labbra chiuse). Si usa un set standard: i 15 visemi del modello Oculus/Meta, oppure i set dei provider TTS. I visemi si mappano sulle blendshape del modello 3D (es. ARKit: `jawOpen`, `mouthClose`, `mouthFunnel`…).

**Fonti della timeline dei visemi**, in ordine di preferenza:

| Fonte | Come | Pro / contro |
|---|---|---|
| **1. TTS con eventi viseme o fonema** | Alcuni TTS emettono direttamente eventi viseme con offset temporale (es. Azure viseme events, Amazon Polly speech marks) | Deterministica e precisa. Dipende dal provider |
| **2. Allineamento TTS + G2P** | Dall'allineamento per carattere o parola (già usato nel playout ledger, §4.2) → conversione grafema→fonema → visema, con durate distribuite nello span della parola | Funziona con qualsiasi TTS che dia timestamp. Precisione leggermente inferiore |
| **3. Audio-driven** | Un modello che stima i visemi o le blendshape direttamente dall'audio (es. approcci tipo Audio2Face, o analisi spettrale leggera) | Indipendente dal TTS. Costa calcolo e aggiunge una piccola latenza. Utile come fallback |

#### Il problema: la deriva tra audio e animazione

L'approccio ingenuo applica ogni visema quando arriva il messaggio, oppure con `setTimeout` o `requestAnimationFrame` basandosi su `Date.now()`. È destinato a desincronizzarsi:

| Fonte di deriva | Effetto |
|---|---|
| **Percorsi di rete diversi** (audio RTP con jitter buffer vs data channel SCTP) | Offset iniziale variabile tra bocca e voce |
| **Jitter buffer adattivo** (NetEQ di WebRTC **accelera o rallenta** l'audio per assorbire il jitter) | La timeline dell'audio si stira rispetto all'orologio di parete, e la deriva **cresce** nel tempo |
| **Clock skew** tra il cristallo della scheda audio e l'orologio di sistema | Decine di ppm che si accumulano su risposte lunghe |
| **Throttling** dei timer (tab in background, carico CPU) | Visemi in ritardo o saltati |

#### La soluzione: l'audio è il master clock

**Regola**: l'unico orologio di riferimento per l'animazione è **`AudioContext.currentTime`**, che avanza con il clock di campionamento del dispositivo di uscita. Se l'audio si ferma, rallenta o va in underrun, l'animazione lo segue *per costruzione*.

**Implementazione di riferimento** (avatar client-side, approccio A):

1. **Audio e visemi viaggiano insieme**: ogni `speech_chunk` (§6.3) porta l'audio del chunk (Opus decodificato con WebCodecs, oppure PCM) **e** la sua timeline di visemi ed espressioni. La sincronizzazione è garantita perché i due dati sono un'unità atomica.
2. **Scheduling sull'AudioContext**: il client accoda ogni chunk con `source.start(when)` in un piccolo playout buffer (~100–150 ms). Al primo chunk del segmento registra l'**ancora**: `anchor[seg] = when − t0`.
3. **Campionamento della timeline nel render loop**:

```js
function frame(rafNow) {
  // contextTime = istante audio che sta uscendo ORA dagli altoparlanti
  const { contextTime, performanceTime } = ctx.getOutputTimestamp();
  const tHeard = contextTime + (rafNow - performanceTime) / 1000;   // allineato al frame
  const t = tHeard - anchor[seg];                                    // secondi nel segmento (dominio audio)

  face.setVisemes(timeline.sample(t + VISUAL_LEAD));   // VISUAL_LEAD ≈ 20–40 ms
  face.setExpressions(exprTrack.sample(t));
  requestAnimationFrame(frame);
}
// fallback se getOutputTimestamp non è disponibile: tHeard ≈ ctx.currentTime − ctx.outputLatency
```

4. **Coarticolazione**: i visemi non si cambiano di scatto. Si interpola con attacco e rilascio (~30–60 ms) e si somma un contributo `jawOpen` proporzionale all'energia dell'audio, per avere naturalezza anche quando la timeline è approssimata (fonte 2).
5. **Anticipo visivo**: la percezione umana tollera meglio l'audio in ritardo che in anticipo rispetto al labiale (soglie di rilevabilità ITU-R BT.1359: circa +45 ms con audio in anticipo, circa −125 ms con audio in ritardo). Per questo i visemi vengono anticipati di qualche decina di millisecondi (`VISUAL_LEAD`).
6. **Barge-in** (§4.2): all'evento `agent_audio_stop` si chiama `stop()` su tutte le sorgenti schedulate, si svuota la timeline e la bocca torna a riposo in ~80 ms. Bocca e voce si fermano nello stesso istante.

**Rischio noto e mitigazione**: riprodurre l'audio dell'agente tramite Web Audio invece che come traccia WebRTC può **escludere quell'audio dal riferimento dell'AEC** del browser, con alcuni browser e piattaforme, e far riemergere il problema dell'eco (§1.4). Da verificare sui browser target. Le mitigazioni sono due: instradare l'output dell'AudioContext in un `MediaStreamDestination` attraverso un loopback `RTCPeerConnection`, oppure, nelle app native, usare l'AEC di sistema, che cancella tutto l'output. **Alternativa**: mantenere la traccia WebRTC e **ri-ancorare periodicamente** la timeline con `RTCRtpReceiver.getSynchronizationSources()` (mappa timestamp RTP → istante di riproduzione) più `getOutputTimestamp()`. È meno precisa, ma assorbe lo stretching di NetEQ.

**Con l'approccio B** (neural server-side) la sincronizzazione avviene sul server: il modello genera i frame dall'audio e audio e video viaggiano nella stessa sessione WebRTC, sincronizzati tramite timestamp RTP e RTCP (lip-sync group).

### 7.2 Espressioni: architettura a 3 layer di animazione

Un volto credibile non è "una animazione per emozione". È la **composizione di layer indipendenti** con frequenze e fonti diverse:

```
┌───────────────────────────────────────────────────────────────────────┐
│ LAYER 3 · Semantico-emotivo   fonte: tag LLM + Affect State utente    │  sporadico, legato
│   happy · empathetic · concerned · surprised · emphasis               │  a parole precise
├───────────────────────────────────────────────────────────────────────┤
│ LAYER 2 · Stato conversazionale   fonte: eventi orchestratore         │  a ogni cambio
│   listening · thinking · speaking · interrupted                       │  di stato
├───────────────────────────────────────────────────────────────────────┤
│ LAYER 1 · Idle procedurale   fonte: generatori locali                 │  sempre attivo
│   battito di palpebre · respiro · micro-saccadi · oscillazione testa  │
└───────────────────────────────────────────────────────────────────────┘
        + VISEMI (§7.1) con priorità sulla regione della bocca
                          ▼
              compositore → pesi blendshape finali → render
```

#### Layer 1: idle procedurale (sempre attivo, 100% client)
- **Battito di palpebre** stocastico (intervalli casuali di ~2–6 s, distribuzione di Poisson, durata ~150 ms) con frequenza modulata dallo stato: più frequente in `thinking`, ridotto mentre l'agente ascolta con attenzione.
- **Respiro**: oscillazione sinusoidale lenta di petto e spalle (~12–16 atti al minuto).
- **Micro-movimenti**: micro-saccadi oculari e lieve oscillazione della testa tramite rumore di Perlin, mai periodico.
- Scopo: un volto fermo appare "morto" anche se il resto è perfetto. Il layer non dipende dal server né dalla rete, quindi l'avatar resta vivo anche durante un problema di connessione.

#### Layer 2: stato conversazionale (deterministico, guidato dagli eventi)

| Stato (evento) | Comportamento |
|---|---|
| `listening` | Sguardo verso l'utente, testa leggermente inclinata. **Cenni del capo** sincronizzati con le pause dell'utente rilevate dal VAD (§2) e con i suoi backchannel |
| `thinking` (esecuzione tool, §1.3) | Sguardo distolto in alto o di lato, sopracciglia leggermente contratte. È la controparte visiva del filler vocale: comunica "sto lavorando" anche in silenzio |
| `speaking` | Movimenti della testa correlati alla prosodia (energia e pitch dell'audio TTS), **beat** (piccoli accenti) sulle parole enfatizzate |
| `interrupted` (§4.2) | Arresto rapido, lieve arretramento, sguardo attento: "ti ascolto" |

Le transizioni passano per la macchina a stati con tempi di blend (~200–400 ms), mai tagli secchi.

#### Layer 3: semantico-emotivo (guidato dai tag dell'LLM)

**Meccanismo**: si estende il protocollo di output (§3.2) con un tag auto-chiudente di espressione, inline nella risposta:

```xml
<answer><expr name="happy" i="0.6"/>Ottima notizia: il volo è confermato!
<expr name="concerned" i="0.4"/>Però il bagaglio da stiva non è incluso.</answer>
```

- **Parser**: lo stream parser riconosce `<expr …/>` (prefisso `<expr ` più lettura degli attributi fino a `>`, con lunghezza massima limitata, es. 64 caratteri). Il tag viene **rimosso dal flusso verso il TTS** e convertito in un evento.
- **Ancoraggio temporale preciso**: la posizione del tag è un offset di carattere nel testo sorgente. Attraverso la **mappa degli offset** del normalizzatore (§4.2) e l'**allineamento del TTS** diventa un **tempo audio**, e finisce nella stessa timeline dei visemi (`expr` in `speech_chunk`, §6.3). Il sorriso compare *esattamente* quando viene pronunciata "Ottima notizia". Si riusano gli stessi componenti di §3 e §4, senza infrastruttura nuova.
- **Coerenza voce-volto**: lo stesso tag può pilotare anche i controlli di stile o emozione del TTS, dove disponibili, così tono di voce ed espressione coincidono.

**Guardrail deterministici** (l'LLM *propone*, l'orchestratore *dispone*):

| Regola | Motivo |
|---|---|
| **Vocabolario chiuso** (`neutral`, `happy`, `empathetic`, `concerned`, `surprised`, `emphasis`) mappato su preset di blendshape. I tag sconosciuti si scartano | Evita espressioni inventate o grottesche |
| **Clamp dell'intensità** (es. ≤ 0.7) e **rate limit** (al massimo un'espressione per frase) | Niente "faccia da cartone animato" |
| **Decadimento verso neutro** dopo `dur` (default ~1.5–2 s) | Le espressioni sono eventi, non stati permanenti |
| **Coerenza con l'Affect State** (§5): con l'utente `frustrated` si sopprime `happy` e si usa `empathetic` o `concerned` come base | Un avatar che sorride a un utente arrabbiato è peggio di nessun avatar |
| **Default senza tag**: espressione derivata dallo stato dell'utente e da un sentiment leggero della frase | Fallback robusto se il modello non emette tag |

#### Compositore: chi possiede quale regione del volto

| Regione | Proprietario primario | Regola di composizione |
|---|---|---|
| **Bocca e mascella** | **Visemi** | Layer 3 (es. sorriso) si somma con peso ridotto e vincolato (clamp) per non compromettere la leggibilità del labiale |
| **Occhi e palpebre** | Layer 1 (blink) | Il blink ha sempre la precedenza, salvo occhi già chiusi da un'espressione |
| **Sopracciglia e fronte** | Layer 3 > Layer 2 | L'espressione semantica sovrascrive lo stato conversazionale |
| **Sguardo e testa** | Layer 2 | Layer 1 aggiunge micro-movimenti in modo additivo |

### 7.3 Metriche

| Metrica | Target |
|---|---|
| Offset A/V del labiale (misurato registrando l'uscita audio e i tempi dei frame) | p95 entro ±40 ms, con visivo in leggero anticipo |
| Deriva su un parlato continuo di 60 s | ~0 (per costruzione, con il master clock audio) |
| Stop congiunto bocca + voce al barge-in | Stesso frame ±1 |
| Frame rate su dispositivi target di fascia bassa | ≥ 30 fps |
| Naturalezza e appropriatezza delle espressioni | MOS da test percettivi, appropriatezza giudicata da annotatori sul contesto |

---

## Sintesi delle decisioni

| # | Area | Decisione chiave | Alternativa scartata (e perché) | Metrica guida |
|---|---|---|---|---|
| 0 | Architettura | Cascade in streaming su WebRTC, orchestratore ReAct proprietario | STS nativo: poco controllo su tool e chain of thought | — |
| 1 | Voce | Budget di latenza per stadio, normalizzazione deterministica, filler a livelli, AEC multilivello | STT/TTS "appesi" al loop testuale: latenza e registro sbagliati | V2V p50 ≤ 700 ms |
| 2 | Turni | VAD acustico propone, turn detector semantico decide, attesa dinamica | Soglia di silenzio fissa: trade-off irrisolvibile | Interruzioni premature vs endpoint latency |
| 3 | Solo la risposta | Default deny: routing nativo a blocchi + parser XML a stati | Filtri regex e post-hoc: fragili in streaming | Leak rate = 0 |
| 4 | Interruzioni | Soft pause → hard stop, troncamento del contesto via timestamp TTS, cancellazione per classe di tool | Stop al primo suono; abort cieco dei tool transazionali | Stop latency p95 ≤ 300 ms, side-effect orfani = 0 |
| 5 | Emozioni | Ramo multimodale parallelo, baseline per utente, stati operativi → leve deterministiche + prompt | Sentiment sul solo testo: cieco alla prosodia | A/B su task completion, false escalation |
| 6 | Avatar | Client-side 3D come default, neural server-side come tier premium, stesso protocollo di eventi | Neural come default: +latenza, costo GPU per sessione | Latenza aggiunta ~0 |
| 7 | Labiale ed espressioni | Visemi nel dominio audio sincronizzati su `AudioContext`, 3 layer di animazione + guardrail | Visemi su orologio di parete: deriva | Offset A/V p95 ±40 ms |

---

## Traccia di lavoro

### Conversazioni con assistenti AI

**Claude (Claude Code)**: sessione iterativa di strutturazione, formalizzazione e stesura del documento.

**Metodologia di utilizzo:**
Il lavoro è partito dalla consultazione della tesi di diploma ITS di un amico (*"Dall'idea alla chiamata: Progettazione di agenti vocali AI in ambito Enterprise"*, documento privato non allegato per riservatezza), da cui ho ripreso l'impostazione *cascaded pipeline* in produzione (confronto con S2S, target 700–1000 ms, orchestrazione `Pipecat`, normalizzazione di date/orari fuori dall'LLM, gestione dei falsi positivi nel *barge-in* e *warm transfer* con riepilogo all'operatore). A partire da queste basi, ho definito e guidato nei prompt le decisioni architetturali e i vincoli di progettazione per tutti i 7 punti (generazione preventiva, principio *default deny* con stream parser XML a stati, gestione transazionale dei tool su barge-in e context truncation con mappa degli offset, ramo emotivo parallelo con baseline per utente, sincronizzazione visemi su `AudioContext.currentTime` e architettura a 3 layer). L'assistente AI è stato utilizzato per formalizzare le macchine a stati, le tabelle comparative, i casi limite e lo pseudocodice.

<details>
<summary><strong>Trascrizione dei Prompt e Risposte della sessione (clicca per espandere)</strong></summary>

#### Iterazione 1 — Architettura di riferimento e Punti 1–3
**Prompt 1:**
> voglio progettare una proposta architetturale per trasformare un classico agente testuale con tool calling e ragionamenti intermedi in un assistente voice real time con avatat.
>
> voglio strutturare la mia proposta partendo da una scelta architetturale precisa, ovvero preferisco una pipeline cascade in streaming (es. stt - llm - tts) su webrtc rispetto a un modello sts puro come openai realtime, perché con tool calling complessi e multistep ho bisogno di controllo deterministico su ciclo ReAct e sul filtraggio del chain of thought, pur mantenendo la latenza entro i 700ms grazie allo streaming end to end.
>
> aiutami a strutturare in modo rigoroso e schematico i primi 3 punti:
>
> la voce; voglio coprire la latenza cumulativa, differenza registro scritto/parlato (tipo markdown contro voice first prompting + text normalization), silenzi durante l'esecuzione dei tool e il problema dell'eco.
>
> i turni di parola, ovvero come superare il limite di una soglia di silenzio fissa combinando un vad acustico con un smart endpoint semantico (come modelli leggeri tipo livekit turn detector) per non interrompere l'utente mentre pensa.
>
> la sola risposta: ovvero come evitare che il tts legga i ragionamenti intermedi e le tool call, usando sia la separazione nativa API sia uno stream parser a macchina a stati basato su tagXML
>
> scrivimi questa prima parte con un taglio da system design document chiaro, pragmatico e orientato alla produzione

**Sintesi risposta 1:** Stesura di `§0` (ADR-001: confronto Cascade vs STS e architettura di riferimento), `§1` (budget di latenza p50/p95 con generazione preventiva, voice-first prompting + normalizzazione per l'italiano, strategia a 5 livelli per i filler guidata dal tool registry, difesa anti-eco a 5 strati), `§2` (endpointing ibrido VAD + turn detector semantico con attesa dinamica e macchina a stati del Turn Manager), `§3` (principio *default deny*, routing a blocchi nativo + parser XML a stati con buffer di lookahead e pseudocodice).

---

#### Iterazione 2 — Interruzioni (Barge-in) ed Emozioni (Punti 4–5)
**Prompt 2:**
> easy, ora aprofondiamo altri 2 punti mantenendo lo stesso rigore architetturale:
>
> le interruzioni, ovvero voglio distinguere 3 aspetti critici che di solito vengono dimenticati:
>    - come evitare i falsi positivi tipo colpi di tosse o backchanneling come 'uh-huh'
>    - cosa succede se interrompo mentre l'agente: svuotamento del buffer webRTC e soprattutto il context truncation tramite i word level timestamps del tts, così l'lllm sa esattamente fino a quale parola l'utente ha sentito davvero.
>    - cosa succede se interrompo mentre l'agente esegue un tool: l'uso di cancellation token o abort controller. distinguendo tra tool di sola lettua e tool transazionali con side effect
>
> l'altro punto sono le emozioni: perché il solo testo non basta. voglio proporre un rilevamento multimodale parallelo (es. feature acustiche/prosodiche come pitch, speech rate, energia) per adattare dinamicamente concizione dell'llm, velocità del tts ed eventuale escalation umana

**Sintesi risposta 2:** Stesura di `§4` (albero di cancellazione per turno, arresto in due fasi *soft pause* → *hard stop*, pacing lato server, *playout ledger* con mappa degli offset tra testo normalizzato e testo sorgente, politiche per classe/fase dei tool con `asyncio.shield` e `tool_result` sintetico) e `§5` (ramo parallelo a latenza zero, fusione acustica/lessicale/comportamentale con normalizzazione sulla baseline dell'utente, mappatura arousal/valenza su stati operativi, policy engine, regole di escalation umana e note su compliance AI Act e bias).

---

#### Iterazione 3 — Avatar, Lip-Sync ed Espressioni (Punti 6–7) e Consolidamento
**Prompt 3:**
> buono, ora chiudiamo il documento con gli ultimi due punti sull'avatar e poi assembla tutto in un unico documento markdown pronto per la consegna.
>
> l'avatar che confronta con una tabella i due approcci principali: client side 3d/webGL contro server side neural rendering fotorealistico via webRTC, motivando quando scegliere l'uno o l'altro.
>
> labiale ed espressioni: il lip-sync che spirga l'uso dei visemi timesstampati sincronizzati rigorosamente sull' 'AudioContext.currentTime' del cliente per evitare drift. riguardo le espressioni, proponi un'architettura a 3 layer di animazione: layer 1 idle procedurale , layer 2 stato conversazionale, layer 3 semantico-emotivo guidato dai tag generati dall'llm.
>
> ora unisci tutti i 7 punti + l'overview architetturale iniziale con un diagramma ASCII in un unico file markdown ben formattato in italiano.

**Sintesi risposta 3:** Aggiornamento del diagramma generale in `§0`, stesura di `§6` (tabella comparativa su 11 dimensioni, criteri di scelta e `Avatar Event Protocol`), `§7` (fonti dei visemi, analisi delle cause di deriva incluso NetEQ, sincronizzazione su `AudioContext` con anticipo visivo ITU-R BT.1359, compositore a 3 layer con tag `<expr/>` e guardrail deterministici) e tabella di sintesi finale.

</details>

### Fonti principali consultate
- **Fonte primaria di partenza (Tesi di diploma ITS di un amico, consultata privatamente):**
  - *"Dall'idea alla chiamata: Progettazione di agenti vocali AI in ambito Enterprise"*, Tesi di diploma del percorso *ITS Artificial Intelligence & Machine Learning Specialist* (Biennio 2024–2026 — documento privato non incluso nel repository per tutela di dati personali e riservatezza aziendale). Consultata come base di partenza pratica sull'architettura *cascaded pipeline* in produzione (compromessi Cascaded vs S2S, budget di latenza entro 700–1000 ms, TTFT/TTFA con modelli MoE/Qwen, `faster-whisper` e `ElevenLabs Flash`), sull'orchestrazione con `Pipecat`, e sulle criticità reali emerse sul campo: pre-processing e normalizzazione deterministica di date/orari e dati strutturati fuori dall'LLM, calibrazione dei falsi positivi nel *barge-in*, mitigazione delle allucinazioni STT tramite soglie di confidenza e *noise gate*, *warm-up* contro il cold start e passaggio di consegne all'operatore umano con contesto.
- **Orchestrazione Voice Agents & Turn Detection:**
  - [LiveKit Agents Documentation — Turn detection & Interruptions](https://docs.livekit.io/agents/build/turns/)
  - [Pipecat (Daily.co) — Open Source Framework for Voice & Multimodal Conversational AI](https://github.com/pipecat-ai/pipecat) e [Smart Turn audio endpointing](https://github.com/pipecat-ai/smart-turn)
  - [Silero VAD — Pre-trained Enterprise-Grade Voice Activity Detector](https://github.com/snakers4/silero-vad)
- **Standard Web Audio, WebRTC e Sincronizzazione A/V:**
  - [W3C Web Audio API — `AudioContext.currentTime` & `getOutputTimestamp()`](https://www.w3.org/TR/webaudio/#dom-audiocontext-getoutputtimestamp)
  - [W3C WebRTC 1.0 — `RTCRtpReceiver.getSynchronizationSources()`](https://www.w3.org/TR/webrtc/#dom-rtcrtpreceiver-getsynchronizationsources)
  - [ITU-R BT.1359-1 — Relative Timing of Sound and Vision for Broadcasting](https://www.itu.int/rec/R-REC-BT.1359)
- **Affective Computing, Speech Emotion Recognition & Normativa:**
  - Eyben et al., [*The Geneva Minimalistic Acoustic Parameter Set (GeMAPS) for Voice Research and Affective Computing*](https://ieeexplore.ieee.org/document/7160715), IEEE Transactions on Affective Computing, 2016 (`openSMILE` / `eGeMAPS`).
  - Costantini et al., [*EMOVO Corpus: an Italian Emotional Speech Database*](https://aclanthology.org/L14-1478/), LREC 2014.
  - [Regolamento (UE) 2024/1689 (EU AI Act)](https://eur-lex.europa.eu/eli/reg/2024/1689/oj) — Art. 5(1)(f) (divieti relativi al riconoscimento delle emozioni) e Art. 50 (obblighi di trasparenza per sistemi AI interattivi e di emotion recognition).
- **Avatar Rendering, Visemi e Blendshapes:**
  - [Apple Developer — ARKit `ARFaceAnchor.BlendShapeLocation` (52 blendshapes)](https://developer.apple.com/documentation/arkit/arfaceanchor/blendshapelocation)
  - [Meta / Oculus Lipsync — Viseme Reference (15 standard visemes)](https://developers.meta.com/horizon/documentation/unity/audio-ovrlipsync-viseme-reference/)
  - [Microsoft Azure AI Speech — Viseme events & facial animation](https://learn.microsoft.com/en-us/azure/ai-services/speech-service/how-to-speech-synthesis-viseme) / [Amazon Polly Speech Marks](https://docs.aws.amazon.com/polly/latest/dg/speechmarks.html)

