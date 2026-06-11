# Esame Analisi dei dati e Big Data

## Introduzione
Questo documento presenta la progettazione e l’analisi di due distinti domini applicativi attraverso l’uso di basi di dati relazionali e a grafo. La prima parte riguarda la gestione informatizzata di un gruppo alberghiero: vengono modellati hotel, camere, clienti e prenotazioni, con l’obiettivo di supportare il controllo della disponibilità, la prevenzione delle sovrapposizioni e la consultazione dello storico. Su questo modello vengono eseguite diverse query, tra cui l’analisi delle prenotazioni per intervallo temporale e l’identificazione dei clienti più attivi.

La seconda parte affronta la costruzione di un knowledge graph dedicato alle pubblicazioni scientifiche di un dipartimento di ricerca. Il modello include autori, articoli, citazioni e temi di ricerca, consentendo l’esplorazione delle reti di collaborazione e delle relazioni tematiche. Anche in questo caso vengono formulate query significative, come l’individuazione degli articoli che citano un determinato lavoro e degli autori che condividono un tema senza aver collaborato.

## 1. Sistema per Gruppo Alberghiero
**Traccia:** Un piccolo gruppo alberghiero vuole informatizzare la gestione delle proprie strutture. Ogni hotel è identificato da un codice, ha un nome, un indirizzo, una città e una categoria espressa in stelle. Ogni hotel dispone di diverse camere, contraddistinte da numero, tipologia, prezzo per notte e stato attuale, ad esempio libera, occupata o in manutenzione. Di ogni cliente si vogliono registrare codice, nome, cognome, telefono ed email. I clienti possono effettuare prenotazioni per una o più notti, indicando data di arrivo, data di partenza e numero di persone. Per ciascuna prenotazione il sistema deve associare una specifica camera e memorizzare anche l’importo totale previsto. La base di dati deve permettere di controllare la disponibilità delle camere e lo storico delle prenotazioni effettuate dai clienti.

### Diagramma ER
Un diagramma Entity–Relationship (ER) è uno strumento fondamentale nella progettazione di basi di dati perché permette di rappresentare in modo formale, visivo e concettualmente chiaro la struttura logica di un dominio. Nel contesto di questo progetto, la costruzione del modello ER è stata realizzata con il software ERDPlus, che consente di definire entità, attributi, relazioni e vincoli in modo rigoroso.

<img width="3552" height="1908" alt="erdplus (2)" src="https://github.com/user-attachments/assets/a2493d74-96f6-4135-9026-48c755dfb769" />

Il grafico rappresenta un'architettura di database altamente ottimizzata e strutturata in cinque entità principali, strettamente interconnesse. Si nota come l'entità Camera sia un'entità debole in relazione con hotel poiché il numero di una stanza (es. "101") non è univoco in senso assoluto nell'intero database, ma lo è solo all'interno del proprio hotel.

Le relazioni spiegano come le entità interagiscono tra di loro:
* POSSIEDE: è una relazione identificante, rappresentata dal doppio rombo. Unisce Hotel (1,N) a Camera (1,1). La doppia linea sul lato Camera indica una partecipazione totale (obbligatoria): una camera non può esistere nel database se non è legata a un hotel.
* CLASSIFICA: Unisce Tipologia (0,N) a Camera (1,1). La doppia linea indica che ogni camera fisica deve necessariamente afferire a una categoria di listino. Il minimo 0 sul lato Tipologia permette invece al management di creare una nuova categoria di stanza a sistema prima ancora che questa venga fisicamente costruita in un hotel.
* EFFETTUA: Collega Cliente (0,N) a Prenotazione (1,1). Un utente può registrarsi senza aver ancora prenotato (minimo 0), ma una prenotazione deve obbligatoriamente essere intestata a un cliente (doppia linea di partecipazione totale).
* ASSOCIA: È il cuore operativo del database. Collega Prenotazione (1,1) a Camera (0,N). Ogni prenotazione deve bloccare esattamente una camera (doppia linea), mentre una camera può accumulare nel tempo da 0 a N prenotazioni nello storico.

Nella progettazione del diagramma ER si è scelto di implementare alcune scelte di ottimizzazione ingegneristica:
* Scomposizione della Tipologia: la traccia suggeriva che la tipologia e il prezzo fossero attributi interni alla camera. In quel modo, per 100 camere "Suite" si sarebbe dovuta ripetere la parola "Suite" e il prezzo "150€" per cento volte. Estrarre l'entità Tipologia elimina alla radice la ridondanza dei dati e previene le anomalie di aggiornamento. Se l'hotel decide di cambiare il prezzo di listino delle Suite, basterà modificare una sola riga nella tabella Tipologia, e non cento righe nella tabella Camera.
* Attributo "Prezzo bloccato" sulla relazione: se si memorizzasse il prezzo solo nell'entità Tipologia, modificarlo in anni successivi rischierebbe di alterare retroattivamente i conti delle prenotazioni passate, distruggendo l'affidabilità dei bilanci storici richiesi dalla traccia. L'inserimento dell'attributo prezzo_notte_bloccato direttamente sul rombo della relazione ASSOCIA funge da snapshot del valore economico al momento della transazione. Il listino generale può cambiare, ma lo storico delle prenotazioni resta intatto e coerente nel tempo.
* Attributo "Importo Totale" come scelta di performance: l'importo totale sarebbe un attributo derivato. La scelta di salvarlo fisicamente come attributo di ASSOCIA è dettata da criteri di performance computazionale. Quando l'amministrazione richiederà report finanziari su scala annuale o mensile, il database non dovrà ricalcolare la matematica di migliaia di righe al volo, ma eseguirà una rapidissima operazione di lettura diretta, ottimizzando i tempi di risposta del sistema.

Questo diagramma costituisce la base concettuale per la successiva progettazione logica e fisica del database, assicurando che i vincoli del dominio siano rispettati e che le query richieste — come l’analisi delle prenotazioni, la verifica della disponibilità delle camere o l’identificazione dei clienti più attivi — possano essere eseguite in modo efficiente e coerente.

### Analisi dei Dati e Query SQL
Una volta definito il modello concettuale tramite diagramma ER, si è passati alla sua implementazione in SQL. Si è usato DBeaver come client per la scrittura e l'esecuzione del codice, utilizzando il server MySQL.

Il codice che segue rappresenta quindi la concretizzazione operativa del modello ER: ogni scelta sintattica e strutturale è finalizzata a preservare i vincoli concettuali, garantire coerenza dei dati e supportare query efficienti.

#### Creazione del DataBase, delle Tabelle e dei Trigger
```sql
CREATE DATABASE IF NOT EXISTS db_alberghiero;
USE db_alberghiero;
```
Crea lo spazio logico di archiviazione per le tabelle. La clausola `IF NOT EXISTS` è una fondamentale misura di sicurezza: impedisce al sistema di andare in blocco o di sovrascrivere un database preesistente contenente già dei dati qualora lo script venisse rieseguito per errore. Seleziona poi il DataBase appena creato come spazio operativo per tutte le istruzione successive.

```sql
CREATE TABLE Hotel (
    codice      VARCHAR(20) PRIMARY KEY,
    denominazione        VARCHAR(100) NOT NULL,
    indirizzo   VARCHAR(200) NOT NULL,
    citta       VARCHAR(100) NOT NULL,
    stelle      INT NOT NULL CHECK (stelle BETWEEN 1 AND 5)
);

INSERT INTO Hotel VALUES
('H001','Hotel Sole','Via Roma 10','Ostuni',4),
('H002','Mare Blu Resort','Lungomare 25','Brindisi',5),
('H003','Collina Verde','Via dei Pini 8','Cisternino',3),
('H004','Trulli Paradise','Contrada Monte 12','Alberobello',4),
('H005','Porto Sereno','Via del Porto 3','Monopoli',4);
```
* Codice `VARCHAR(20) PRIMARY KEY`: Definisce la chiave primaria. L'uso di `VARCHAR` rispetto a un tipo `CHAR` fisso permette di risparmiare spazio su disco (allocando solo i caratteri effettivamente usati) e garantisce flessibilità se in futuro il gruppo alberghiero decidesse di adottare codici alfanumerici complessi;
* Il vincolo `NOT NULL` impone l'obbligatorietà del dato. Lasciare i campi opzionali (valore `NULL` consentito) comprometterebbe la qualità dei dati aziendali, rendendo impossibile la fatturazione o il contatto con la struttura;
* `stelle INT NOT NULL CHECK (stelle BETWEEN 1 AND 5)` implementa un vincolo di integrità sui valori inseribili. Risulta più efficace rispetto a un semplice campo intero aperto perché delega il controllo della qualità del dato direttamente al motore del database.
Si è poi popolata la tabella creata.

```sql
CREATE TABLE Tipologia (
    id_tipologia        VARCHAR(20) PRIMARY KEY,
    categoria                VARCHAR(50) NOT NULL,
    prezzo_per_notte   DECIMAL(10, 2) NOT NULL CHECK (prezzo_per_notte >= 0)
);

INSERT INTO Tipologia VALUES
('T1','Singola',55),
('T2','Doppia',85),
('T3','Suite',150);
```
La scelta del tipo di dato `DECIMAL` rispetto a `FLOAT` o `REAL` è molto importante. I tipi a virgola mobile soffrono di errori di arrotondamento; nei calcoli finanziari e di bilancio aziendali tali discrepanze si accumulerebbero, quindi `DECIMAL(10,2)` garantisce accuratezza matematica memorizzando fino a 10 cifre totali di cui esattamente 2 decimali.

`CHECK (prezzo_per_notte >= 0)` blocca l'inserimento di tariffe negative, un paradosso commerciale che manderebbe in anomalia i calcoli del fatturato.

Si sono poi definite tre categorie e il listino prezzi e sono stati inseriti nella tabella, fornendo la basa per la classificazione delle camere.

```sql
CREATE TABLE Camera (
    hotel_codice    VARCHAR(20) NOT NULL,
    numero          VARCHAR(10) NOT NULL,
    id_tipologia    VARCHAR(20) NOT NULL,
    stato           VARCHAR(20) DEFAULT 'Libera' 
        CHECK (stato IN ('Libera', 'Occupata', 'In manutenzione')),
    
    PRIMARY KEY (hotel_codice, numero),
    
    FOREIGN KEY (hotel_codice) 
        REFERENCES Hotel(codice)
        ON UPDATE CASCADE
        ON DELETE CASCADE,
        
    FOREIGN KEY (id_tipologia) 
        REFERENCES Tipologia(id_tipologia)
        ON UPDATE CASCADE
        ON DELETE RESTRICT
);

INSERT INTO Camera (hotel_codice, numero, id_tipologia, stato)
SELECT
    CASE
        WHEN n <= 20 THEN 'H001'
        WHEN n <= 40 THEN 'H002'
        WHEN n <= 60 THEN 'H003'
        WHEN n <= 80 THEN 'H004'
        ELSE 'H005'
    END AS hotel_codice,
    LPAD((n-1)%20+1, 3, '0') AS numero,
    ELT(FLOOR(1 + RAND()*3), 'T1','T2','T3') AS id_tipologia,
    ELT(FLOOR(1 + RAND()*3), 'Libera','Occupata','In manutenzione') AS stato
FROM (
    SELECT @row := @row + 1 AS n
    FROM information_schema.columns, (SELECT @row := 0) r
    LIMIT 100
) AS seq;
```
Si è scelto di impostare `DEFAULT 'Libera'` poiché quando una nuova camera viene inserita nel sistema, lo stato iniziale più probabile è che sia pronta e disponibile. Evita di dover specificare questo valore manualmente in ogni istruzione di inserimento.

`CHECK (stato IN (...))` restringe il dominio dei dati consentiti alle sole tre parole autorizzate. Impedisce l'introduzione di stringhe personalizzate o errori di battitura.

La riga `PRIMARY KEY (hotel_codice, numero)` concretizza la natura di entità debole della camera. La combinazione dei due campi costituisce una chiave primaria composta. È una scelta strutturalmente migliore rispetto alla creazione di un ID autoincrementale singolo (es. ID_camera 1, 2, 3...) perché rispecchia fedelmente la logica del mondo reale.
Le righe successive stabiliscono il vincolo di integrità referenziale con la tabella `Hotel`.

Così com'è programmato (`ON UPDATE CASCADE`), se il codice di un hotel dovesse variare, la modifica si propagherebbeautomaticamente a tutte le camere collegate, azzerando i tempi di manutenzione manuale. `ON DELETE CASCADE`, invece, è essenziale per le entità deboli: se un hotel viene rimosso dal database, vengono rimosse anche tutte le camere.

`FOREIGN KEY (id_tipologia)` collega la camera alla sua tipologia. `ON DELETE RESTRICT` impedisce ad un operatore di rimuovere una categoria (es. Suite) fino a quando c'è almeno una camera collegata ad essa. Questo impedisce che le camere rimangano improvvisamente prive di una categoria e di un prezzo associato.

La fase di popolamento della tabella `Camera` sfrutta una strategia che delega al motore SQL la generazione controllata dei dati. Per ottenere esattamente cento camere, lo script utilizza una sottoquery — indicata con l’alias `seq` — che interroga la vista di sistema `information_schema.columns` non per ricavarne informazioni strutturali, ma come puro generatore di righe. Combinando questa entità con una variabile inizializzata a zero, si ottiene una sequenza numerica progressiva (`n`), limitata a cento elementi tramite `LIMIT 100`. Tale sequenza costituisce l’ossatura logica su cui costruire gli attributi dell’entità debole `Camera`.

Il valore di `n` viene dapprima interpretato dal costrutto `CASE`, che suddivide l’intervallo in cinque gruppi da venti unità, assegnando ciascun blocco a un diverso hotel. Questa scelta riflette il vincolo concettuale secondo cui le camere dipendono dall’hotel e consente di distribuire in modo uniforme le unità tra le strutture. Successivamente, il numero di camera viene ricostruito come identificatore parziale che riparte da 1 per ogni hotel: la funzione `LPAD` garantisce una formattazione coerente a tre cifre, migliorando leggibilità e ordinamento. Per introdurre variabilità realistica, lo script impiega una combinazione di `RAND`, `FLOOR` ed `ELT`. `RAND` genera un valore casuale continuo, trasformato in un intero compreso tra 1 e 3 tramite `FLOOR`; questo indice viene poi utilizzato da `ELT` per selezionare una tipologia o uno stato da un insieme predefinito. In questo modo, la casualità rimane confinata entro un dominio controllato, evitando l’inserimento di valori non ammessi e rispettando i vincoli di integrità.
Nel complesso, questa architettura consente di ottenere un popolamento coerente, vario e pienamente aderente al modello concettuale, senza ricorrere a script esterni o procedure iterative. La logica è interamente demandata al database, che garantisce prestazioni elevate e un controllo rigoroso sui vincoli referenziali e di dominio.

```sql
CREATE TABLE Cliente (
    codice_cliente      VARCHAR(20) PRIMARY KEY,
    nome        VARCHAR(50) NOT NULL,
    cognome     VARCHAR(50) NOT NULL,
    telefono    VARCHAR(20),
    email       VARCHAR(100) UNIQUE NOT NULL
);

SET @nomi = 'Marco,Giulia,Luca,Sara,Francesco,Chiara,Alessandro,Martina,Giorgio,Elena';
SET @cognomi = 'Rossi,Bianchi,Verdi,Ferrari,Esposito,Romano,Gallo,Costa,Fontana,Greco';

INSERT INTO Cliente (codice_cliente, nome, cognome, telefono, email)
SELECT
    CONCAT('C', LPAD(n,3,'0')),
    ELT(FLOOR(1 + RAND()*10), 'Marco','Giulia','Luca','Sara','Francesco','Chiara','Alessandro','Martina','Giorgio','Elena'),
    ELT(FLOOR(1 + RAND()*10), 'Rossi','Bianchi','Verdi','Ferrari','Esposito','Romano','Gallo','Costa','Fontana','Greco'),
    CONCAT('3', LPAD(FLOOR(RAND()*999999999), 9, '0')),
    CONCAT(
        LOWER(ELT(FLOOR(1 + RAND()*10), 'marco','giulia','luca','sara','francesco','chiara','alessandro','martina','giorgio','elena')),
        '.',
        LOWER(ELT(FLOOR(1 + RAND()*10), 'rossi','bianchi','verdi','ferrari','esposito','romano','gallo','costa','fontana','greco')),
        n,
        '@example.com'
    )
FROM (
    SELECT @i := @i + 1 AS n
    FROM information_schema.columns, (SELECT @i := 0) r
    LIMIT 60
) AS seq;
```
Il telefono è l'unico attributo descrittivo che non presenta il vincolo `NOT NULL`. Logicamente, un utente potrebbe effettuare una prenotazione online inserendo solo l'email. L'aggiunta del vincolo `UNIQUE` sull'email garantisce un'importante regola di pulizia del database: non possono esistere due profili clienti differenti con la stessa identica casella postale. Evita la proliferazione di account duplicati.

Sfruttando lo stesso motore sequenziale basato sul conteggio delle colonne di sistema, si è popolata la tabella con 60 clienti fittizzi. `CONCAT(LOWER(...), '.', LOWER(...), n, '@example.com')` costruisce dinamicamente un indirizzo email realistico e formalmente valido, combinando il nome e il cognome estratti, convertiti in minuscolo tramite `LOWER`, e aggiungendo il valore incrementale n per garantire matematicamente il rispetto del vincolo di univocità (`UNIQUE`) precedentemente dichiarato.

```sql
CREATE TABLE Prenotazione (
    id_prenotazione          VARCHAR(20) PRIMARY KEY,
    cliente_codice           VARCHAR(20) NOT NULL,
    hotel_codice             VARCHAR(20) NOT NULL,
    camera_numero            VARCHAR(10) NOT NULL,
    data_arrivo              DATE NOT NULL,
    data_partenza            DATE NOT NULL,
    numero_pax               INT NOT NULL CHECK (numero_pax > 0),
    stato_prenotazione       VARCHAR(20) DEFAULT 'Confermata' 
        CHECK (stato_prenotazione IN (
            'In attesa', 'Confermata', 'In corso', 
            'Completata', 'Cancellata', 'No-Show'
        )),
    importo_totale           DECIMAL(10, 2) NOT NULL CHECK (importo_totale >= 0),
    prezzo_notte_bloccato    DECIMAL(10, 2) NOT NULL CHECK (prezzo_notte_bloccato >= 0),
    
    FOREIGN KEY (cliente_codice) 
        REFERENCES Cliente(codice_cliente)
        ON UPDATE CASCADE
        ON DELETE CASCADE,
        
    FOREIGN KEY (hotel_codice, camera_numero) 
        REFERENCES Camera(hotel_codice, numero)
        ON UPDATE CASCADE
        ON DELETE CASCADE,
    
    CHECK (data_partenza > data_arrivo)
);
```
`numero_pax INT NOT NULL CHECK (numero_pax > 0)` applica un controllo fondamentale a livello di database: una prenotazione deve comprendere almeno un ospite pagante. L'attributo `stato_prenotazione` così programmato traccia con precisione l'intero ciclo di vita di una prenotazione, permettendo al sistema di distinguere tra una riserva attiva, un cliente attualmente in hotel (`In corso`), un soggiorno concluso con successo (`Completata`) o contratti falliti/annullati. `importo_totale` e `prezzo_notte_bloccato` memorizzano fisicamente i due parametri economici discussi nella fase concettuale. Questa soluzione di archiviazione è preferibile rispetto al ricalcolo dinamico perché tutela l'immutabilità dello storico finanziario della catena e velocizza le risposte del database. Con `ON DELETE CASCADE` sulle `FOREIGN KEY (cliente_codice)`, se un cliente viene rimosso dal sistema, anche lo storico delle sue prenotazioni scompare a cascata. La `FOREIGN KEY (hotel_codice, camera_numero)` implementa la relazione ASSOCIA. Essendo la camera identificata da una chiave composta, la chiave esterna della prenotazione deve obbligatoriamente ereditare ed esprimere entrambi i campi (`hotel_codice` e `camera_numero`).
`CHECK (data_partenza > data_arrivo)` impedisce errori materiali da parte degli operatori della reception o dei clienti sul web, bloccando alla radice inserimenti in cui la data di partenza sia antecedente o uguale alla data di arrivo.

A questo punto si procede ad attivare un `TRIGGER` per calcolare l'importo totale prima di popolare la tabella, poiché il trigger deve essere già attivo al momento dell'esecuzione dell’`INSERT ... SELECT`, così ogni nuova prenotazione ha l’`importo_totale` calcolato automaticamente, invece di rimanere a 0. In questo modo, il dataset di test rispetta da subito la logica di business: l’importo non è un valore arbitrario, ma deriva coerentemente da prezzo per notte e numero di notti. Il codice si è scritto su un altro script per pulizia e praticità nell'esecuzione e nella modifica.

```sql
USE db_alberghiero;

DELIMITER $$

CREATE TRIGGER trg_calcola_importo_totale
BEFORE INSERT ON Prenotazione
FOR EACH ROW
BEGIN
    DECLARE numero_notti INT;
    SET numero_notti = DATEDIFF(NEW.data_partenza, NEW.data_arrivo);
    SET NEW.importo_totale = NEW.prezzo_notte_bloccato * numero_notti;
END$$

DELIMITER ;
```
* `CREATE TRIGGER trg_calcola_importo_totale` definisce un trigger associato alla tabella `Prenotazione`.
* `BEFORE INSERT`: il codice viene eseguito prima dell’inserimento della riga, permettendo di modificare i valori di `NEW`.
* `FOR EACH ROW`: il trigger si applica a ogni prenotazione inserita, non una sola volta per l’intera istruzione.
* `BEGIN`: apre il blocco di istruzioni del trigger. `DECLARE` dichiara una variabile locale `numero_notti` di tipo intero, usata per memorizzare la durata del soggiorno. Migliora la leggibilità rispetto a usare l’espressione direttamente nella moltiplicazione. `SET` calcola il numero di notti come differenza tra data di partenza e data di arrivo della prenotazione appena inserita (`NEW`). Il `SET` successivo imposta l'importo totale come prodotto tra il prezzo per notte bloccato e il numero di notti. In questo modo, l’importo è sempre derivato automaticamente dal motore SQL e non dipende da calcoli esterni o da valori inseriti manualmente, che potrebbero essere incoerenti.

Si procede a questo punto a popolare la tabella delle `Prenotazioni`.

```sql
INSERT INTO Prenotazione (
    id_prenotazione, cliente_codice, hotel_codice, camera_numero,
    data_arrivo, data_partenza, numero_pax,
    stato_prenotazione, importo_totale, prezzo_notte_bloccato
)
SELECT
    CONCAT('P', SUBSTRING(UUID(), 1, 8)) AS id_prenotazione,
    CONCAT('C', LPAD(FLOOR(1 + RAND()*60),3,'0')) AS cliente_codice,
    c.hotel_codice,
    c.numero AS camera_numero,
    DATE_ADD('2026-01-01', INTERVAL @arr := FLOOR(RAND()*300) DAY) AS data_arrivo,
    DATE_ADD('2026-01-01', INTERVAL @arr + FLOOR(1 + RAND()*7) DAY) AS data_partenza,
    FLOOR(1 + RAND()*3) AS numero_pax,
    ELT(FLOOR(1 + RAND()*6),
        'In attesa','Confermata','In corso','Completata','Cancellata','No-Show'
    ) AS stato_prenotazione,
    0 AS importo_totale,
    CASE c.id_tipologia
        WHEN 'T1' THEN 55
        WHEN 'T2' THEN 85
        ELSE 150
    END AS prezzo_notte_bloccato
FROM (
    SELECT @p := @p + 1 AS n
    FROM information_schema.columns, (SELECT @p := 0) r
    LIMIT 250
) AS seq
JOIN (
    SELECT hotel_codice, numero, id_tipologia
    FROM Camera
    ORDER BY RAND()
) AS c
ON 1=1;
```
`CONCAT('P', SUBSTRING(UUID(), 1, 8))` genera una chiave alfanumerica univoca per ogni prenotazione estraendo i primi 8 caratteri da un identificativo generato casualmente tramite algoritmo `UUID()`. Questo approccio imita i codici di prenotazione reali usati dalle compagnie aeree e dalle agenzie di viaggio. `DATE_ADD('2026-01-01', ...)` sposta le date di arrivo e partenza in avanti nel corso di tutto l'anno 2026 usando la variabile di sessione `@arr`. Garantisce matematicamente che la data di partenza superi quella di arrivo di un valore variabile da 1 a 7 notti, rispettando pienamente il vincolo `CHECK`. `0 AS importo_totale` inserisce temporaneamente il valore 0 poichè, seguendo l'ottimizzazione architetturale corretta, lo script presuppone che a questo punto sia stato attivato nel database il trigger `trg_calcola_importo_totale`. Il trigger intercetta questa `INSERT`, calcola la durata del soggiorno in notti e sovrascrive istantaneamente lo zero calcolando l'importo corretto in modo automatico. `JOIN ... ON 1=1` è un'istruzione di Cross Join: moltiplica la sequenza di 250 cicli per le camere estratte casualmente tramite `ORDER BY RAND()`, generando così una vasta mole di record storici.

Si procede infine a visualizzare le prima 20 righe di tutte le tabelle.
```sql
SELECT * FROM Hotel LIMIT 20;
SELECT * FROM Camera LIMIT 20;
SELECT * FROM Cliente LIMIT 20;
SELECT * FROM Prenotazione LIMIT 20;
SELECT * FROM Tipologia;
```
Successivamente, si è pensato di inserire un ulteriore trigger per evitare che si effettui una prenotazione per una camera che risutla già occupata. Questo è stato implementato in questa fase come scelta strategica poiché risponde alla necessità metodologica legata alla gestione dei dati di test. La query di inserimento analizzata in precedenza genera in modo casuale 250 prenotazioni distribuite nell'arco di un anno per 100 camere. Trattandosi di un algoritmo basato sulla casualità pura delle date, la probabilità matematica che si verifichino sovrapposizioni temporali tra le prenotazioni destinate alla stessa camera è pressoché del 100%.

Se il trigger venisse attivato prima dell'esecuzione di questo inserimento di massa, intercetterebbe la primissima sovrapposizione casuale e interromperebbe immediatamente lo script, lasciando il database privo dei dati necessari per i successivi test. Pertanto, è meglio effettuare prima il caricamento del dataset storico e solo successivamente registrare il trigger nel catalogo. In questo modo, l'oggetto entra in funzione come un "guardiano" in tempo reale esclusivamente per le transazioni operative future. Questo trigger è utile e strategico poiché in questo modo la gestione dell'overbooking è affidata direttamente al motore SQL, quindi nessuna prenotazione può essere effettuata se non rispetta il vincolo, indipendentemente da chi esegue l'`INSERT`. Inoltre, traduce operativamente quanto detto nel diagramma ER, ovvero che una camera non può avere più prenotazioni contemporaneamente. 

```sql
USE db_alberghiero;

DELIMITER $$

CREATE TRIGGER trg_no_overlapping_reservations
BEFORE INSERT ON Prenotazione
FOR EACH ROW
BEGIN
    IF EXISTS (
        SELECT 1
        FROM Prenotazione p
        WHERE 
            p.hotel_codice = NEW.hotel_codice
            AND p.camera_numero = NEW.camera_numero
            AND p.stato_prenotazione NOT IN ('Cancellata', 'No-Show')
            AND (
                NEW.data_arrivo < p.data_partenza
                AND NEW.data_partenza > p.data_arrivo
            )
    ) THEN
        SIGNAL SQLSTATE '45000' 
		SET MESSAGE_TEXT = 'Errore: La camera selezionata è già occupata in quelle date.';
    END IF;
END$$

DELIMITER ;
```
`DELIMITER $$` cambia il delimitatore dei comandi da `;` a `$$`, così MySQL può interpretare l’intero blocco `BEGIN…END` come un’unica istruzione, non spezzata dai `;` interni. `IF EXISTS (...)` verifica se esiste almeno una prenotazione già registrata che entra in conflitto con quella nuova. `p.stato_prenotazione NOT IN ('Cancellata', 'No-Show')` esclude dal controllo le prenotazioni che non occupano realmente la camera; è una scelta logica: solo le prenotazioni attive o concluse ma effettivamente svolte devono impedire nuove prenotazioni sulle stesse date. Se esiste almeno una prenotazione che soddisfa queste condizioni e quelle sulle date, `IF EXISTS` risulta vero e si entra nel blocco `THEN`. `SIGNAL SQLSTATE '45000'` genera un errore personalizzato e interrompe l’operazione di inserimento. `SET MESSAGE_TEXT = '...'` imposta il messaggio di errore che verrà restituito al client; in questo caso, comunica chiaramente che la camera è già occupata nelle date indicate.

#### Query effettuate
Una volta costruito il modello concettuale, tradotto in schema logico‑relazionale e popolato il database con dati coerenti, si è deciso di creare una serie di query per estrarre conoscenza, verificare la correttezza del modello e, soprattutto, simulare le reali esigenze gestionali di una struttura alberghiera. In particolare:
* alcune interrogazioni permettono di valutare la capacità ricettiva e la distribuzione delle camere (es. numero totale di camere per hotel, camere disponibili in un intervallo di date);
* altre misurano la domanda effettiva, individuando gli hotel più prenotati, le camere più richieste e i periodi di maggiore attività;
* altre ancora analizzano il comportamento dei clienti, identificando quelli più attivi o più redditizi, oppure quelli che generano più cancellazioni;
* un gruppo di query è dedicato alla performance economica, con calcoli di fatturato totale, fatturato per hotel, andamento mensile e impatto economico delle cancellazioni;
* infine, alcune interrogazioni svolgono un ruolo di controllo e diagnostica, come l’individuazione di prenotazioni sovrapposte o situazioni di potenziale overbooking.

Ciascuna query verrà analizzata nel dettaglio, evidenziando: quale domanda gestionale risolve, quale logica implementa, quali informazioni restituisce e come tali informazioni possono essere utilizzate in un contesto reale.

La prima query mira ad ottenere la lista degli hotel con il numero totale di camere associate a ciascuno. È una delle prime informazioni che un gestore o un analista vorrebbe conoscere per capire la capacità ricettiva del gruppo alberghiero.
```sql
SELECT 
    h.codice AS hotel_codice,
    h.denominazione   AS hotel_nome,
    COUNT(c.numero) AS totale_camere
FROM Hotel h
LEFT JOIN Camera c
    ON c.hotel_codice = h.codice
GROUP BY h.codice, h.denominazione
ORDER BY totale_camere DESC;
```
`SELECT` definisce le colonne da restituire nel risultato. `COUNT(c.numero) AS totale_camere` conta quante righe di `Camera` sono associate a ciascun hotel; poiché ogni riga rappresenta una camera, il conteggio restituisce il numero totale di camere per hotel. `LEFT JOIN` collega la tabella `Camera` (`c`) alla tabella `Hotel` (`h`), mantenendo tutti gli hotel anche se non hanno camere associate; questo è importante in fase di analisi e di test del modello: un hotel con `totale_camere = 0` segnala un’anomalia o una fase di configurazione incompleta. `ON c.hotel_codice = h.codice` specifica la condizione di join: una camera è associata a un hotel quando il suo hotel_codice coincide con il codice dell’hotel. `GROUP BY h.codice, h.denominazione` raggruppa le righe per hotel e, infine con `ORDER BY totale_camere DESC` si ordina il risultato in ordine decrescente.

Per ogni hotel, la query fornisce: il codice identificativo (`hotel_codice`), utile per riferimenti tecnici; il nome dell’hotel (`hotel_nome`), utile per i report; il numero totale di camere (totale_camere), che quantifica la capacità fisica della struttura. Queste informazioni sono utili per capire quali hotel hanno maggiore capacità e quindi possono assorbire più domanda in alta stagione, valutare se la distribuzione delle camere tra le strutture è equilibrata rispetto alla domanda attesa, individuare rapidamente eventuali errori di popolamento.

La seconda query è ancora descrittiva, usata per calcolare il numero totale di camere per tipologia.
```sql
SELECT 
    t.id_tipologia,
    t.categoria,
    COUNT(*) AS numero_camere
FROM Camera c
JOIN Tipologia t
    ON t.id_tipologia = c.id_tipologia
GROUP BY t.id_tipologia, t.categoria
ORDER BY numero_camere DESC;
```
I risultati permettono di capire come è stutturata l'offerta ricettiva, aiutando ad individuale la necessità di ristrutturazione o di riconversione delle camere.

Successivamente si è voluto calcolare quali hotel ricevono più prenotazioni. È una metrica chiave per valutare la domanda, la popolarità e la performance commerciale delle strutture.
```sql
SELECT 
    h.codice AS hotel_codice,
    h.denominazione   AS hotel_nome,
    COUNT(p.id_prenotazione) AS numero_prenotazioni
FROM Prenotazione p
JOIN Hotel h
    ON h.codice = p.hotel_codice
GROUP BY h.codice, h.denominazione
ORDER BY numero_prenotazioni DESC;
```
`JOIN` collega ogni prenotazione al relativo hotel. `ON h.codice = p.hotel_codice` associa ogni prenotazione all’hotel a cui appartiene. Essendo un `JOIN` (non `LEFT JOIN`), gli hotel senza prenotazioni non compariranno nel risultato; infatti, la query vuole analizzare solo gli hotel che hanno ricevuto almeno una prenotazione.

Per ogni hotel, la query fornisce: il codice identificativo (`hotel_codice`), il nome dell’hotel (`hotel_nome`) e il numero totale di prenotazioni (`numero_prenotazioni`). Questi dati permettono di confrontare rapidamente le strutture.

Questa query permette di individuare strutture con bassa domanda che potrebbero necessitare interventi, effettuare una pianificazione operativa e prendere decisioni strategiche sugli investimenti da compiere.

Successivamente, si è formulata una query per calcolare le notti medie prenotate per camera, non in un singolo periodo ma sui dati dell'intero DataBase. Questa misura può essere un indicatore di intensità di utilizzo delle camere, domanda media e performance dell'hotel.
```sql
SELECT
    h.codice AS hotel_codice,
    h.denominazione   AS hotel_nome,
    SUM(DATEDIFF(p.data_partenza, p.data_arrivo)) AS notti_prenotate,
    COUNT(DISTINCT c.hotel_codice, c.numero)      AS numero_camere,
    SUM(DATEDIFF(p.data_partenza, p.data_arrivo)) 
        / NULLIF(COUNT(DISTINCT c.hotel_codice, c.numero), 0) AS notti_medie_per_camera
FROM Prenotazione p
JOIN Hotel h
    ON h.codice = p.hotel_codice
JOIN Camera c
    ON c.hotel_codice = p.hotel_codice AND c.numero = p.camera_numero
GROUP BY h.codice, h.denominazione;
```
`SUM(DATEDIFF(p.data_partenza, p.data_arrivo)) AS notti_prenotate` calcola la somma di tutte le notti prenotate nell'hotel. `COUNT(DISTINCT c.hotel_codice, c.numero) AS numero_camere` conta solo le camere coinvolte in almeno una prenotazione. `SUM(DATEDIFF(...)) / NULLIF(COUNT(DISTINCT ...), 0) AS notti_medie_per_camera` misura quanto ogni camere, in media, è stata occupata. Al numeratore ci sono il totale delle notti prenotate, mentre al denominatore c'è il numero di camere coinvolte; `NULIF(...,0)` evita la divisione per zero.

I risultati di questa query possono essere utilizzati per valutare la performance dell'hotel, identificare gli hotel che possono aumentare i prezzi delle camere o quelli che hanno bisogno di promozioni, e pianificare gli investimenti per l'espansione della struttura. Inoltre, i risultati possono essere utilizzati in un'analisi comparativa con il fatturato dell'hotel per ottenere una visione più completa della perfomance della struttura.

Si è poi pensata una query che determinasse quali camere sono disponibili in un intervallo di date, evitando sovrapposizioni con prenotazioni attive. Questa è una query fondamentale da implementare in motori di prenotazione e per la gestione operativa.
```sql
SELECT 
    c.hotel_codice,
    c.numero AS camera_numero
FROM Camera c
WHERE NOT EXISTS (
    SELECT 1
    FROM Prenotazione p
    WHERE p.hotel_codice = c.hotel_codice
      AND p.camera_numero = c.numero
      AND p.stato_prenotazione NOT IN ('Cancellata', 'No-Show')
      AND (
            :data_inizio < p.data_partenza
        AND :data_fine   > p.data_arrivo
      )
);
```
`WHERE NOT EXISTS ( ... )` permette di mostrare solo le camere per cui non esiste una prenotazione in quel periodo. La subquery interna associa la prenotazione alla camera tramite la chiave composta e considera solo le prenotazioni che occupano realmente la camera (quindi tutti gli stati della prenotazione tranne `Cancellata` e `No-Show`). `AND ( :data_inizio < p.data_partenza AND :data_fine > p.data_arrivo )` è la condizione di sovrapposizione degli intervalli temporali; se la condizione è vera, la camera è occuoata. Per via di `NOT EXISTS`, la camera viene restituita solo se non esiste alcuna prenotazione attiva che si sovrappone al periodo richiesto. Nell'output della query ogni riga rappresenta una camera disponibile nel periodo richiesto.

Questa query è essenziale per la gestione operativa e strategica del sistema alberghiero. Può essere usata per l'assegnazione delle camere, quindi come base per check-in, estensioni di soggiorno e gestione di guasti improvvisi, e per la pianificazione delle risorse umane, usandola come un ihdicatore per decidere la turnazione del personale; previene, inoltre, l'overbooking e permette di gestire la chiusura dei canali di vendita e la scelta di adottare strategie last-minute (quando ci sono molte camere libere) o pricing dinamico (aumentando i prezzi quando ci sono poche camere libere). Infine, la query può essere utilizzata per identificare periodi di alta o bassa occupazione e analizzare pattern stagionali.

Si è poi voluto valutare quali fossero le camere più richieste, cioè quelle che hanno ricevuto il maggior numero di prenotazioni. Questa domanda può essere importante per comprendere le preferenze dei clienti e la distribuzione della pressione operativa sulle camere, permettendo al front offici di distribuire meglio le prenotazioni.
```sql
SELECT 
    p.hotel_codice,
    p.camera_numero,
    COUNT(*) AS numero_prenotazioni
FROM Prenotazione p
GROUP BY p.hotel_codice, p.camera_numero
ORDER BY numero_prenotazioni DESC;
```
Le camere più richieste possono avere caratteristiche apprezzate dai clienti, e questo potrebbe guistare scelte di ristrutturazione; inoltre, le camere più richieste possono essere vendute ad un prezzo più alto, mentre le camere meno richieste possono essere migliorate o rese oggetto di promozioni. Infine, è utile per gestire la pianificazione della manutenzione, poiché le camere più richieste vanno incontro a maggiore usare e devono essere controllate più spesso.

Continuando l'analisi sulla disponibilità e l'utilizzo delle camere, si è pensata una query per identificare tutte le coppie di prenotazioni che si sovrappongono sulla stessa camere, quindi potenziali casi di overbooking o giorni a rischio di overbooking. Questa può essere utilizzata come query diagnostica per controllare se il trigger anti-overbooking sta funzionando.
```sql
SELECT 
    p1.hotel_codice,
    p1.camera_numero,
    p1.id_prenotazione AS pren1,
    p2.id_prenotazione AS pren2,
    p1.data_arrivo,
    p1.data_partenza,
    p2.data_arrivo,
    p2.data_partenza
FROM Prenotazione p1
JOIN Prenotazione p2
    ON p1.hotel_codice = p2.hotel_codice
   AND p1.camera_numero = p2.camera_numero
   AND p1.id_prenotazione < p2.id_prenotazione
   AND p1.stato_prenotazione NOT IN ('Cancellata', 'No-Show')
   AND p2.stato_prenotazione NOT IN ('Cancellata', 'No-Show')
   AND (
        p1.data_arrivo < p2.data_partenza
    AND p1.data_partenza > p2.data_arrivo
   );
```
La query confronta ogni prenotazione con tutte le altre della stessa camera. È un self‑join della tabella `Prenotazione`. `p1.id_prenotazione < p2.id_prenotazione` serve per evitare duplicati (pren1‑pren2 e pren2‑pren1) e confronti di una prenotazione con sé stessa. 

Se il trigger funziona correttamente, questa query dovrebbe restituire zerp righe, quindi è utile per controllare il suo funzionamento. I risultati permettono di identificare possibili errori di assegnazione o periodi di alta pressione. Il personale può utilizzare questi dati per contattare i clienti coinvolti e spostare le prenotazioni, in modo da evitare disservizi.

Successivamente, si è implementata una query per verificare quali clienti hanno effettuato più prenotazioni. Può essere utile per identificare clienti abituali e opportunità di fidelizzazione.
```sql
SELECT 
    c.codice_cliente,
    c.nome,
    c.cognome,
    COUNT(p.id_prenotazione) AS numero_prenotazioni
FROM Prenotazione p
JOIN Cliente c
    ON c.codice_cliente = p.cliente_codice
GROUP BY c.codice_cliente, c.nome, c.cognome
ORDER BY numero_prenotazioni DESC;
```
Se integrata con l'analisi dei clienti che hanno speso di più e la durata media dei soggiorni, questa query permette analisi avanzate di comportamente, rendendo possibile distinguere i clienti occasionali, quelli ricorrenti e quelli premium (molte prenotazioni e alta spesa). Si può pensare di offrire ai clienti più attivi offerte personalizzare e programmi fedeltà, e i dati dei clienti abituali possono essere utilizzati per prevedere la domanda.

Si è continuato, quindi, in questa direzione, scrivendo una query che calcoli quali clienti hanno speso di più.
```sql
SELECT 
    c.codice_cliente,
    c.nome,
    c.cognome,
    SUM(p.importo_totale) AS totale_speso
FROM Prenotazione p
JOIN Cliente c
    ON c.codice_cliente = p.cliente_codice
WHERE p.stato_prenotazione IN ('Completata', 'In corso')
GROUP BY c.codice_cliente, c.nome, c.cognome
ORDER BY totale_speso DESC;
```
Successivamente si è anche calcolata la durata media dei soggiorni.
```sql
SELECT 
    AVG(DATEDIFF(p.data_partenza, p.data_arrivo)) AS durata_media_notti
FROM Prenotazione p
WHERE p.stato_prenotazione IN ('Completata', 'In corso', 'Confermata');
```
La durata media del soggiorno permette di identificare il comportamento del clienti: soggiorni brevi indicano prevalentemente clienti che hanno viaggiato per lavoro, mentre soggiorni più lunghi indicato clienti che sono in vacanza. La lunghezza del soggiorno è utile per la pianificazione della turnazione del personale e per la gestione dei check-in e check-out.

Si è poi unito tutto in una query che permettesse di visualizzare i risultati delle precedenti.
```sql
SELECT 
    c.codice_cliente,
    c.nome,
    c.cognome,
    
    COUNT(p.id_prenotazione) AS numero_prenotazioni,

    SUM(p.importo_totale) AS totale_speso,

    AVG(DATEDIFF(p.data_partenza, p.data_arrivo)) AS durata_media_soggiorno

FROM Prenotazione p
JOIN Cliente c
    ON c.codice_cliente = p.cliente_codice
WHERE p.stato_prenotazione IN ('Completata', 'In corso', 'Confermata')
GROUP BY c.codice_cliente, c.nome, c.cognome
ORDER BY totale_speso DESC;
```

Successivamente, ci si è concentrati sull'analizzare le prenotazioni. La prossima query permette di ottenere l'elenco dei clienti con prenotazioni attive. Permette di identificare clienti che devono ancora arrivare e clienti già in struttura.
```sql
SELECT DISTINCT
    c.codice_cliente,
    c.nome,
    c.cognome
FROM Prenotazione p
JOIN Cliente c
    ON c.codice_cliente = p.cliente_codice
WHERE p.stato_prenotazione IN ('In attesa', 'Confermata', 'In corso');
```

Si è poi implementata una query per verificare le prenotazioni di un hotel in un intervallo di date. Questa è utile per l'analisi dell'occupazione, il controllo della domanda e per la pianificazione delle risorse.
```sql
SELECT 
    p.*
FROM Prenotazione p
WHERE p.hotel_codice = :hotel_codice
  AND (
        :data_inizio < p.data_partenza
    AND :data_fine   > p.data_arrivo
  )
ORDER BY p.data_arrivo;
```
`SELECT p.*` restituisce tutte le colonne della tabella `Prenotazione`. `:hotel_codice` è un parametro dinamico.

Successivamente, si è deciso di continuare con l'analisi del fatturato. La prossima query calcola il fatturato totale del sistema.
```sql
SELECT 
    SUM(p.importo_totale) AS fatturato_totale
FROM Prenotazione p
WHERE p.stato_prenotazione IN ('Completata', 'In corso');
```
I risultati di questa query misurano la performance economica globale, utile a monitorare la crescita del sistema e l'andamento del business. Il valore ottenuto può essere utile anche a confrontare i ricavi rispetto ai costi operativi.

Per permettere di andare più nel dettaglio, e quindi di fare un'analisi più granulare, si sono implementate le query per calcolare il fatturato per hotel e il fatturato mensile del sistema.
```sql
-- Fatturato per hotel
SELECT 
    h.codice AS hotel_codice,
    h.denominazione   AS hotel_nome,
    SUM(p.importo_totale) AS fatturato
FROM Prenotazione p
JOIN Hotel h
    ON h.codice = p.hotel_codice
WHERE p.stato_prenotazione IN ('Completata', 'In corso')
GROUP BY h.codice, h.denominazione
ORDER BY fatturato DESC;

-- Fatturato mensile
SELECT 
    YEAR(p.data_arrivo)  AS anno,
    MONTH(p.data_arrivo) AS mese,
    SUM(p.importo_totale) AS fatturato
FROM Prenotazione p
WHERE p.stato_prenotazione IN ('Completata', 'In corso')
GROUP BY YEAR(p.data_arrivo), MONTH(p.data_arrivo)
ORDER BY anno, mese;
```
In particolare, l'analisi del fatturato mensile permette di identificare la stagionalità della domanda, in modo da pianificare promozioni nei mesi deboli e massimizzare i ricavi nei mesi forti.

Su questa base, si è aggiunta una query che permettesse di avere l'andamento storico dei prezzi, ovvero il prezzo medio per notte per mese. Questa query permette analisi storiche e stagionali, supporta le decisioni sulla modifica dei prezzi e sulle strategie di gestione.
```sql
SELECT 
    YEAR(p.data_arrivo)  AS anno,
    MONTH(p.data_arrivo) AS mese,
    AVG(p.prezzo_notte_bloccato) AS prezzo_medio_notte
FROM Prenotazione p
GROUP BY YEAR(p.data_arrivo), MONTH(p.data_arrivo)
ORDER BY anno, mese;
```

Si è poi voluto indagare l'impatto delle cancellazioni sul sistema. La prima query in questo ambito ha l'obiettivo di calcolare il tasso di cancellazione.
```sql
SELECT 
    COUNT(*) AS totale_prenotazioni,
    SUM(stato_prenotazione IN ('Cancellata', 'No-Show')) AS totale_cancellate,
    AVG(stato_prenotazione IN ('Cancellata', 'No-Show')) AS tasso_cancellazione
FROM Prenotazione;
```
L’espressione `stato_prenotazione IN ('Cancellata', 'No-Show') ` restituisce 1 se la prenotazione è cancellata o no‑show, 0 altrimenti. `SUM(...)` somma tutti gli 1, quindi conta quante prenotazioni sono state cancellate. La media di valori binari (0/1) è esattamente la percentuale di valori uguali a 1, quindi il risultato di `AVG(stato_prenotazione IN ('Cancellata', 'No-Show')) AS tasso_cancellazione` è un valore tra 0 e 1, interpretabile come percentuale.

Questo è un valore importante per valutare l'affidabilità della domanda e valutare le politiche di cancellazione. Il tasso di cancellazione è essenziale per decidere se applicare overbooking controllato o definire politiche di cancellazione più rigide. Questo risultato può essere confrontato con le cancellazioni in hotel diversi e con quali clienti effettuano più cancellazioni.

In questo senso sono state pensate le query successive.
```
-- Cancellazioni per hotel
SELECT 
    h.codice AS hotel_codice,
    h.denominazione   AS hotel_nome,
    COUNT(*) AS numero_cancellazioni
FROM Prenotazione p
JOIN Hotel h
    ON h.codice = p.hotel_codice
WHERE p.stato_prenotazione IN ('Cancellata', 'No-Show')
GROUP BY h.codice, h.denominazione
ORDER BY numero_cancellazioni DESC;

-- Cancellazioni per cliente
SELECT 
    c.codice_cliente,
    c.nome,
    c.cognome,
    COUNT(*) AS numero_cancellazioni
FROM Prenotazione p
JOIN Cliente c
    ON c.codice_cliente = p.cliente_codice
WHERE p.stato_prenotazione IN ('Cancellata', 'No-Show')
GROUP BY c.codice_cliente, c.nome, c.cognome
ORDER BY numero_cancellazioni DESC;
```

Infine, l'ultima query è stata pensata per valutare l'impatto economico delle cancellazioni.
```sql
SELECT 
    SUM(p.importo_totale) AS importo_teorico_cancellato
FROM Prenotazione p
WHERE p.stato_prenotazione IN ('Cancellata', 'No-Show');
```
Questo è un indicatore fondamentale per valutare la potenziale perdita di ricavi e la necessità di cambiare alcune policy del sistema. Il risultato può essere confrontato con il fatturato totale e con il tasso di cancellazione per avere una visione completa dell'impatto economico.

### Conclusione 
Il lavoro ha sviluppato un sistema completo di analisi gestionale per il dominio alberghiero, integrando progettazione del database, implementazione dei vincoli, popolamento realistico e un insieme articolato di interrogazioni avanzate. Le query prodotte hanno permesso di esplorare ogni dimensione operativa ed economica del sistema: disponibilità e occupazione delle camere, comportamento della clientela, dinamiche tariffarie, performance economiche e impatto delle cancellazioni. Nel loro insieme, queste analisi dimostrano la coerenza del modello dati, la correttezza dei meccanismi applicativi (trigger, vincoli, calcoli) e la capacità del database di supportare decisioni strategiche basate su informazioni affidabili e strutturate.

## 2. Rete di pubblicazioni scientifiche.
**Traccia:** Un dipartimento di ricerca vuole costruire un knowledge graph delle proprie pubblicazioni. Ogni autore ha nome, cognome, affiliazione, email e area di ricerca. Ogni articolo scientifico è descritto da titolo, anno, DOI e venue di pubblicazione. Gli articoli possono citare altri articoli, essere scritti da più autori e trattare uno o più temi scientifici. I temi sono concetti come machine learning, bioinformatica, database o computer vision. Il grafo deve consentire di analizzare collaborazioni scientifiche, reti di citazione e collegamenti tra temi di ricerca. Gli studenti devono modellare il dominio in modo da poter rispondere sia a domande strutturali sia a domande di esplorazione.


