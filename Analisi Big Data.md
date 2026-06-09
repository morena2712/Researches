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

### 3. Analisi dei Dati e Query SQL
Una volta definito il modello concettuale tramite diagramma ER, si è passati alla sua implementazione in SQL. Si è usato DBeaver come client per la scrittura e l'esecuzione del codice, utilizzando il server MySQL.

Il codice che segue rappresenta quindi la concretizzazione operativa del modello ER: ogni scelta sintattica e strutturale è finalizzata a preservare i vincoli concettuali, garantire coerenza dei dati e supportare query efficienti.

#### Creazione del DataBase
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

Successivamente, si è popolata la tabella. Per creare 100 camere si sono innanzitutto create 100 righe vuote con... CONTINUA DA QUI



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
