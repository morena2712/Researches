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

#### Creazione del DataBase e delle Tabelle
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
SPIEGA QUI.

Si procede poi a visualizzare le prima 20 righe di tutte le tabelle.
```sql
SELECT * FROM Hotel LIMIT 20;
SELECT * FROM Camera LIMIT 20;
SELECT * FROM Cliente LIMIT 20;
SELECT * FROM Prenotazione LIMIT 20;
SELECT * FROM Tipologia;
```

### Query A: Clienti Top Spenders
Questa analisi individua i clienti che hanno generato il maggior fatturato per la catena.

```sql
SELECT 
    c.codice_cliente,
    c.nome,
    c.cognome,
    SUM(p.importo_totale) AS totale_speso
FROM Prenotazione p
JOIN Cliente c ON c.codice_cliente = p.cliente_codice
WHERE p.stato_prenotazione IN ('Completata', 'In corso')
GROUP BY c.codice_cliente, c.nome, c.cognome
ORDER BY totale_speso DESC;
