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

## 11. Catalogo dispositivi (senza prezzo)

Ogni voce ha categoria, codice prodotto, descrizione — nessun prezzo, mai. Gestione da **Amministrazione → Catalogo**. Questo è il catalogo **generico**, usato da chi non ha un listino dedicato assegnato (vedi punto successivo).

## 12. Listini dedicati per cliente (con import da Excel, prezzo e stampa)

Da **Amministrazione → Listini** puoi creare listini specifici — pensati per clienti/laboratori che devono vedere solo un sottoinsieme di lavorazioni invece del catalogo generico.

- **Creazione**: ogni listino ha un **nome** (es. "Studio Rossi") e un **codice**. Se quel codice coincide con il **codice utente** di una persona (impostato quando l'admin crea/modifica il suo account), quella persona vedrà automaticamente le lavorazioni di questo listino al posto del catalogo generico in "Nuova prescrizione" — nessuna assegnazione manuale ulteriore da fare.
- **Popolare un listino manualmente**: da "Gestisci lavorazioni" del listino, stesso modulo del catalogo generico (categoria, codice, descrizione) più un campo **Prezzo** opzionale.
- **Popolare un listino da Excel**: nello stesso pannello, carica un file `.xlsx` con 4 colonne nell'ordine **Categoria, Codice, Descrizione, Prezzo** (il prezzo è opzionale, puoi lasciare la colonna vuota) e la prima riga di intestazioni (verrà ignorata). Le righe vengono aggiunte a quelle già presenti. Fogli `.xls` e `.csv` con la stessa struttura funzionano allo stesso modo.
- **Prezzo — solo lato amministrazione**: il prezzo si vede e si modifica (anche inline, riga per riga) solo qui, nel pannello admin. **Non compare mai** nell'interfaccia dell'utente normale che compila una prescrizione (né nella selezione dispositivo, né nella prescrizione salvata, né in stampa) — è pensato solo per uso interno e per la stampa del listino stesso.

  **Nota tecnica onesta**: come già per il vecchio listino con prezzi, questa protezione è a livello di interfaccia. Firestore non permette di nascondere singoli campi lato server nelle regole di sicurezza, quindi un utente molto esperto che ispezionasse il traffico di rete del browser potrebbe intercettare il prezzo grezzo insieme al resto dei dati della lavorazione. Per un uso interno tra colleghi di fiducia è comunque un buon livello di protezione; per una riservatezza totale garantita servirebbe una Cloud Function.
- **Stampa listino**: pulsante "Stampa listino" in alto nella pagina del listino — genera un PDF/foglio stampabile con logo e dati del cliente (se il suo profilo è compilato), lavorazioni raggruppate per categoria, con relativo prezzo. Utile da consegnare al cliente su richiesta. Il documento non ha limite di una pagina: se il listino è lungo, prosegue su più pagine.
- **Se un utente non ha un listino corrispondente** (o il suo codice utente è vuoto, o il listino trovato non ha lavorazioni), l'app usa automaticamente il catalogo generico come prima.
- **Eliminare un listino** cancella anche tutte le sue lavorazioni.

## 13. Uso quotidiano (utente normale)

- **Nuova prescrizione**: compila tutte le sezioni del modulo, poi "Salva prescrizione" (assegna il codice univoco) e/o "Stampa / Salva PDF".
- **Elenco prescrizioni**: storico delle proprie prescrizioni con codice.
- **Cambia password**: in alto.
