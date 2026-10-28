# AI Engineer — Round tecnico

Hai **un'ora** a disposizione per fare ricerca e preparare una proposta di design. Non ti chiediamo di scrivere codice: ci interessa capire come ragioni, quali soluzioni conosci (o riesci a trovare) e come motivi le tue scelte.

## Da dove si parte: un agente testuale

Prendi come riferimento un classico agente con chiamate a tool:

```
utente ──testo──▶ modello ──▶ serve un tool?
                    ▲            │ sì                  │ no
                    │            ▼                     ▼
                    └── risultato del tool      risposta finale ──testo──▶ utente
```

1. L'utente scrive un messaggio.
2. Il modello riceve la conversazione e l'elenco dei tool a disposizione.
3. Decide se chiamare uno o più tool (una ricerca, una query al database, un'API…) oppure rispondere subito.
4. I risultati tornano al modello e il ciclo ricomincia, finché la risposta non è pronta.
5. Nel frattempo il modello produce anche ragionamenti e commenti intermedi ("adesso cerco…", "verifico i dati…"), che in una chat l'utente vede scorrere a schermo.

## Dove vogliamo arrivare: un agente con cui parlare

Vogliamo trasformare questo agente in un assistente vocale. I punti qui sotto sono **in ordine di priorità**: ciascuno parte dal precedente e pesa un po' meno, ma è comunque un valore aggiunto. Meglio affrontarne pochi e bene che tutti in superficie.

1. **La voce** — Come modificheresti il ciclo perché l'utente possa parlare e l'agente rispondere a voce? La soluzione più immediata è mettere uno speech-to-text all'ingresso e un text-to-speech all'uscita: cosa manca perché funzioni davvero?
2. **I turni di parola** — Come eviti che l'utente debba premere un pulsante per iniziare e finire di parlare? Come fa l'agente a capire che l'utente ha finito?
3. **Solo la risposta** — Mentre lavora, l'agente ragiona e commenta ciò che fa. Come fai in modo che l'utente senta soltanto la risposta finale, e non tutto il resto?
4. **Le interruzioni** — Se l'utente prende la parola mentre l'agente sta lavorando o rispondendo, come fa a fermarlo o a correggerne la rotta, come farebbe con una persona in ascolto?
5. **Le emozioni** — Come capisci lo stato d'animo dell'utente (ad esempio frustrato, confuso, di fretta)? E come lo usi?
6. **L'avatar** — Come daresti un volto all'agente?
7. **Labiale ed espressioni** — Come sincronizzi il labiale dell'avatar con la voce e gli dai espressioni coerenti con quello che dice?

## Come consegnare

Allo scadere dell'ora, aggiungi la tua proposta come file Markdown in:

    submissions/nome-cognome.md

(tutto minuscolo, senza accenti né spazi: ad esempio `mario-rossi.md`)

e apri una Pull Request verso questo repository, nel modo che preferisci:

- **Dal browser:** *Add file → Create new file*, scrivi il percorso, incolla il testo e clicca su *Propose changes*. GitHub crea il fork in automatico: a quel punto clicca su *Create pull request*.
- **Con git:** fai il fork, aggiungi il file con un commit, fai push e apri la Pull Request.

Fa fede l'orario di apertura della Pull Request.
