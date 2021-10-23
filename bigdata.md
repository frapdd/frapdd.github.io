# Big Data

Trattiamo i sistemi distribuiti per supportare applicazioni basate su big data. 
Caratteristiche dei dati trattati (4 V):

- Volume: l'approccio di solito è di mantenere i dati di default (per eventuali ricerche future), questo fa esplodere le dimensioni.
- Velocity: i dati potrebbero arrivare come stream e bisogna processarla in fretta perché il valore dipende dal tempo
- Variety: nessun constraint sul formato (unstructured data)
- Veracity: i dati potrebbero non essere corretti, e dobbiamo essere in grado di escluderli

Un sistema distribuito del genere deve quindi essere molto scalabile, flessibile per supportare veloci cambiamenti nei dati e supportare sistemi per paralelizzazione e distribuzione automatica. 

## 1 MapReduce
Si tratta di un algoritmo molto generale per data analysis su larga scala, semplice dal punto di vista del programmatore (anche se molto limitato) in cui la complessità del sistema distribuito è mascherata da una piattaforma sottostante che implementa un database distribuito. 

Molti problemi si possono modellare come una computazione a livello di un grafo, esempio: calcolare per ogni punto d'interesse la gas station più vicina, modellando le distanze come edge di un grafo.  
Il problema è che in un sistema distribuito non conosciamo tutto il grafo (ognuno ne conosce una parte e magari non abbiamo abbastanza memoria per memorizzarlo tutto in un nodo per applicare algoritmi centralizzati), quindi le soluzioni classiche di DP non si possono applicare direttamente. 

Seguendo l'esempio delle gas station e dei punti di interesse, vediamo i due step dell'algoritmo standard: 

**Map**: dividiamo il grafo considerando l'intorno di ogni gas station e misuriamo il path minimo tra ognuna e i punti di interesse nel proprio intorno. I nodi che si occupano delle computazioni di ogni intorno operano indipendentemente tra di loro. 

Alcuni punti di interesse appartengono a più intorni, e per quelli bisognerebbe calcolare l'opzione ottimale (la minima), ma questo richiede comunicazione. La soluzione sta nel secondo step. 

**Reduce**: i risultati di ogni blocco vengono suddivisi per punto di interesse e distribuiti ai nodi del secondo step: tutti i dati relativi a un determinato punto di interesse verranno ricevuti dallo stesso nodo che potrà così applicare la funzione di reduce, che in questo caso è min(). Ogni punto di interesse può essere elaborato in parallelo rispetto agli altri. 

Idea chiave dell'algoritmo: piuttosto che uno stato globale mutabile, per ottenere l'informazione operiamo trasformazioni di dati immutabili. 

Lo sviluppatore scrive le funzioni assumendo di vedere un solo dato alla volta, per permettere l'implementazione su un database distribuito. Quindi sia la funzione map che quella reduce devono utilizzare iteratori. 

Per la funzione map, bisogna descrivere una funzione stateless per elaborare la visione locale dei dati. Per la funzione di reduce bisogna specificare che operazione svolgere sulla lista di valori associati alla stessa chiave. 

La "piattaforma" sottostante al sistema di mapreduce si occupa del database distribuito, che deve essere implementato efficientemente (elementi con la stessa chiave sono memorizzati in aree continue di memoria, data locality) e affidabile. La piattaforma si occupa anche di spostare i dati dai mappers ai reducers.

C'è un master che gestisce tutta la procedura. All'inizio usa tutti i computer per il mapping, quando tutti hanno finito (punto di sincronizzazione: limitazione), allora sa che i risultati sono sul distributed file system e si può iniziare la parte di reduce, per cui usa di nuovo tutti i computer. Se sospettiamo un failure, scartiamo la computazione e ricominciamo su un altro nodo. Viene implementato un sistema di ping continui (heartbeats) per monitorare la situazione. 

L'hardware del sistema può essere eterogeneo, quindi facilmente esistono nodi più lenti di altri (stragglers). Questo fa si che si creino situazioni in cui mi rimangono nodi liberi mentre alcuni finiscono di computare: posso replicare il task sui nodi liberi che magari sono più veloci. 

## 2 Modern MapReduce variants

L'algoritmo originale si può generalizzare:

- Due step --> Grafo aciclico di trasformazioni (set di primitives disponibili allo sviluppatore) con anche loops
- Batch processing --> Stream processing
- Memoria di massa --> Main memory: grazie allo sviluppo delle memorie, ora ci si può permettere di spostare i dati nella RAM che ha un accesso più veloce. 

Vediamo due applicazioni di riferimento. 

### 2.1 Apache Spark
Simile al MapReduce tradizionale. 

Spark aggiunge il concetto di stage: blocco di operazioni (non solo map e reduce) consecutive: non richiedono ricombinazione dei dati tra di esse. Tutta l'infrastruttura processa uno stage alla volta e salva il risultato per il prossimo stage. Questo approccio si chiama "scheduling of tasks". 

Può supportare dati statici o streaming grazie all'approccio micro-batch: accumula un po' di dati da mandare poi in esecuzione. Questo richiede uno stato che persiste attraverso le diverse batch per rendere il processo incrementale.

Quando possibile viene utilizzata la main memory: priorità ai nodi che hanno i dati in main memory, rispetto a quelli che li hanno su disco. 

*Latency:* alta in quanto devo valutare il tempo necessario a riempire una micro-batch e il delay dello scheduling. 

*Throughput:* comparabile a Flink, ma la possibilità di fare scelte a runtime con lo scheduling può migliorare la performance. 

*Load balancing:* stesso discorso di throughput. 

*Elasticity:* quando il servizio scala il prezzo automaticamente in base al load, lo scheduling funziona meglio in quanto le decisioni vengono prese a runtime.

*Fault tolerance:* se c'è qualche fail, reschedule dell'operazione.  Posso recuperare i dati in quanto tutto si basa su una sequenza di funzioni deterministiche a partire dall'input. Ogni tanto vengono salvati i risultati intermedi, nel caso peggiore riparto dall'input (sempre salvato in qualche memoria stabile).

### 2.2 Apache Flink
Flink è stato pensato per streaming, basato sul concetto di pipelining. 

I processi (operatori) di computazione necessari vengono istanziati nel momento in cui diventano necessari. Quando è possibile, sfrutti il concetto di pipeline tra i processi correntemente attivi. Tpicamente il sistema instanzia un po' di processi per ogni stage e si occupa del load balance. Non mi serve uno storage intermedio perché i dati fluiscono (tramite TCP channels) da uno stage all'altro.

Se i dati non sono di tipo streaming, vengono streammati.

*Latency:* minima in quanto il dato viene processato non appena è disponibile. 

*Throughput:* comparabile a Spark. 

*Load balancing:* stesso discorso di throughput. 

*Elasticity:* solo soluzioni naive per gestire lo scaling: snapshot di tutto il sistema periodico ed eventuale restart con più risorse. 

*Fault tolerance:* di default non vengono salvati i risultati intermedi. Assumo ancora che l'input sia salvato in modo stabile da qualche parte. Si può ripetere anche qua tutto il flusso di operazioni, ma per ottimizzare viene implementata una forma di checkpointing su disco. 
