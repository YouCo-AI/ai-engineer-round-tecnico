# Da agente testuale ad assistente vocale con avatar

Mohamed Farah

## In breve

Per trasformare l'agente in un assistente vocale metterei in fila tre pezzi che lavorano in streaming: riconoscimento vocale (STT), l'agente con i tool e sintesi vocale (TTS). In streaming vuol dire che ogni pezzo passa il lavoro al successivo mentre sta ancora andando avanti, senza aspettare che finisca. L'obiettivo è che l'utente senta l'inizio della risposta entro circa 1 secondo dalla fine della sua frase (obiettivo da misurare, non un dato).

Ho scelto questa soluzione "a cascata" e non un modello speech-to-speech perché mi permette di controllare meglio i tool, cosa viene detto ad alta voce e le interruzioni. Per non partire da zero userei un framework come LiveKit Agents o Pipecat, che già gestiscono audio in tempo reale, turni e interruzioni.

**Cosa ho già fatto e cosa no.** Nel tirocinio a NewMediaADV ho usato le REST API v2 di HeyGen: davo un testo e HeyGen generava un video con voce e labiale. Controllavo lo stato con il polling e, per le chiamate esterne (Gemini, HeyGen), avevo retry con exponential backoff. Non ho mai fatto riconoscimento vocale in tempo reale, gestione dei turni, interruzioni o un avatar in streaming. Queste parti sono ricerca e proposta, e le segno come tali.

## Schema

    microfono → VAD + fine turno → STT → agente + tool → filtro "solo risposta" → TTS → audio
                     ↑                                                              │
                     └───────────── se l'utente parla: stop (barge-in) ◀────────────┘

## 1. La voce: cosa manca oltre a STT e TTS

Mettere STT in ingresso e TTS in uscita non basta. Mancano:
- **Streaming ovunque.** Senza, la risposta arriva dopo diversi secondi e sembra una chat letta ad alta voce.
- **Testo adatto al parlato.** Elenchi, markdown, link e numeri lunghi suonano male. Serve un prompt che chieda frasi brevi e un passaggio che pulisce il testo prima del TTS.
- **Riempitivi mentre un tool lavora.** Una frase breve come "un attimo, controllo", decisa dal sistema, per non lasciare silenzio.
- **Audio pulito.** Cancellazione dell'eco e del rumore; italiano e nomi propri riconosciuti bene.
- **Errori gestiti.** Timeout corti e un messaggio vocale gentile se qualcosa va storto. Nei miei client verso servizi esterni riprovo gli errori temporanei con backoff; a voce serve anche un piano B.

**Budget di latenza** (valori da misurare, non stimati): fine turno → STT finale → primo token del modello → primo audio del TTS. Misuro ogni pezzo separatamente, così so dove intervenire.

**Alternativa:** un modello speech-to-speech (es. Gemini Live o API realtime di OpenAI). Ha poca latenza e voce naturale, ma ho meno controllo sui tool e su cosa viene detto. Lo proverei come confronto.

## 2. I turni di parola

- Un **VAD** (voice activity detection, es. Silero) capisce se c'è voce o silenzio.
- Aspettare solo un tot di silenzio non basta: se è troppo corto taglio l'utente, se è troppo lungo rispondo tardi.
- Meglio un **modello di fine turno** che guarda anche il significato ("allora…" non è finito). LiveKit lo ha già, come "turn detector"; Pipecat ha le strategie di turno.
- Parto con un'attesa adattiva e misuro: quante volte taglio l'utente e quanto ritardo c'è.
- Pulsante "parla" come piano B se c'è troppo rumore.

## 3. Solo la risposta

L'agente produce ragionamenti, commenti sui tool e la risposta finale. Voglio che solo l'ultima venga letta ad alta voce.
- Divido i flussi **nel codice**: ragionamenti e chiamate ai tool non arrivano mai al TTS. Non mi fido solo del prompt ("non commentare"), perché il modello a volte lo ignora.
- **Come riconosco la risposta finale.** Due opzioni:
  - un campo dedicato (es. `speak` in un JSON): chiaro, ma in streaming devo leggere un JSON ancora incompleto;
  - considerare "risposta finale" il messaggio del modello che non contiene chiamate a tool, oppure far usare al modello un tool `speak(testo)`: più semplice da mandare al TTS frase per frase.
  Partirei dalla seconda e valuterei la prima solo se serve.
- In AI Support Agent (LangGraph) la struttura è già così: nodi che chiamano i tool e un nodo finale che prepara la risposta per l'utente. In versione vocale, solo l'uscita di quel nodo va al TTS.
- I commenti utili ("sto cercando") possono andare a schermo, non a voce.

## 4. Le interruzioni

Quando l'utente parla mentre l'agente sta parlando o lavorando:
1. **Stop immediato** della voce, buttando via l'audio non ancora riprodotto.
2. **Annullo** la generazione del modello e i tool in attesa. Se un tool sta scrivendo dati, lo lascio finire o lo rendo sicuro da ripetere.
3. **Tengo nella conversazione solo ciò che l'utente ha sentito davvero.** Servono i tempi parola per parola del TTS (se il fornitore li dà), così taglio la risposta nel contesto al punto giusto. Altrimenti il modello crede di aver detto frasi mai pronunciate.
4. **Distinguo le interruzioni vere dai "mh" e "ok"**: serve una durata o un numero minimo di parole.
5. **Cancellazione dell'eco**, altrimenti l'agente sente sé stesso e si interrompe da solo.
6. La nuova frase dell'utente entra nel contesto con una nota ("ha interrotto mentre rispondevo"), così l'agente può correggere la rotta.

Questi punti li ho verificati leggendo la documentazione di Pipecat e LiveKit (vedi fonti): fanno già stop, svuotano il buffer e tengono nel contesto solo il testo detto; LiveKit ha opzioni come durata minima, parole minime e ripresa dopo un falso allarme.

## 5. Le emozioni

**Come le capisco**
- **Dal testo trascritto**: un modello leggero classifica se l'utente sembra frustrato, confuso o di fretta.
- **Dall'audio**: velocità, volume, tono e pause (esistono modelli e servizi per questo).
- **Dal comportamento**: ripete la stessa cosa, interrompe spesso, dice "no, ti ho detto…".

**Come le uso**
- Di fretta: risposte più corte, niente giri di parole.
- Confuso: passi piccoli e domanda di controllo.
- Frustrato: tono calmo, riconosco il problema e propongo un operatore umano.
- Stessa emozione anche per lo stile della voce e l'espressione dell'avatar.

**Attenzione**: indovinare le emozioni è incerto, quindi è un segnale leggero e non dico mai all'utente "sei arrabbiato". C'è anche un tema di privacy (GDPR e AI Act): informativa chiara e nessun salvataggio dell'audio se non serve.

## 6. L'avatar

| Opzione | Pro | Contro |
| --- | --- | --- |
| Servizio di avatar (HeyGen, Tavus, Simli, ecc.) | Veloce da integrare, realistico, con labiale | Costo, latenza, dipendi dal fornitore |
| Avatar 3D nel browser (Three.js + VRM) | Controllo totale, leggero, nessun costo a minuto | Meno realistico, più lavoro |
| Avatar 2D/illustrato | Semplice, meno effetto "strano" | Meno espressivo |

**Cosa ho fatto davvero.** Con HeyGen API v2 il flusso era: testo → richiesta di generazione → controllo dello stato a intervalli (polling) → video pronto. Voce e labiale erano gestiti da HeyGen. Non ho misurato in modo preciso quanto ci metteva.

**Il limite.** Un video generato in attesa non va bene per una conversazione. Per il tempo reale servono un servizio con avatar in streaming (che i fornitori citati dichiarano di offrire; da verificare nella loro documentazione prima di sceglierlo) oppure un avatar 3D animato da me. Per un primo prototipo userei un servizio in streaming collegato all'audio; per più controllo e meno costi valuterei il 3D.

## 7. Labiale ed espressioni

- **Labiale.** Si può avere dai **visemi** (le forme della bocca per ogni suono) che arrivano dal TTS, oppure da un modello che li ricava dall'audio. L'importante è che audio e video usino lo stesso orologio. Con HeyGen se ne occupava il servizio.
- **Espressioni.** Per ogni frase il modello indica anche l'emozione (es. `calmo`, `empatico`, `sorridente`) in un campo a parte, e l'avatar la trasforma in espressione e gesti.
- **Ascolto.** Quando parla l'utente, l'avatar passa in posa di ascolto (sguardo, cenni del capo).
- Sulle espressioni dell'avatar non ho esperienza diretta: oltre ai movimenti naturali del servizio non ho fatto altro.

## Rischi e come li misuro

- **Latenza**: obiettivo sotto 1 secondo tra fine frase e primo audio, misurata per ogni pezzo.
- **Turni e interruzioni sbagliate**: quante volte taglio l'utente o non mi fermo.
- **Qualità STT in italiano** e nei nomi propri.
- **Costo a minuto** (STT + modello + TTS + avatar).
- **Privacy**: consenso, pochi dati salvati.

## Piano per fasi

1. Voce in streaming e filtro "solo risposta".
2. Turni (VAD + fine turno) e interruzioni.
3. Emozioni per adattare le risposte.
4. Avatar con labiale ed espressioni.

## Traccia di lavoro

**Come ho usato l'AI**
Ho usato Claude per risparmiare tempo: ho chiesto una prima struttura della proposta, ho confrontato le opzioni (cascata o speech-to-speech) e ho ripassato concetti che non ho mai implementato, come barge-in e rilevamento dei turni. Claude mi ha anche indicato LiveKit e Pipecat come documentazione da controllare.

Poi i controlli li ho fatti io: ho letto le pagine di LiveKit e Pipecat citate sotto per verificare i punti 2, 3 e 4, ho tolto i dettagli di cui non ero sicuro e ho separato ciò che ho fatto davvero (HeyGen API v2 con polling, retry con backoff, AI Support Agent) da ciò che è solo proposta.

**Fonti che ho consultato**
- [LiveKit Agents: turn detection e interruzioni](https://docs.livekit.io/agents/logic/turns). Mi è servita per i turni: VAD, modello di fine turno, attesa minima e massima, e opzioni sulle interruzioni (durata minima, parole minime, ripresa dopo falso allarme).
- [Pipecat: interruzioni](https://docs.pipecat.ai/pipecat/fundamentals/interruptions). Mi è servita per il punto 4: all'interruzione il bot si ferma, la generazione del modello viene cancellata, i buffer del TTS si svuotano e nel contesto resta solo il testo davvero pronunciato.

**Conversazione con Claude**


**Io:** Ti incollo la traccia del test. Ho un'ora. Come imposteresti l'architettura per passare da agente testuale a vocale?

**Claude:** Ci sono due strade. La prima è a cascata: STT → agente con tool → TTS, tutto in streaming. La seconda è un modello speech-to-speech (es. API realtime). La cascata dà più controllo su tool e su cosa viene detto; speech-to-speech ha meno latenza ma è più opaco. Per una proposta partirei dalla cascata e citerei l'altra come confronto.

**Io:** Cosa manca oltre a STT e TTS perché funzioni davvero?

**Claude:** Streaming in ogni passaggio, testo adatto al parlato (niente markdown o elenchi), riempitivi mentre un tool lavora, audio pulito con cancellazione dell'eco, gestione degli errori con un messaggio vocale di riserva. Senza questi punti sembra una chat letta ad alta voce.

**Io:** Come faccio a far sentire solo la risposta finale?

**Claude:** Separando i flussi nel codice, non solo nel prompt: ragionamenti e chiamate ai tool non vanno al TTS, ci va solo il messaggio finale. Il prompt "non commentare" non basta perché il modello a volte lo ignora. Attenzione: se la risposta è in un campo JSON, in streaming devi leggere JSON parziale; un modo più semplice è considerare finale il messaggio senza tool call.

**Io:** E le interruzioni? Non ho mai implementato il barge-in.

**Claude:** Quindi trattalo come ricerca, non come esperienza. In pratica: stop dell'audio, cancellazione della generazione, e nel contesto resta solo ciò che l'utente ha sentito davvero. Per saperlo servono i tempi delle parole dal TTS. Ti consiglio di controllare come lo fanno Pipecat e LiveKit nella documentazione e di scrivere solo ciò che trovi lì.

**Io:** Posso scrivere "circa 1 secondo" di latenza?

**Claude:** Come obiettivo sì, come risultato no, perché non l'hai misurato. Scrivilo come obiettivo e aggiungi che misuri ogni pezzo separatamente.

**Io:** Sull'avatar ho solo usato HeyGen con il polling.

**Claude:** Va bene dirlo com'è: ho usato il video generato, che non va bene per il tempo reale. Poi spiega cosa servirebbe (avatar in streaming o 3D) e segna cosa è da verificare nella documentazione dei fornitori.
