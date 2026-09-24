# SecureNotes

SecureNotes è un'applicazione nativa per macOS dedicata alla scrittura di note private e cifrate.

È sviluppata in Swift e SwiftUI, funziona completamente in locale e non richiede account, servizi cloud, server esterni o dipendenze di terze parti.

Le note vengono archiviate in un vault cifrato e possono essere visualizzate solamente dopo aver inserito la password principale corretta.

> SecureNotes è pensata principalmente per uso personale, educativo e sperimentale.

## Features

- Applicazione nativa macOS
- Interfaccia SwiftUI
- Vault protetto da password
- Note completamente cifrate
- Titoli e contenuti non salvati in chiaro
- Salvataggio automatico
- Ricerca nelle note dopo lo sblocco
- Creazione, modifica ed eliminazione delle note
- Blocco manuale del vault
- Blocco automatico dopo inattività
- Blocco durante sleep o blocco della sessione macOS
- Cambio della password principale
- Esportazione di backup cifrati
- Importazione sicura dei backup
- Supporto Light Mode e Dark Mode
- Supporto Reduce Motion
- Animazione personalizzata di avvio
- Funzionamento completamente offline
- Nessun account richiesto
- Nessuna telemetria
- Nessun servizio online

## Sicurezza

SecureNotes è progettata affinché la password principale e il contenuto delle note non vengano mai salvati sul disco in chiaro.

### Password e derivazione della chiave

La password principale non viene salvata.

Ad ogni sblocco, SecureNotes utilizza CommonCrypto per derivare una chiave AES-256 tramite:

- PBKDF2
- HMAC-SHA256
- salt casuale da 16 byte
- 600.000 iterazioni

La password non viene mai utilizzata direttamente come chiave di cifratura.

Il salt e i parametri necessari alla derivazione della chiave possono essere salvati insieme al vault, poiché non costituiscono informazioni segrete.

### Crittografia

Il contenuto del vault viene cifrato utilizzando:

`AES-256-GCM`

tramite CryptoKit.

AES-GCM fornisce sia:

- confidenzialità;
- autenticità e integrità dei dati.

Ogni salvataggio utilizza un nonce differente.

Anche informazioni come:

- versione del formato;
- algoritmo KDF;
- numero di iterazioni;
- salt;

vengono associate crittograficamente ai dati tramite Additional Authenticated Data.

Se il ciphertext o il tag di autenticazione vengono modificati, SecureNotes rifiuta il vault.

### Storage

Il vault principale viene salvato in:

```text
~/Library/Application Support/SecureNotes/vault.dat
```

La directory viene creata con permessi:

```text
0700
```

mentre il vault utilizza:

```text
0600
```

Il contenuto delle note non viene salvato in file separati in chiaro.

### Scrittura sicura del vault

SecureNotes evita di sovrascrivere direttamente il vault esistente.

Durante il salvataggio:

1. viene creato un file temporaneo nella stessa directory;
2. il nuovo vault viene scritto;
3. il file viene sincronizzato sul disco;
4. viene riletto;
5. viene verificata l'autenticazione AES-GCM;
6. solo dopo la verifica sostituisce il vault precedente tramite un'operazione atomica.

Lo stesso principio viene utilizzato per:

- cambio della password;
- importazione dei backup;
- aggiornamenti del formato del vault.

Questo riduce il rischio di perdere completamente le note in caso di crash o errore durante una scrittura.

## Gestione della memoria

La password e la chiave derivata non vengono salvate sul disco.

Mentre il vault è sbloccato, la chiave rimane disponibile solamente in memoria.

Quando il vault viene bloccato:

- le note vengono rimosse dalla UI;
- il contenuto decifrato viene eliminato dallo stato dell'app;
- vengono rimossi i riferimenti alla chiave crittografica.

Swift e CryptoKit non garantiscono tuttavia la cancellazione fisica immediata di ogni possibile copia temporanea dei dati dalla memoria del processo.

SecureNotes non pretende quindi di offrire protezione contro un attaccante con pieno accesso alla memoria del processo mentre il vault è già sbloccato.

## Blocco automatico

SecureNotes può bloccare automaticamente il vault.

Il timeout predefinito è:

```text
5 minuti
```

ed è configurabile nelle impostazioni.

L'app tenta inoltre di bloccare il vault quando:

- il Mac entra in stop;
- macOS blocca la sessione;
- l'app rimane in background oltre il timeout configurato;
- l'utente preme manualmente il pulsante di blocco.

## Password errata e vault modificato

AES-GCM non consente di distinguere in modo affidabile tra:

- password errata;
- chiave errata;
- ciphertext modificato;
- tag di autenticazione non valido.

Per questo motivo, in questi casi SecureNotes rifiuta l'accesso al vault senza mostrare contenuti parzialmente decifrati.

L'interfaccia può mostrare semplicemente:

```text
Password non corretta.
```

File malformati, versioni incompatibili del formato o strutture del vault non valide vengono invece gestiti separatamente.

## Backup

SecureNotes può esportare una copia cifrata del vault.

Il backup:

- non contiene note in chiaro;
- mantiene la cifratura del vault;
- può essere importato successivamente nell'app.

Durante l'importazione, SecureNotes verifica il backup prima di sostituire il vault esistente.

La sostituzione viene eseguita solamente dopo che autenticazione e decifratura sono andate a buon fine.

## Cambio password

La password principale può essere modificata dalle impostazioni.

Durante l'operazione:

1. viene verificata la password attuale;
2. viene richiesta la nuova password;
3. viene generato un nuovo salt;
4. viene derivata una nuova chiave;
5. viene creato un nuovo vault cifrato;
6. il nuovo vault viene verificato;
7. solamente dopo la verifica sostituisce quello precedente.

Il vault originale non viene quindi distrutto prima della creazione corretta del nuovo file.

## Struttura del progetto

```text
SecureNotes/
├── SecureNotes.xcodeproj/
│   ├── project.pbxproj
│   └── xcshareddata/
│       └── xcschemes/
│           └── SecureNotes.xcscheme
│
├── Sources/
│   ├── SecureNotesApp.swift
│   │
│   ├── Models/
│   │   └── Note.swift
│   │
│   ├── Security/
│   │   ├── KeyDerivation.swift
│   │   └── CryptoVault.swift
│   │
│   ├── Storage/
│   │   └── VaultStorage.swift
│   │
│   ├── Services/
│   │   └── AutoLockManager.swift
│   │
│   ├── State/
│   │   └── VaultController.swift
│   │
│   └── Views/
│       ├── RootView.swift
│       ├── BootAnimationView.swift
│       ├── SetupPasswordView.swift
│       ├── UnlockView.swift
│       ├── NotesView.swift
│       ├── SettingsView.swift
│       ├── ChangePasswordView.swift
│       └── ImportBackupView.swift
│
├── Assets.xcassets/
└── README.md
```

## Requisiti

- macOS 14 o successivo
- Mac Intel o Apple Silicon
- Xcode compatibile con Swift 5
- Nessuna dipendenza esterna

SecureNotes utilizza esclusivamente framework Apple:

- SwiftUI
- AppKit
- Foundation
- Combine
- CryptoKit
- CommonCrypto
- Security
- UniformTypeIdentifiers

Non sono necessari Swift Package Manager, CocoaPods o altri package esterni.

## Compilazione con Xcode

Clona la repository:

```sh
git clone https://github.com/YOUR_USERNAME/SecureNotes.git
```

Entra nella directory:

```sh
cd SecureNotes
```

Apri:

```text
SecureNotes.xcodeproj
```

con Xcode.

Poi:

1. seleziona lo schema `SecureNotes`;
2. seleziona `My Mac` come destinazione;
3. scegli la configurazione Release;
4. esegui:

```text
Product > Build
```

Per trovare l'app compilata puoi usare:

```text
Product > Show Build Folder in Finder
```

e quindi aprire:

```text
Build/Products/Release/SecureNotes.app
```

Durante lo sviluppo puoi invece trovare:

```text
Build/Products/Debug/SecureNotes.app
```

## Build da Terminale

È possibile creare una build Release anche tramite `xcodebuild`:

```sh
xcodebuild \
  -project SecureNotes.xcodeproj \
  -scheme SecureNotes \
  -configuration Release \
  -derivedDataPath build \
  build
```

Al termine, l'app compilata sarà disponibile in:

```text
build/Build/Products/Release/SecureNotes.app
```

## Installazione

Dopo la compilazione, trascina:

```text
SecureNotes.app
```

nella cartella:

```text
/Applications
```

oppure nella cartella Applicazioni del tuo utente.

Da quel momento SecureNotes può essere aperta normalmente tramite:

- Finder;
- Spotlight;
- Launchpad.

Non è necessario mantenere Xcode o il codice sorgente per utilizzare l'app compilata.

## Primo avvio

Al primo avvio SecureNotes chiede di creare una password principale.

La password deve avere almeno:

```text
12 caratteri
```

La password non viene salvata e non esiste un sistema di recupero.

> Se dimentichi la password principale, il contenuto del vault non può essere recuperato dall'app.

Conserva quindi la password in un luogo sicuro.

## Firma e Gatekeeper

SecureNotes è pensata principalmente per l'utilizzo locale.

Le build possono utilizzare una firma ad hoc o `Sign to Run Locally` e non necessitano di pubblicazione sul Mac App Store.

L'app non è necessariamente notarizzata da Apple.

Una build creata direttamente sul proprio Mac normalmente può essere eseguita senza particolari problemi.

Se macOS blocca l'apertura perché l'app proviene da uno sviluppatore non identificato:

1. apri Finder;
2. fai Control-clic su `SecureNotes.app`;
3. seleziona `Apri`;
4. conferma nuovamente `Apri`.

Non è necessario disattivare Gatekeeper.

## Privacy

SecureNotes è progettata per funzionare interamente offline.

L'app non richiede:

- account;
- login;
- server;
- cloud;
- analytics;
- advertising;
- telemetria;
- connessione Internet.

Le note rimangono sul dispositivo dell'utente salvo esportazione manuale di un backup.

## Limitazioni di sicurezza

SecureNotes utilizza primitive crittografiche moderne, ma non deve essere considerata automaticamente equivalente a software sottoposto ad audit professionale.

Il progetto non ha necessariamente ricevuto:

- audit crittografici indipendenti;
- penetration test professionali;
- certificazioni di sicurezza.

Non utilizzare SecureNotes come unico sistema di conservazione per informazioni la cui perdita avrebbe conseguenze critiche.

Mantieni sempre backup cifrati aggiornati.

## Roadmap

Possibili sviluppi futuri:

- cartelle per organizzare le note;
- tag;
- note preferite;
- editor Markdown;
- allegati cifrati;
- esportazione selettiva;
- protezione aggiuntiva tramite Touch ID;
- Secure Enclave;
- import/export migliorato;
- versioning delle note;
- ricerca più avanzata.

## Contributing

Issue e pull request sono benvenute.

Prima di inviare modifiche relative alla crittografia o al formato del vault, verifica attentamente che non introducano regressioni nella sicurezza o incompatibilità con vault esistenti.

## License

SecureNotes è distribuito sotto licenza MIT.

Consulta il file:

```text
LICENSE
```

per maggiori informazioni.

---

Built with Swift, SwiftUI and CryptoKit for macOS.
