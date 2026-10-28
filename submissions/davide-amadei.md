#Davide Amadei
#1) la voce
#utilizzo gemini, ho scritto nel promt:
#Sto lavorando su trasformare un agente AI testuale in un assistente vocale con cui parlare a voce. La soluzione è utilizzare una componente
#speech to text per trascrivere in testo l'audio dell'utente e alla fine utilizzare una componente text to speech per trasformare in audio la risposta testuale.
#quali sono i problemi in questo utilizzo?
Ho cercato di capire prima quali sono i problemi associati a utilizzare la voce.
Il primo problema evidenziato è una latenza totale tra la richiesta e la risposta dell'agente composta da:
- voice activity detenction (VAD): il sistema deve aspettare tra i 500-800 ms che l'utente smetta di parlare (pari alla pausa tra la risposta di due persone in una conversazione)
- STT: trascrizione dell'audio tra i 100- 400 ms
- LLM: tempo di elaborazione e generazione tra i 300 e gli 800 ms
- TTS tempo conveersione da testale a vocale tra i 100 e 300 ms
Risultato finale è fino ai 3 secondi di attesa per l'utente

Il secondo problema è la propagazione di errori nella fase di SST per esempio una parola chiave sbagliata può cambiare l'elaborazione dell'LLM

Il terzo problema è che alcune formattazioni come tabelle, blocchi di codice, ecc... sono compatibili sono con il formato testuale

#soluzione:
1- si utilizza la catena di montaggio: se viene inviato all'LLM un pezzo parziale (per esempio ogni volta che è stato identificato un punto), allora mentre LLM sta elaborando,
allora il TTS sta convertendo in audio il pezzettino per passo precedente

2- si può utilizzare una lista di parole chiave che possono essere confrontate con quelle ricevute dall'SST e fare delle eventuali correzzioni

3- adottare una soluzione mista tra l'utilizzo della voce e l'utilizzo del testo: in fase in ingresso per esempio posso scrivere il mio codice e utilizzare la voce per dire che è il mio codice,
analogamente in uscita è possibile produrre una tabella (o codice) mentre vocalmente si sente che è stato prodotto quanto richiesto

#2) i turni di parola
#utilizzo gemini, ho scritto nel promt:
#Come eviti che l'utente debba premere un pulsante per iniziare e finire di parlare? Come fa l'agente a capire che l'utente ha finito?

#soluzione:
Per eliminare il pulsante, si utilizzano due livelli di comprensione se la persona sta parlando:
1) utilizzare una detection della voce e se non si "sente parlare" si attiva un timer
2) si utilizza un controllo semantico: se la frase non è finita e c'è una pausa che ha fatto scattere il timer, allora avviene anche un controllo semantico e una frase non è completa,
allora vuol dire che la persona deve terminare la frase nonostante la detection della voce e poi il timer pensino che la persona abbai finito
(quindi utilizziamo un and logico di queste due situazioni)
