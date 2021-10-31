# Communication

I sistemi distribuiti si basano spesso su middleware: un livello intermedio tra transport e application layer nell'ISO OSI. Un middleware di solito offre soluzioni per: naming, security, scaling, marshalling (conversione tra diverse convenzioni) e comunicazione. 

In generale, le strategie di comunicazione possono implementare o meno queste due proprietà: 

- Transient: entrambi i nodi coinvolti nella comunicazione sono attivi, l'alternativa è un sistema data-centric in cui uno dei due nodi è passivo
- Synchronous

## 1 Remote procedure call

Si tratta di una tecnica di comunicazione per mascherare la complessità necessaria a invocare una funzione su una macchina remota, in modo da permettere agli sviluppatori di scrivere codice senza considerare la distribuzione del sistema. 

Il sistema viene implementato con uno scambio di messaggi: uno che contiene i dettagli dell'invocazione, e uno in risposta che contiene l'output della funzione. I parametri (se strutturati vengono serializzati automaticamente) vengono aggiunti al primo messaggio, che viene costruito sulla base della signature della funzione chiamata. 

Le signature vengono definite nel "Interface Definition Language", un formato di alto livello per cui esistono i mapping rispetto ai vari linguaggi di programmazione. Questo serve per permettere l'interazioni tra processi programmati con linguaggi diversi. 

I passaggi di reference spesso non sono supportati. Nel caso degli array si può implementare un sistema di passaggio value/result, che permette di raggiungere lo stesso risultato di un passaggio by reference. Tra due processi che runnano sulla stessa macchina posso utilizzare una versione semplificata di RPC che exploita il fatto di condividere la main memory: un processo scrive i parametri sullo stack poi con un context switch passa il flow di esecuzione all'altro processo. Il secondo processo computa la funzione e prima di ritornare il flow di esecuzione copia sullo stack il risultato perché sia visibile al primo processo. 

### 1.2 Asynchronous RPC

Di default RPC mima le invocazioni di funzioni in locale, sospendendo il chiamante fino al termine dell'esecuzione della funzione. Questo è uno spreco di risorse e di potenziale parallelismo, ancora più grave se si considerano i casi di procedure void. 

Esistono diversi approcci per affrontare la questione, in generale: 

- Per procedure void si attende solo un ack dal server prima di riprendere l'esecuzione. 
- Per procedure non void, quando i risultati sono pronti è il callee a invocare nuovamente il caller. Alternativamente, il callee ritorna immediatamente un "future" utilizzato in un secondo momento dal caller per ottenere il risultato. 

### 1.2 Open Network Computing RPC

Implementazione standard di Sun Microsystem. Il sistema in questione offre RPC ma con alcune limitazioni: solo passaggi per copia, un solo parametro in input, un solo output. 

Il problema del binding tra caller e callee viene affrontato introducendo un processo portmap:

- I server segnalano a portmap su quali porte rendono disponibili quali servizi
- I client contattano portmap richiedendo i servizi e ricevono l'indirizzo + porta del server 

Esiste un portmap locale per ogni rete, e il suo indirizzo deve essere noto a propri ai processi. 

Questo sistema offre un servizio di dynamic activation dei server, facendo passare tutte le richieste attraverso un daemon locale che controlla se esiste un server già attivo a cui inviare la richiesta o se è necessario accenderne uno. Chiaramente questo diminuisce molto la performance per le richieste che devono accendere il server (come la prima), ma rende più efficiente il sistema nel complesso. 

Inoltre, questa implementazione permette di operare in maniera batched: tutte le chiamate void vengono accumulate in un buffer e mandate in ordine tutte assieme quando si rende necessaria una chiamata non void. Si può considerare una forma si RPC asincrono. 

### 1.3 Distributed Computing Environment 

Implementazione standard di Open Group che offre molti servizi basati su RPC (distributed time, distributed filesystem). 

In modo analogo al portmap, il client si deve interfacciare con un directory service (in qualche modo raggiungibile a priori) che questa volta può essere distribuito per migliorare la scalability del sistema.

## 2 Remote method invocation

Equivalente di RPC per OOP, quindi molto simile. L'idea è che tra nodi diversi ci si scambia proxy degli oggetti, in modo da evitare trasporto di codice. La complessità viene mascherata al client, il proxy si occupa di tutte le operazioni necessarie per il passaggio di parametri (come la serializzazione). 

Come in RPC, viene utilizzato un IDL per la descrizione degli oggetti. 

Le due implementazioni più famose sono: 

- Java RMI: entrambi i processi coinvolti nella comunicazione utilizzano lo stesso linguaggio nella stessa macchina virtuale, quindi gli oggetti possono essere passati non solo by reference ma anche via copy, in quanto il passaggio di codice è sicuro. 
- Corba: sistema di RMI che permette l'interazione tra processi scritti in linguaggi diversi. Gli oggetti possono essere mandati via copia ma la compatibilità tra i due comunicanti deve essere curata dal programmatore.  

Entrambe le implementazioni permettono il passaggio by reference o by copy di tipi arbitrari di dati. 

## 3 Message-oriented communication

L'idea è rendere la comunicazione tra processi asincrona, meno rigida e non point-to-point. 

Classificazioni di comunicazione transient (entrambi devono essere attivi) tra due processi A (client) B (server):

- Pure asynchronous: A manda un messaggio a B e continua l'esecuzione senza aspettare una risposta. B processa la richiesta non appena può
- Receipt-based synchronous: A manda un messaggio a B e si ferma fino a che non riceve un ack. B manda l'ack non appena riceve il messaggio ma magari processa la richiesta più tardi
- Delivery-based synchronous: A manda un messaggio a B e si ferma fino a che non riceve un ack. B manda l'ack non appena inizia a processare la richiesta 
- Response-base synchronous: A manda un messaggio a B e si ferma fino a che non riceve un ack. B manda l'ack non appena ha terminato di processare la richiesta

Classificazioni di comunicazione persistent (non devono essere entrambi attivi) tra A (client) e B (server): 

- Asynchronous: A manda un messaggio a B e continua l'esecuzione. B riceve il messaggio non appena si attiva
- Synchronous: A manda un messaggio a B e aspetta un ack. B manda un ack anche se non è attivo, poi processerà il messaggio non appena si attiverà

### 3.1 Message passing (MPI)

Modello di riferimento per la comunicazione via messaggi, praticamente una versione ad alto livello e protocol-independent delle socket datagram UDP multicast (scambio di dati simmetrico, senza connessione e potenzialmente con più entità). Essendo più ad alto livello, non è richiesta la serializzazione dei dati come con le socket. 

La comunicazione è organizzata per gruppi di processi: ogni messaggio ha destinazione (e sorgente) composti da una tupla processID/groupID. 

Può rientrare in più classificazioni tra quelle sopra, a seconda delle primitive utilizzate. 

 ### 3.2 Message queuing

Comunicazione point-to-point, persistent e asynchronous che segue un approccio data-centered. Ogni entità coinvolta (peer to peer) ha una coda in entrata e una in uscita. 

Garantisce il massimo livello di decoupling della comunicazione (sia nel tempo che nello spazio), e garantisce solo l'effettivo invio del messaggio nella coda in entrata del destinatario. 

Viene implementata nel Message Oriented Middleware con 4 primitive di base: put, get (blocca finché coda non-vuota e prendi il primo), poll (come get ma senza blocco), notify (chiama quando arriva un messaggio nella coda specificata). 

Questo approccio fitta bene con il paradigma client-server:

- I client non devono rimanere connessi, mandano solo il messaggio nella coda del server
- Il load balancing è più facile: coda condivisa tra più server uguali, si distribuiscono i messaggi tra di loro

Le code sono identificate da nomi simbolici, quindi serve un sistema di lookup distribuito. In general i messaggi viaggiano su una overlay network di queue managers (routers) per raggiungere la coda in entrata del destinatario. Questo sistema permette comunicazione multicast a un livello più alto di ip. 

Le architetture che implementano questa stategia devono eventualmente prevedere "message brokers", cioè nodi che covertono i messaggi nella comunicazione tra due processi non direttamente compatibili. 

### 3.3 Publish/subrscribe

Approccio event-based alla comunicazione via messaggi, transient asynchronous, multipoint. Ogni entità può pubblicare notifiche di propri eventi (come messaggi) e ricevere le notifiche degli eventi a cui è iscritta: le primitive sono solo due: publish, subscribe. Il sistema prevede dei componenti chiamati "event dispatcher" che raccolgono le iscrizioni e distribuiscono le notifiche degli eventi agli iscritti; spesso questo componente viene implementato in maniera distribuita perché non diventi il bottleneck del sistema. 

Il "subscription language" determina l'espressività dell'iscrizione a un evento: 

- topic-based: ogni notifica di evento deve avere un oggetto (nel senso della mail) scelto tra un set noto a priori. I subscriber si iscrivono ai topic
- content-based: le iscrizioni sono filtri sul contenuto delle notifiche. Un singolo evento potrebbe matchare subscription di tipo diverso

Diverse strategie di forwarding delle notifiche basate su overlay network in cui i nodi sono message brokers (implementazione distribuita dell'event dispatcher): 

- Message forwarding / acyclic graph: i messaggi vengono propagati su tutta la rete, ogni nodo gestisce un sottoinsieme delle subscriptions. Adatto a sistemi con molte subs e pochi messaggi
- Subscription forwarding / acyclic graph: ogni nodo sa in che direzione mandare il messaggio per raggiungere le subs interessate (path/filtri costruiti man mano che arrivano le subs) e quindi può far percorrere al messaggio il numero minimo di hop. Adatto a sistemi con poche subs e molti messaggi
- Hierarchical forwarding / acyclic graph: la rete ha una root, verso cui fluiscono tutti i messaggi. Se durante il percorso qualche ramo ha una sub interessata, il messaggio si sdoppia e scende lungo quel ramo. Efficienza intermedia rispetto ai due precedenti
- DHT-based / cyclic graph: l'idea è sfruttare il routing delle hash per implementare topic-based subscription ( hash(topic) = id). In ogni nodo della hash vengono salvate le subs, così quando viene emesso un evento bisogna solo accedere al nodo corrispondente e mandare la notifica a chi è nella lista (sulle slide c'è scritta una roba un po' diversa ma così è come l'ha spiegata a lezione)
- Per-source-forwarding / cyclic graph: l'idea è sfruttare (leverage) i cicli per ottimizzare la distribuzione delle notifiche. Quando un nodo deve diffondere la notifica di un evento usa uno shortest-path-tree verso i nodi interessati. I messaggi vengono cioè guidati da delle forwarding table in ogni nodo. Questo sistema supporta content-based subs

Un evoluzione di questi sistemi è l'approccio Complex Event Processing, che permette di definire eventi complessi a partire da una composizione di eventi semplici. 

## 4 Stream-oriented communication

Con questo tipo di dato il fattore tempo determina la correttezza del sistema, non solo la sua perfromance. 

Uno stream di dati può essere semplice o complesso (composto da più stream semplici associati). La sincronizzazione potrebbe avvenire sul receiver o sul sender. 

I tipi di trasmissione sono: 

- Asynchronous: i pacchetti vengono spediti semplicemente uno dopo l'altro in ordine senza altri constraint
- Synchronous: esiste un timeout oltre il quale il pacchetto viene skippato (max jitter)
- Isochronous: viene aggiunto anche un bound inferiore (min jitter) al tempo di ricezione del pacchetto

La performance di questi sistemi si misura secondo canoni di Quality Of Service: maximum delay to setup the session, maximum end-to-end delay etc etc. Il protocollo IP non basta per queste garanzie (best effort protocol), quindi vanno implementate strategie a livelli più alti: 

- Buffering: trade-off tra max jitter e setup-time
- Forward error correction: se un pacchetto arriva corrotto posso provare a ricostruirlo con un semplice checksum. Evita di richiedere il pacchetto (backward correction) per piccoli errori
- Interleaving data: i pacchetti mandati contengono frames non consecutivi, così che se si perde un pacchetto ci sono un po' di buchi piccoli nello stream invece che un buco grande
