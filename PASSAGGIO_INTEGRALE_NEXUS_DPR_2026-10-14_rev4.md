# NEXUS: passaggio di consegne per la prosecuzione dello sviluppo DPR

**Revisione:** rev. 4 del 14 ottobre 2026, sera, alla consegna della **rc8**. Storia: rev. 1 dopo i test PostgreSQL rc6/rc7; rev. 2 verifica del codice; rev. 3 decisioni di Federico e piano rc8.

**Base del codice:** pacchetto completo `NEXUS_0.18.0_rc8_2026-10-14.zip`, SHA-256 `103266d53d7aca18802b5a5b197f8bf14632386a964418cc8adb705f72f48a40` (569 file; impronte dei singoli file in `SHA256SUMS.txt`). La copia di questo documento dentro lo ZIP non può contenere l'impronta dello ZIP stesso. Schema Alembic **0029**, invariato rispetto alla rc7.

**Stato:** collaudo. Non è una release per la produzione. In caso di divergenza vale il codice; prima di assumere che una funzione sia completa, verificare con Federico.

**Da consegnare alla prossima istanza, esattamente due allegati:**
1. lo ZIP integrale rc8;
2. l'ultima revisione di **questo documento**. Una copia è anche dentro lo ZIP; vale la più recente.

Non partire da copie parziali o patchate a mano, né dalla rc5 del PC di Federico. Non far lavorare due istanze sugli stessi file.

## 0. Regole permanenti

1. **Conferma preventiva** (Federico, 14/10/2026, 11:26): «prima di modificare codice o logiche esistenti chiedi sempre conferma, anche con esempi che mi aiutino a capire». Proporre ogni modifica con un esempio concreto prima/dopo, attendere il sì, poi scrivere.
2. **Niente funzioni nuove se non richieste** (Federico, 12:09, a proposito della correzione dei numeri): si preferiscono i campi e i flussi esistenti.
3. **R111.** Il motore VerbalizzAI 1.0.1 vendorizzato in `gestionale/terzi/verbalizzai/` non si modifica; `tests/test_verbalizzai_identico.py` lo verifica. Si integra solo dal ponte NEXUS.
4. **Generazione dei verbali e storico intoccabili** (Federico, 13:48, sul punto 4): nessuna modifica deve alterare come si generano i verbali o lo storico (verbali emessi, revisioni, fotografie). Ogni modifica che li sfiora va accompagnata da prove di non regressione.
5. **Appunti di collaudo ≠ ordine di sviluppo.** Quando Federico detta appunti «da mettere nel documento», si registrano e non si implementano finché non lo chiede (esempi: §§ 3.6 e 3.7).
6. **Onestà sulle prove.** Si dichiara sempre cosa è stato eseguito davvero, dove (sistema operativo, versione di PostgreSQL) e cosa no.

## 1. Obiettivo e vincoli approvati

NEXUS deve sostituire i tre gestionali della software house Garda Informatica: **ingecert / AD_dpr**, **adsrl / AD_dm** e **IEC / I&C**. AD_dpr e AD_dm sono due rami della stessa società, AD; I&C è una società distinta. Clienti, sedi e parte delle attrezzature sono comuni, ma le finanze di I&C restano separate. Gli utenti I&C vedono solo le informazioni pertinenti, e la vista AD_dm non deve mostrare riferimenti espliciti a I&C. L'audit riservato deve rimanere fedele.

NEXUS ha un backend e un archivio PostgreSQL comuni, con esperienze operative separate per attività. Clienti, sedi, preventivi, accettazioni, affidamenti, verifiche, riesami e fatturazione usano servizi comuni dove le regole coincidono, mantenendo ambito, titolarità, permessi e sezionali. Non creare una seconda logica commerciale DPR né una seconda anagrafica per attività.

**Sequenza operativa.** VerbalizzAI autonomo è in produzione su `verbalizzai.adsrl.net` con SQLite e riceve verbali fino al passaggio a NEXUS_DPR. Obiettivo: NEXUS_DPR operativo per *tutti* i nuovi incarichi DPR a gennaio 2027, al più tardi a marzo, con l'intero storico ingecert importato. Dopo il DPR, AD_dm e IEC sostituiranno i rispettivi gestionali Garda. Nessuna sincronizzazione automatica Garda ↔ NEXUS nella transizione: fermo e riconciliazione finali. Migrazione completa di dati e allegati come requisito del rilascio. **NEXUS girerà su un VPS Linux Ubuntu** (D2).

Il gestionale deve essere corretto **e** veloce. L'usabilità si prova per ruolo con dati realistici, annotando tempo, clic, informazioni riscritte, errori, passaggi fra pagine e momenti in cui l'azione successiva non è evidente.

### Orizzonte AD_dm / IEC da preservare

- AD dispone del sorgente PHP Garda e dei tre database MariaDB. L'importazione risolve duplicati di clienti e sedi con corrispondenze documentate, mai con fusioni cieche.
- IEC opera sulle stesse attrezzature di AD_dm: GVR (tarature/prove di efficienza PSV, CND) e SC/SP (indagini supplementari).
  - L'anagrafica dell'attrezzatura è condivisa.
  - Eventi di prova e report appartengono alla società che li produce.
  - AD_dm vede ultimo e prossimo CND, numero e PDF del report senza identificare IEC nella vista ordinaria.
  - Nessuna società corregge i report dell'altra.
  - Serve una prossima scadenza definita.
- Futuro ambito commerciale **EX** di AD_dm (PSV e CND eseguiti da AD), su modello IEC, senza duplicare le logiche. «Taratura» e «prova di efficienza» PSV sono sinonimi. Mai doppie esportazioni INAIL della stessa verifica (raddoppierebbero il contributo del 5%).

## 2. Stato tecnico al 14/10/2026 (rc8)

| Area | Stato | Limite |
|---|---|---|
| Backend | PostgreSQL per il database comune; sul PC di Federico PostgreSQL 18 con `nexus_locale` di collaudo. | Storico dei tre MariaDB non importato. |
| Verbali DPR incorporati | Schema `verbali` in PostgreSQL (rc4–rc7), adattatore del motore, importatore dei backup completi, blocco contro l'attivazione su archivio diverso. | `verbali_postgresql` parte da `false`: non attivarlo sui dati reali senza prova di importazione. |
| VerbalizzAI autonomo | In produzione su SQLite. | Il suo archivio cambia ogni giorno: una copia di oggi non è il taglio finale. |
| Interfaccia DPR | **rc8:** barra in alto; pagina del verbale dentro NEXUS con ritorno alla verifica; firme, destinazioni e dati societari in Configurazione. | Da provare con gli operatori (§ 4). |
| Storico Garda | Importatore da dump SQL (`moduli/garda/dump.py`), orientato in parte a SQLite. | Nessuna migrazione completa ingecert → PostgreSQL quadrata (fatture, collegamenti, allegati). |

### 2.1 Prove eseguite davvero

**Da Federico su Windows / PostgreSQL 18:**
- rc3: 35 test nativi passati; dump `nexus_locale_rc3_0028.dump` e backup applicativo verificati.
- rc5 + file rc6: `tests/postgresql/test_verbali_pg.py`, **5 passed** (55,93 s). Il primo tentativo aveva rivelato il difetto di `_righe()` su `cursor.description=None`, corretto nella rc6.
- File rc7 `tests/postgresql/test_dpr_api_pg.py`: **3 passed** (bozza → emissione; recupero del collegamento; riesame non conforme → revisione).
- La cartella del PC è una rc5 patchata a mano. La rc8 completa **non è ancora stata provata su Windows**.

**Da Claude in ambiente cloud Linux** (Python 3.13, librerie installate senza rete da `librerie_linux.zip` fornito da Federico, PostgreSQL **16** locale):

| Prova | Esito |
|---|---|
| Suite completa rc7, base di confronto | **421 passed**, 0 skipped, 9 min 38 s |
| Suite completa rc8 | **433 passed**, 0 skipped, 10 min 34 s |
| Controlli nel browser (Chromium headless) sull'ambiente di prova | barra, tendine, menu stretto, pagina del verbale, viste, campo numero, Firme, Destinazioni, Dati della società |

**Mai provato finora:** PostgreSQL 18 con la rc8, dati reali, importazione integrale dello storico, ripristino su server, invii email reali.

**Come riprodurre l'ambiente di sviluppo cloud.** PyPI non è raggiungibile per policy dell'organizzazione. Federico può generare le librerie Linux dal PC:

```
.venv\Scripts\python.exe -m pip download -r requirements-sviluppo.txt -d librerie_linux --platform manylinux2014_x86_64 --platform manylinux_2_28_x86_64 --python-version 3.13 --implementation cp --only-binary=:all:
```

poi si installano con `pip install --no-index --find-links`. PostgreSQL 16 è nell'immagine (`/usr/lib/postgresql/16/bin`): `initdb` + `pg_ctl` su una porta di prova, poi `NEXUS_TEST_POSTGRESQL_ADMIN_URL=postgresql://postgres@127.0.0.1:5433/postgres`.

## 3. Decisioni di dominio e rilievi, con stato

### 3.1 Scheda impianto e potenza (D1): realizzata la parte essenziale nella rc8

**R122** (11/10/2026), testo letterale:

> «una sola scheda tecnica per l'impianto. diciamo che tutti i campi del verbale (ad esclusione di "positivo" / "negativo", durata verifica, persona presente in occasione della verifica, resistenza di terra, sono attributi dell'impianto. il verbale precompilato di verbalizzAI ripropone l'ultimo aggiornamento.»

Restano solo nel verbale i dati di ogni singola verifica: numero, date, motivo, strumenti, note, non conformità, valori misurati. Il verbale emesso conserva sempre la sua fotografia.

**Decisioni del 14/10/2026 (testo letterale):**
- **D1** (11:26): «nexus, però considera che ogni "impianto" dovrà avere gli attributi di cui hai letto, parte dei quali riproposti nel verbale.»
- **Precisazione** (11:30): «potenza è attributo dell'impianto, e determina la tariffa applicabile a preventivo e quindi a fattura, oltre che verbale.»
- **D1-bis** (11:32): «la fattura segue SEMPRE preventivo accettato / ordine. se il verificatore si accorge che la potenza è diversa occorre emettere revisione preventivo».
- **D1-quater** (11:37): «se l'idea è sostituire verbalizzAI dentro nexus allora gli attributi si importano in questo. n.b. quando sarà pienamente integrato non caricheremo più dei verbali in word per avere la scheda impianto popolata, ma avremo già gli attributi migrati da verbalizzAI. vorrei fosse chiaro questo punto».
- **Elementi** (11:41): «si, anche sulla scheda impianto, ma praticamente la bozza di verifica riproporrà il "parco dispositivi" con possibilità di indicarli o meno in verbale così che rimanga lo storico e si indichi solo il campione testato».
- **Transizione** (11:41): la sequenza importazione → migrazione delle schede → elenco degli impianti senza corrispondenza → quadratura va bene, con attenzione al passaggio SQLite → PostgreSQL.
- **Punto 4 nella rc8** (13:48): «includilo in rc8, ma verifica che non alteri le logiche nella generazione verbali e soprattutto nello storico verbali».

**Cosa fa la rc8** (punto 4):
- La bozza **nuova** di un verbale M01 (messa a terra) prende dall'impianto NEXUS potenza («250 kW»), POD, cabine e sottostazioni, come già cliente, sede e indirizzo. Un campo vuoto in NEXUS lascia il valore del motore.
- Le bozze già aperte, che possono contenere correzioni del tecnico, le revisioni e i verbali emessi **non vengono toccati**. Il test `test_punto4_non_tocca_bozze_aperte_ne_storico` confronta l'intero archivio dei verbali prima e dopo una modifica dell'impianto.
- Dopo l'emissione con «Aggiorna la scheda» tornano all'impianto la potenza (come prima), cabine e sottostazioni. Il POD torna solo se l'impianto non ne ha e non appartiene a un altro impianto. Con «Conserva la scheda» NEXUS resta com'è.
- Il preventivo calcola già la tariffa dalla potenza NEXUS (`TERRA-KW`); la fattura usa la tariffa fissata nella riga del preventivo (D1-bis rispettata).
- «kVA» non viene convertito: il valore resta nel verbale e nella scheda, NEXUS non cambia.

**Ancora aperto su D1:**
- **D1-ter.** Il codice (`preventivi/servizio.py::crea_revisione`, D16/D19) ammette la revisione solo da preventivi inviati o rifiutati, non accettati. Restano quattro domande:
  - (a) revisione anche di un preventivo accettato quando la potenza cambia lo scaglione?
  - (b) cosa fa la fattura finché la revisione non è accettata, o se il cliente la rifiuta?
  - (c) revisione solo se cambia lo scaglione?
  - (d) proposta di Claude: all'emissione NEXUS confronta la potenza con quella del preventivo, avvisa il tecnico e segnala alla segreteria, che crea la revisione.
- **Scheda completa** (tensione, sistema, Ul, If/tf, descrizioni, gruppi, attributi degli elementi) e **migrazione al taglio**: vedi § 7.2.

### 3.2 Verbale dentro NEXUS e firme: realizzato nella rc8

- La pagina `/verbali` resta generata dal motore, ma ha la barra NEXUS, nasconde il menu di VerbalizzAI e mostra la striscia di contesto con verifica, stato, passo successivo e «← Torna alla verifica». Dopo l'emissione il ritorno diventa il pulsante principale. Le viste «Verbali emessi · Bozze · Clienti e impianti» sostituiscono il menu nascosto. La voce di menu resta «**Verbali**»: la proposta «Archivio verbali» non è stata accolta.
- **Firme:** «Firma DPR» e «Firma DM» stanno nella stessa scheda del tecnico (profilo e Gestione › Firme dei verificatori, `/firme`). La firma DPR resta dove la cerca il motore, cioè nell'archivio dei verbali, e si scrive con le sue funzioni. I verbali già emessi conservano la firma inclusa. Il documento DM usa solo la firma DM (documento 39, invariato).
- **Da provare con gli operatori:** stesso tecnico con entrambe le firme, Word/PDF di entrambi gli ambiti, sostituzione di una firma.

### 3.3 Fattura di cortesia e configurazione: realizzato nella rc8, una verifica di Federico aperta

- Gestione › Configurazione › **Dati della società e pagamenti** (`/dati-societari`) contiene dati fiscali dell'emittente, modalità di pagamento e regole generali delle scadenze; i pulsanti relativi non sono più nella pagina Fatture. Le scadenze della singola fattura restano nella sua scheda.
- Gli avvisi della bozza dicono «Società emittente AD, dati fiscali (…)» oppure «Cliente «…», scheda del cliente: …», con un pulsante verso la sezione giusta e il ritorno alla bozza. Chi non ha il permesso `sistema.impostazioni` vede l'invito a rivolgersi a chi gestisce la Configurazione. Il nome della società viene dalla fattura, non è più «AD» fisso.
- Il PDF definitivo non ha la scritta «BOZZA» (`finanza/stampa.py`, solo se senza codice): da verificare con un'emissione locale di prova, senza invii a SdI o clienti.
- **Q4 aperta:** Federico deve verificare se, nel caso «PROVA POSTGRESQL», l'avviso spariva dopo aver salvato i dati AD («non mi ricordo lo verifico dopo»).

### 3.4 Navigazione e densità delle griglie: realizzato nella rc8

Barra orizzontale in alto (D3: «fai direttamente nella prossima scrittura del codice»). Misure a 1366×768:

| Pagina | Larghezza griglia | Colonne visibili |
|---|---|---|
| Scadenze | 996 → 1284 px | 6 → 8 |
| Preventivi | 996 → 1284 px | 7 → 9 |
| Verifiche | 996 → 1284 px | 8 → 10 |

Intestazione di pagina più compatta. **Ancora da fare:** sopra la griglia restano testo istruttivo, filtri rapidi, barra strumenti, barra delle selezioni e riga d'aiuto (Verifiche: griglia a circa 400 px dall'alto). È un intervento successivo, da proporre con esempi.

### 3.5 Destinazioni dei documenti e posta: destinazioni realizzate, posta aperta

- **D2** (11:26): «sul server in futuro su spazio condiviso, per il momento mi interessa che crei la funzione in configurazione per stabilire la destinazione. nexus girerà su vps linux ubuntu.»
- **rc8:** Gestione › Configurazione › **Destinazioni dei documenti**, con cinque percorsi del server (verbali DPR, verbali DM, schede tecniche DM, PDF fatture, XML fatture), il pulsante «Prova» e la traccia nel registro.
  - Copia automatica attiva solo per verbali e schede DM (come prima, ora dalle destinazioni). Le schede vanno nella loro destinazione se impostata, altrimenti con i verbali.
  - `cartella_verbali_dm` di config.json resta il valore iniziale.
  - Verbali DPR, PDF e XML: destinazione impostabile, copia **non ancora attiva** (servono nomi e struttura delle cartelle concordati con Federico).
  - Sul PC dell'operatore resta lo scaricamento dal browser.
- **Posta (aperto):** invio dal gestionale di preventivi, verbali DPR/DM e fatture di cortesia, con testi per tipologia (e per attività dove serve), destinatari proposti e modificabili, anteprima e storico degli invii. Oggi esistono gli invii di preventivi, verbali DPR e DM e la posta per attività; non esiste l'invio della fattura di cortesia. Durante il collaudo non inviare email reali se non a destinatari di prova autorizzati.

### 3.6 Flusso preventivo → incarico frammentato: registrato, non implementare finché non richiesto

Testo letterale (11:55): «gestione del flusso invio preventivo, accettazione, riesame, affidamento incarico di verifica troppo "frammentato" tra schede. magari posizione più "strategica" dei pulsanti o evidenziazione icona pulsante fase successiva (es., riesame per preventivo accettato, affidamento incarico a seguito di riesame, etc.) potrebbe aiutare.» La striscia «passo successivo» realizzata per il verbale (§ 3.2) è un possibile modello.

### 3.7 «Programma verifica» con email al cliente: registrato, non implementare finché non richiesto

Testo letterale (11:55): «implementare "programma verifica" con invio mail (da personalizzare, in configurazione), a cliente.» La programmazione esiste (`verifiche/servizio.py::programma`, `dm/tecnico.py::programma`), ma senza email. Va inserita nel sistema comune dei testi email per tipo (§ 3.5).

### 3.8 Difetti B1 e B2 sulla numerazione dei verbali: corretti nella rc8

- **B1** (11:57): dopo VP/001/26 una nuova verifica ha ricevuto di nuovo VP/001/26. Causa: data di esecuzione scritta 14/10/0226, quindi anno 226 → primo numero di «quell'anno» → codice «…/26». Il parser delle date accettava qualunque anno e il vincolo era solo su (anno, numero).
- **B2** (12:07): «in verbalizzAI originale il numero era inserito a mano, se ora lo cancello per via dell'errore non me lo ripropone più».
- **Decisioni (testo letterale):**
  - 12:07: «per il discorso date metterei un limite all'anno corrente, il precedente o il successivo. il codice del verbale deve comunque essere unico (n.b. VP o VS non possono avere stesso numero, è comunque un progressivo annuo)»;
  - 12:09: «se cambio data incluso anno chiaramente il numero di verbale cambierà di conseguenza. non introduciamo funzioni nuove»;
  - 12:15: «se scrivo 0026 e poi me ne accorgo (necessariamente), cambio data, quando metto 2026 mi proporrà un numero sequenziale dell'anno»;
  - 12:16: limite dell'anno «alla data che sto registrando in quel momento»; numero lasciato libero = «buco annotato».
- **rc8:**
  - Il limite dell'anno vale per registrazione, programmazione e data del verbale (DPR).
  - Il codice è unico per VP e VS.
  - Il numero nella bozza è in sola lettura e torna sempre quello della verifica.
  - Con un cambio d'anno arriva il primo numero libero dell'anno, la verifica si aggiorna e il registro lo annota; il vecchio numero non viene riassegnato.
  - Periodica ↔ Straordinaria cambia solo il prefisso.
  - Anteprima ed emissione sono bloccate se il numero non coincide.
  - Sull'archivio di collaudo di Federico, la verifica ALCOOLITAL si corregge salvando la bozza con la data 2026 e riceve il primo numero libero del 2026.
- **Vincolo per il passaggio finale:** il progressivo NEXUS dell'anno deve proseguire dall'ultimo numero usato in ingecert/VerbalizzAI autonomo (es. ultimo VP/649/26 → NEXUS da VP/650/26).
- **Aperto:** applicare il limite dell'anno anche al DM, se Federico lo chiede.

## 4. Cosa fare adesso

1. **Collaudo rc8 di Federico** su una cartella nuova, senza toccare la rc5 patchata:
   - `INSTALLA_WINDOWS.bat`, poi `pip install -r requirements-sviluppo.txt`;
   - test PostgreSQL nativi con `NEXUS_TEST_POSTGRESQL_ADMIN_URL`, poi suite completa;
   - prove manuali per ruolo (tecnico, RT distinto, segreteria, amministrazione) con gli stessi casi del 14/10: numerazione, firme, verbale dentro NEXUS, avvisi della fattura, destinazioni.
2. **Registro dei rilievi** dal collaudo: separare blocchi e dati errati da inefficienze e aspetto.
3. **Decisioni da chiudere:**
   - D1-ter (§ 3.1);
   - nomi e struttura delle cartelle per le copie DPR/fatture (§ 3.5);
   - scheda completa e migrazione (§ 7.2);
   - limite dell'anno nel DM;
   - Q4.
4. **Importazioni su copie isolate:**
   - censimento di ingecert MariaDB, allegati e corrispondenze, con quadrature;
   - backup *completo* di VerbalizzAI autonomo importato in un database di prova e confrontato con le verifiche importate.

   Non importare in modo definitivo finché VerbalizzAI riceve verbali.
5. **Backup/ripristino e server di collaudo isolato sul VPS**, distinto dalla VerbalizzAI di produzione. Prove su LibreOffice, ruoli, prestazioni e ripristino.
6. **Passaggio DPR:**
   1. fermo delle scritture sulla copia autonoma;
   2. backup finale;
   3. importazione una tantum;
   4. migrazione delle schede negli impianti e quadratura;
   5. allineamento del progressivo dei numeri;
   6. ripristino esercitato;
   7. avvio di tutti i nuovi incarichi DPR su NEXUS.

## 5. Istruzioni di continuità per l'istanza successiva

- Leggere: questo documento, `PASSAGGIO_CONSEGNE_NEXUS_0.18.0_rc8.md` (modifiche tecniche), i passaggi rc4–rc7, `NEXUS_DPR_POSTGRESQL_COLLAUDO.md`, le decisioni R1xx in `documentazione/fonti_ricevute_2026-10-13/18_Ripresa_unica_Gestionale_integrato_2026-10-11.md` § 7, e il codice.
- Verificare l'impronta dello ZIP. La rc8 è stata sviluppata con Git a partire dalla rc7 (7 commit, un argomento ciascuno); i messaggi sono riportati nel passaggio rc8.
- Non attivare `verbali_postgresql=true` né importare su `nexus_locale` senza una copia isolata e un piano di ripristino.
- A ogni modifica: proposta con esempio e conferma (§ 0), test di regressione, suite completa, aggiornamento di questo documento (§ 8) e di `VERSIONI.md`.
- Nel dubbio fra «comodo» e «non tocca lo storico dei verbali», scegliere il secondo e chiedere.

## 6. Decisioni e domande: riepilogo

| ID | Tema | Stato |
|---|---|---|
| D1, D1-bis, D1-quater | Attributi dell'impianto in NEXUS; fattura dal preventivo accettato | Decise; parte essenziale realizzata nella rc8 (§ 3.1) |
| **D1-ter** | Revisione di un preventivo accettato quando la potenza cambia lo scaglione | **Aperta** |
| D2 | Destinazioni sul server, VPS Ubuntu | Decisa; funzione realizzata, copie DPR/fatture da attivare |
| D3 | Barra in alto e verbale integrato nella rc8 | Decisa e realizzata |
| **Q4** | Avviso della fattura «PROVA POSTGRESQL» dopo il salvataggio dei dati AD | **Aperta** (verifica di Federico) |
| Q5 | Ruolo dell'istanza | Claude sviluppa la rc8 con conferma preventiva |
| B1, B2 | Numerazione dei verbali | Decise e realizzate |
| Nome del menu | «Verbali» o «Archivio verbali» | Resta «Verbali» |

## 7. Lavori della rc8 e passi successivi

### 7.1 Modifiche della rc8

Il dettaglio per file è in `PASSAGGIO_CONSEGNE_NEXUS_0.18.0_rc8.md`. In sintesi:
- barra in alto (D3);
- B1/B2;
- verbale dentro NEXUS;
- firme DPR+DM nella scheda del tecnico;
- destinazioni dei documenti;
- dati della società e pagamenti in Configurazione, con avvisi della fattura chiari;
- attributi NEXUS nella bozza nuova.

Nessuna migrazione, motore VerbalizzAI invariato, 433 test passati in Linux/PostgreSQL 16.

### 7.2 Scheda tecnica completa e migrazione: proposta da confermare con Federico

Stato attuale (rc8):
- la scheda completa resta nel profilo del motore, cioè nello schema `verbali` dello **stesso** database NEXUS;
- NEXUS ne mostra una copia nella linguetta «Scheda tecnica» e la modifica con «Modifica scheda tecnica», che ora si apre dentro NEXUS;
- i quattro attributi che decidono tariffa e anagrafica (potenza, POD, cabine, sottostazioni) fanno fede in NEXUS e passano nella bozza.

Il motivo è il vincolo del § 0, punto 4: il motore genera la bozza dal proprio profilo, e spostare tutta la scheda in tabelle NEXUS significherebbe cambiare come nasce la bozza.

Proposta per la versione successiva, **da presentare a Federico con esempi prima di scrivere codice**:
- (a) maschera NEXUS della scheda completa, che scrive nel profilo con la funzione di correzione già presente nel motore (`save_plant_state`, origine `manual_correction`), mantenendo la data di riferimento del profilo. Una data nuova bloccherebbe l'aggiornamento della scheda all'emissione di verbali con data precedente;
- (b) aggiunta di elementi dalla scheda impianto: oggi il motore li aggiunge o dismette solo in verifica, quindi serve una soluzione che non tocchi il motore oppure una sua nuova versione approvata;
- (c) al taglio: importazione del backup VerbalizzAI, poi `scheda.sincronizza_tutte` porta le schede negli impianti collegati, con un rapporto degli impianti senza corrispondenza o con conflitti e una quadratura.

## 8. Registro delle revisioni di questo documento

| Rev. | Data | Autore | Modifiche |
|---|---|---|---|
| 1 | 14/10/2026 | Istanza LLM precedente, con Federico | Stesura dopo i test PostgreSQL rc6/rc7 e il primo giro di appunti. |
| 2 | 14/10/2026 | Claude | Verifica dello ZIP rc7 e del codice; percorso della potenza; domande D1–Q5. Nessun codice modificato. |
| 3 | 14/10/2026 | Claude, con Federico | Regola di conferma preventiva; decisioni D1–D3, Q5, D1-bis, D1-quater, B1, B2 in testo letterale; appunti §§ 3.6–3.7; piano rc8; base di confronto 421 test. |
| 4 | 14/10/2026 | Claude | Consegna della rc8: stato per ogni decisione, prove eseguite (433 test), istruzioni per riprodurre l'ambiente, riepilogo delle decisioni aperte (D1-ter, Q4, cartelle DPR/fatture, scheda completa e migrazione, limite dell'anno nel DM), proposta § 7.2. |
