# GesPro

App per creare prescrizioni odontotecniche complete: anamnesi, dati paziente, schema dentale con icone per tipo di dente, descrizione dispositivo, materiali, impronte, colore, codice univoco progressivo, stampa/PDF a due colonne (scheda + modulo prove). Pannello di amministrazione con gestione utenti, catalogo e prescrizioni.

Tecnologie: React + Firebase (Authentication + Firestore), un unico file `index.html`, nessun build tool richiesto.

## 1. Pubblicare su GitHub Pages

1. Repository GitHub `gespro`, **pubblico**.
2. Carica `index.html` e questo `README.md`.
3. Settings → Pages → Branch `main`, cartella `/ (root)` → Save.
4. Online dopo 1-2 minuti su `https://TUO-USERNAME.github.io/gespro/`.

## 2. Autorizzare il dominio in Firebase

Console Firebase → **Authentication → Settings → Authorized domains** → **Add domain** → `TUO-USERNAME.github.io`.

## 3. Regole di sicurezza di Firestore

Console Firebase → **Firestore Database → Regole** → incolla e pubblica (**è cambiata**: aggiunta la collezione `counters` per i codici progressivi):

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    function isAdmin() {
      return request.auth != null &&
        get(/databases/$(database)/documents/users/$(request.auth.uid)).data.role == 'admin';
    }

    match /users/{userId} {
      allow read: if request.auth != null && (request.auth.uid == userId || isAdmin());
      allow create: if request.auth != null && (
        (request.auth.uid == userId && request.resource.data.role == 'user') ||
        isAdmin()
      );
      allow update: if isAdmin();
    }

    match /catalog/{itemId} {
      allow read: if request.auth != null;
      allow write: if isAdmin();
    }

    match /prescriptions/{prescriptionId} {
      allow read: if request.auth != null && (resource.data.ownerId == request.auth.uid || isAdmin());
      allow create: if request.auth != null && (request.resource.data.ownerId == request.auth.uid || isAdmin());
      allow update, delete: if request.auth != null && (resource.data.ownerId == request.auth.uid || isAdmin());
    }

    match /counters/{counterId} {
      allow read, write: if request.auth != null;
    }

    match /listini/{listinoId} {
      allow read: if request.auth != null;
      allow write: if isAdmin();
      match /lavorazioni/{itemId} {
        allow read: if request.auth != null;
        allow write: if isAdmin();
      }
    }

    match /settings/{docId} {
      allow read: if request.auth != null;
      allow write: if isAdmin();
    }
  }
}
```

## 4. Codice univoco prescrizione

Ogni prescrizione riceve automaticamente al salvataggio un codice progressivo tipo `RX-2026-0001` (anno + numero progressivo), generato con un contatore transazionale su Firestore — quindi garantito senza duplicati anche se più utenti salvano nello stesso istante.

## 5. Modulo prescrizione completo

Oltre ai dati paziente e allo schema dentale, il form ora comprende:

- **Anamnesi**: "Nulla da segnalare" selezionato di default; selezionando una o più voci specifiche (allergie, disfunzioni articolari, altri dispositivi, handicap psicomotori, bruxismo, malattie infettive, altro) compare un campo di testo libero per i dettagli.
- **Altre informazioni**: età, sesso, forma del viso (quadrato/tondo/triangolare, con icone).
- **Descrizione del dispositivo**: testo libero oltre alla selezione dal catalogo.
- **Materiale da utilizzare**: menu con le opzioni comuni (Lega Co-Cr, Zirconia, Resina, Ceramica, Titanio, Metallo-ceramica) o "Altro" con testo libero.
- **Materiale allegato**: selezione multipla (radiografie, cere, modelli provvisori, fotografie, siliconi, registrazione pantografica, modelli già sviluppati, resine, arco facciale, registrazioni occlusali).
- **Impronte**: giorno di rilevazione, disinfettante utilizzato, materiale.
- **Colore**: colore e campionario di riferimento.

## 6. Schema dentale con icone realistiche

Le icone cambiano forma in base al tipo di dente secondo la numerazione FDI/ISO: incisivo, canino, premolare, molare — con legenda sotto lo schema.

## 7. Stampa PDF a due colonne, con intestazione del cliente

Cliccando "Stampa / Salva PDF": in alto compare l'**intestazione del cliente** (di chi possiede la prescrizione — sé stesso, oppure l'utente selezionato in "Per conto di") con logo (se caricato), nominativo/ragione sociale, indirizzo, P.IVA e telefono, presi dal suo profilo. Sotto, la colonna sinistra contiene tutte le informazioni della prescrizione (anamnesi, paziente, denti, dispositivo, materiali, impronte, colore); la colonna destra contiene un **modulo delle prove** (tabella con 4 righe vuote: data prova, esito/note, firma) da compilare a mano durante gli appuntamenti successivi.

Questi dati vengono "fotografati" nella prescrizione al momento del salvataggio (non recuperati di nuovo in stampa), così restano coerenti anche se il profilo dell'utente viene modificato in seguito. Le prescrizioni salvate **prima** di questo aggiornamento non hanno questi dati: l'intestazione risulterà vuota per quelle.

Il logo si carica dal profilo utente (Amministrazione → Utenti → crea o modifica un utente → "Dati per intestazione di stampa"): viene salvato come immagine incorporata nel database, quindi conviene usare un file piccolo (sotto ai 700 KB, l'app blocca file più pesanti).

## 8. Gestione utenti, anagrafica e password (pannello Admin → Utenti)

- **Creazione**: l'admin inserisce anagrafica (persona fisica: nome/cognome, oppure azienda/studio: ragione sociale), indirizzo, P.IVA, telefono, logo opzionale, email di accesso, **codice utente** (obbligatorio — sarà usato in futuro per abbinare un listino/catalogo specifico a ciascun utente) e ruolo. L'app genera una password temporanea mostrata una sola volta, da comunicare alla persona.
- **Modifica**: pulsante "Modifica" su ogni riga della tabella utenti — permette di correggere in qualsiasi momento nominativo, ragione sociale, codice utente e tutti i dati dell'intestazione di stampa (indirizzo, P.IVA, telefono, logo).
- **Reset password per utenti esistenti**: pulsante "Invia email reset password" — invia l'email standard di Firebase con un link che permette alla persona di scegliere una nuova password.
- **Email**: è anche la credenziale di accesso — non è modificabile dall'app né dall'admin né dall'utente. Per cambiarla, se necessario, va fatto dalla console Firebase (Authentication → Users → seleziona l'utente → modifica l'email), aggiornando poi a mano anche il campo `email` nel documento corrispondente su Firestore (collezione `users`).

### Diventare il primo amministratore (passaggio manuale, una tantum)

1. Console Firebase → **Authentication → Users → Add user** → email + password.
2. Copia lo **User UID** generato.
3. Console Firebase → **Firestore Database → Dati** → crea/apri la raccolta `users` → **Aggiungi documento** → come ID documento incolla lo UID.
4. Campi: `email` (string), `role` (string, `admin`), `createdAt` (timestamp, ora).
5. Login nell'app con quell'account.

## 9. Compilare una prescrizione per conto di un utente (senza impersonificazione)

Quando l'admin apre "Nuova prescrizione" vede in cima un riquadro **"Per conto di"**, con un menu a tendina per scegliere un utente registrato invece di "Me stesso". La prescrizione viene salvata con quell'utente come proprietario: comparirà nel suo elenco personale, non in quello dell'admin, esattamente come se l'avesse compilata lui.

**Nota tecnica**: non è un vero login come quell'utente (impersonificazione) — quella richiederebbe un backend (Cloud Functions) per generare un token di accesso a suo nome, cosa che qui non serve: basta assegnare correttamente il proprietario della prescrizione al salvataggio. Il campo `createdBy` nel database tiene comunque traccia di chi l'ha materialmente compilata (l'admin), per trasparenza.

## 10. Gestione prescrizioni (pannello Admin → Tutte le prescrizioni)

L'admin vede tutte le prescrizioni di tutti gli utenti, può aprire il **Dettaglio** di ognuna (anamnesi, materiali, impronte, colore, ecc.) e **eliminarle**. La modifica dei singoli campi di una prescrizione già salvata non è ancora implementata: se ti serve, dimmelo e la aggiungo.

## 11. Catalogo dispositivi (senza prezzo, con nota e immagine)

Ogni voce ha categoria, codice prodotto, descrizione — nessun prezzo, mai. Gestione da **Amministrazione → Catalogo**. Questo è il catalogo **generico** (usato da chi non ha un listino dedicato assegnato) ed è anche la fonte di verità per i codici: **il codice deve essere univoco** — l'app blocca l'aggiunta di un codice già esistente.

Ogni voce può avere anche una **nota esplicativa** (testo libero: indicazioni cliniche, quando usarlo, differenze rispetto a dispositivi simili) e un'**immagine** (facoltativa, sotto ~700 KB) per aiutare il clinico a scegliere il dispositivo giusto. Quando l'utente seleziona un dispositivo in "Nuova prescrizione", se sono presenti nota e/o immagine compaiono automaticamente sotto il menu di selezione, prima di aggiungerlo alla prescrizione. Nota e immagine si possono modificare in qualsiasi momento con "Modifica" nella tabella del catalogo (categoria, codice e descrizione restano invece fissi dopo la creazione, per non disallineare i listini che li referenziano).

**Importare il catalogo completo da Excel**: carica un file `.xlsx` con le colonne, in quest'ordine — **Categoria, Codice, Descrizione, Nota** (Nota è opzionale) — e la prima riga di intestazioni (verrà ignorata). Prima di scrivere qualsiasi cosa nel database, l'app mostra un'**anteprima da confermare**: quante voci sono nuove (verranno aggiunte) e quante corrispondono a codici già esistenti (verranno **sovrascritte** — con un confronto "prima/dopo" per ogni voce che cambierà). Solo cliccando "Conferma e importa" le modifiche vengono applicate; "Annulla" scarta tutto senza toccare il database. Righe con codice duplicato ripetuto nello stesso file vengono scartate e conteggiate a parte. La sovrascrittura aggiorna categoria e descrizione; la nota viene aggiornata solo se il file ne contiene una per quella riga (altrimenti resta quella già presente); l'immagine non è mai toccata dall'import (va caricata singolarmente). Fogli `.xls` e `.csv` con la stessa struttura funzionano allo stesso modo.

**Esportare il catalogo**: pulsante "Esporta catalogo (Excel)" accanto all'importazione — scarica un file `.xlsx` con le stesse colonne usate per l'import (Categoria, Codice, Descrizione, Nota), pronto per essere modificato e ricaricato. Comodo per il ciclo "esporta → modifica in blocco fuori dall'app → reimporta e sovrascrivi".

Queste due informazioni vengono copiate automaticamente anche quando una lavorazione viene aggiunta a un listino personale (manualmente o da Excel), così restano visibili al clinico anche se sta usando un listino dedicato invece del catalogo generico.

## 12. Listini dedicati per cliente (con import da Excel, prezzo e stampa)

Da **Amministrazione → Listini** puoi creare listini specifici — pensati per clienti/laboratori che devono vedere solo un sottoinsieme di lavorazioni invece del catalogo generico.

- **Il listino si crea da solo**: quando l'admin crea un utente (o gli assegna/modifica un codice utente da "Modifica"), l'app crea automaticamente un listino con quel codice e il nome ricavato dall'anagrafica (nominativo o ragione sociale) — **non serve più crearlo a mano**. Se un listino con quel codice esiste già, non ne viene creato uno doppio. Il modulo "Crea nuovo listino" in Amministrazione → Listini resta disponibile solo per casi particolari (es. un listino condiviso da più codici, o per ricrearne uno mancante).
- **Le lavorazioni di un listino sono legate al catalogo generico**: non puoi inventare una lavorazione nuova dentro un listino — il **codice interno deve corrispondere esattamente** a uno già presente nel catalogo generico. Categoria e descrizione interne vengono ereditate automaticamente da lì. Se il codice che ti serve non esiste ancora, vai prima ad aggiungerlo in Amministrazione → Catalogo.
- **Codice cliente e descrizione personalizzata**: per ogni lavorazione del listino puoi anche impostare un **codice cliente** (il codice che usa quello specifico cliente, abbinato al tuo codice interno) e una **descrizione personalizzata** (come la chiama lui). Se presenti, l'utente che usa quel listino li vede al posto dei tuoi riferimenti interni — sia nella selezione del dispositivo, sia nella prescrizione salvata, sia in stampa. Il codice cliente, se impostato, deve essere univoco all'interno dello stesso listino (non può ripetersi su due lavorazioni diverse). Sono entrambi opzionali: se non li imposti, l'app mostra semplicemente codice e descrizione interni.
- **Popolare un listino manualmente**: seleziona il codice interno (un suggerimento a tendina mostra i codici del catalogo con relativa descrizione), inserisci prezzo, codice cliente e descrizione personalizzata (questi ultimi due opzionali), "+ Aggiungi". Ogni lavorazione già inserita si può correggere in qualsiasi momento con "Modifica".
- **Popolare un listino da Excel**: carica un file `.xlsx` con le colonne, in quest'ordine — **Codice, Prezzo, Codice cliente, Descrizione personalizzata** (solo Codice è obbligatorio, le altre 3 sono opzionali) — e la prima riga di intestazioni (verrà ignorata). Ogni riga viene validata: se il codice non esiste nel catalogo generico, o è duplicato (codice interno o codice cliente già usati, nel listino o ripetuti nel file), viene scartata e conteggiata a parte nel messaggio di riepilogo — non blocca l'import delle righe valide. Fogli `.xls` e `.csv` con la stessa struttura funzionano allo stesso modo.
- **Esportare un listino**: pulsante "Esporta (Excel)" accanto a "Stampa listino" — scarica le lavorazioni del listino nello stesso formato usato per l'import (Codice, Prezzo, Codice cliente, Descrizione personalizzata), pronte per essere modificate e ricaricate.
- **Prezzo — solo lato amministrazione**: il prezzo si vede e si modifica (anche inline, riga per riga) solo qui, nel pannello admin. **Non compare mai** nell'interfaccia dell'utente normale che compila una prescrizione (né nella selezione dispositivo, né nella prescrizione salvata, né in stampa) — è pensato solo per uso interno e per la stampa del listino stesso.

  **Nota tecnica onesta**: come già per il vecchio listino con prezzi, questa protezione è a livello di interfaccia. Firestore non permette di nascondere singoli campi lato server nelle regole di sicurezza, quindi un utente molto esperto che ispezionasse il traffico di rete del browser potrebbe intercettare il prezzo grezzo insieme al resto dei dati della lavorazione. Per un uso interno tra colleghi di fiducia è comunque un buon livello di protezione; per una riservatezza totale garantita servirebbe una Cloud Function.
- **Stampa listino**: pulsante "Stampa listino" in alto nella pagina del listino — genera un PDF/foglio stampabile con logo e dati del cliente (se il suo profilo è compilato), lavorazioni raggruppate per categoria, con relativo prezzo. Utile da consegnare al cliente su richiesta. Il documento non ha limite di una pagina: se il listino è lungo, prosegue su più pagine.
- **Se un utente non ha un listino corrispondente** (o il suo codice utente è vuoto, o il listino trovato non ha lavorazioni), l'app usa automaticamente il catalogo generico come prima.
- **Eliminare un listino** cancella anche tutte le sue lavorazioni.

## 13. Modalità manutenzione

Da **Amministrazione → Impostazioni** puoi attivare/disattivare la modalità manutenzione, con un messaggio personalizzabile.

- **Quando è attiva**: ogni utente normale che apre l'app (o che ha già una sessione aperta e ricarica la pagina) vede una schermata di manutenzione al posto dell'app, con il messaggio che hai impostato (o uno generico se lo lasci vuoto). Può comunque disconnettersi.
- **Tu, come amministratore, non vieni mai bloccato**: continui a usare l'app normalmente (vedrai una scritta "Manutenzione attiva" accanto al logo, per ricordartelo), così puoi lavorare — es. aggiornare listini o correggere dati — e poi disattivarla quando hai finito.
- Ricorda di disattivarla al termine dell'intervento, altrimenti gli utenti restano bloccati.

## 14. Uso quotidiano (utente normale)

- **Nuova prescrizione**: compila tutte le sezioni del modulo, poi "Salva prescrizione" (assegna il codice univoco) e/o "Stampa / Salva PDF".
- **Elenco prescrizioni**: storico delle proprie prescrizioni con codice.
- **Cambia password**: in alto.
