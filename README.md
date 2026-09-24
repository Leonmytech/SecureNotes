# SecureNotes

Applicazione nativa SwiftUI per macOS, pensata per l'uso locale. Non richiede servizi online né dipendenze esterne.

## Architettura e sicurezza

- `VaultController` gestisce la sessione sbloccata, le note in memoria, la ricerca e le operazioni sul vault.
- `VaultStorage` conserva un solo file cifrato in `~/Library/Application Support/SecureNotes/vault.dat`. La directory ha permessi `0700` e il file `0600`.
- La password non viene salvata. A ogni sblocco, CommonCrypto deriva una chiave AES-256 con PBKDF2-HMAC-SHA256, salt casuale di 16 byte e 600.000 iterazioni. La password non viene usata direttamente come chiave.
- CryptoKit cifra l'intero contenuto con AES-256-GCM. Ogni salvataggio usa un nuovo nonce; formato, algoritmo KDF, iterazioni e salt sono autenticati insieme ai dati. Un tag non valido impedisce di visualizzare le note.
- La password e la chiave non sono salvate su disco. Al blocco, l'app svuota le note dalla UI e rimuove i riferimenti alla chiave; Swift/CryptoKit non garantiscono la cancellazione fisica di ogni copia temporanea dalla memoria del processo.
- Ogni scrittura usa un file temporaneo nella stessa directory, lo sincronizza, lo rilegge e lo autentica; poi lo installa con una rinomina atomica. Anche il cambio password e l'importazione verificano i dati prima di sostituire il vault esistente.
- Il backup esportato è una copia cifrata del vault. L'importazione richiede la password del backup e sostituisce il vault corrente solo dopo l'autenticazione.
- L'app tenta il blocco automatico dopo 5 minuti di inattività, configurabile nelle impostazioni, quando passa in background per il periodo scelto, quando il Mac va in stop e quando riceve la notifica di blocco sessione.

Il fallimento dell'autenticazione AES-GCM non permette di distinguere una password sbagliata da un ciphertext/tag manomesso: in entrambi i casi l'app rifiuta i dati e mostra “Password non corretta.”. File malformati o formati incompatibili ricevono un messaggio distinto.

## Struttura del progetto

```text
SecureNotes/
├── SecureNotes.xcodeproj/
│   ├── project.pbxproj
│   └── xcshareddata/xcschemes/SecureNotes.xcscheme
├── Sources/
│   ├── SecureNotesApp.swift
│   ├── Models/Note.swift
│   ├── Security/KeyDerivation.swift
│   ├── Security/CryptoVault.swift
│   ├── Storage/VaultStorage.swift
│   ├── Services/AutoLockManager.swift
│   ├── State/VaultController.swift
│   └── Views/
│       ├── RootView.swift
│       ├── SetupPasswordView.swift
│       ├── UnlockView.swift
│       ├── NotesView.swift
│       ├── SettingsView.swift
│       ├── ChangePasswordView.swift
│       └── ImportBackupView.swift
└── README.md
```

## Aprirlo e compilare con Xcode

1. Apri `SecureNotes.xcodeproj` con Xcode.
2. Seleziona lo schema **SecureNotes** e la destinazione **My Mac**.
3. Seleziona **Product > Build**. Il progetto usa Swift 5, macOS 14 come versione minima e genera automaticamente `Info.plist`.
4. In Xcode, scegli **Product > Show Build Folder in Finder** e apri `Build/Products/Release/SecureNotes.app` (oppure `Debug/SecureNotes.app` se hai compilato Debug).

Gli import usano solo framework Apple: SwiftUI, AppKit, Combine, CryptoKit, CommonCrypto, Security, Foundation e UniformTypeIdentifiers. Non occorre aggiungere pacchetti o collegare framework manualmente.

Questa copia del progetto è già stata compilata con Xcode. Il bundle prodotto si trova in:

```text
build/Build/Products/Release/SecureNotes.app
```

Puoi anche compilare da Terminale, se preferisci:

```sh
xcodebuild -project SecureNotes.xcodeproj -scheme SecureNotes -configuration Release -derivedDataPath build build
```

## Copiarla in Applicazioni e avviarla

Nel Finder, trascina `SecureNotes.app` da `Build/Products/Release` nella cartella **Applicazioni**. In alternativa, copiala nella cartella `Applicazioni` del tuo utente. Quindi aprila con doppio clic. Al primo avvio scegli una password principale di almeno 12 caratteri e confermala: SecureNotes crea il vault cifrato e non potrà recuperare la password se la dimentichi.

Il progetto applica una firma **ad hoc** locale e non è notarizzato, perché è destinato al solo uso personale. Una build creata direttamente sul Mac di solito si apre senza avvisi. Se macOS mostra l'avviso per uno sviluppatore non identificato, nel Finder fai Control-clic su `SecureNotes.app`, scegli **Apri**, poi conferma **Apri** nella finestra di dialogo. Non è necessario disattivare Gatekeeper.
