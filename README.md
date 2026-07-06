# GesPro

App per creare prescrizioni odontotecniche: anagrafica paziente, schema dentale (FDI/ISO), selezione dispositivo dal listino, stampa/PDF locale.

Tecnologie: React + Firebase (Authentication + Firestore), un unico file `index.html`, nessun build tool richiesto.

## 1. Pubblicare su GitHub Pages

1. Crea un nuovo repository su github.com, chiamalo `gespro`.
2. Carica il file `index.html` (e questo `README.md`) nel repository (pulsante "Add file" → "Upload files").
3. Vai su **Settings → Pages**.
4. In "Branch" seleziona `main` e cartella `/ (root)`, poi **Save**.
5. Dopo 1-2 minuti l'app sarà online su `https://TUO-USERNAME.github.io/gespro/`.

## 2. Autorizzare il dominio in Firebase

Perché il login funzioni sul dominio pubblico:

1. Console Firebase → **Authentication → Settings → Authorized domains**.
2. Clicca **Add domain** e inserisci `TUO-USERNAME.github.io`.

## 3. Sistemare le regole di sicurezza di Firestore (importante prima dell'uso reale)

Ora il database è in "modalità di test" (accesso libero, scade dopo 30 giorni). Prima di usarlo con pazienti veri, vai su **Firestore Database → Regole** e incolla:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /users/{userId}/{document=**} {
      allow read, write: if request.auth != null && request.auth.uid == userId;
    }
  }
}
```

Questo garantisce che ogni utente veda e modifichi solo le proprie prescrizioni.

## 4. Personalizzare il listino

Nel file `index.html`, cerca il blocco `PRICE_LIST` (in cima allo script) e sostituisci categorie/dispositivi/prezzi con quelli reali del laboratorio. Struttura:

```js
const PRICE_LIST = [
  { category: "Nome categoria", items: [
    { name: "Nome dispositivo", price: 123 },
  ]},
];
```

## 5. Uso quotidiano

- **Nuova prescrizione**: compila paziente, clicca i denti coinvolti sullo schema, aggiungi uno o più dispositivi dal listino, poi "Salva prescrizione" e/o "Stampa / Salva PDF" (quest'ultimo apre la finestra di stampa del browser: puoi stampare su carta o salvare come PDF).
- **Elenco prescrizioni**: mostra lo storico salvato su Firestore, visibile solo all'utente che le ha create.
