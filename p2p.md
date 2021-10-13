# <span style="color:white">Peer 2 Peer</span>

Paradigma/stile architetturale che promuove la condivisione di risorse e servizi tramite interazioni diretti tra pari: processing, storage, network bandwidth e dati.
Una overlay network è una rete logica di connessioni tra peer, un'astrazione sulla rete fisica in cui gli archi sono "virtual links". Alcuni protocolli cercano di tenere un legame tra prossiità fisica e logica, ma non per forza. 
 
> i nodi vanno e vengono in modo imprevedibile

> le capacità dei nodi sono molto variabili

> scala geografica molto ampia. 


## 1 Retrieving resources
> per scoprire una overlay network si utilizza un protocollo esterno 

Cercare in modo efficiente i nodi che possono rispondere a una certa richiesta. 
Ci sono due varianti: 
- Search: cercare tutte le risorse con una certa proprietà
- Lookup: cercare una risorsa specifica tramite id
La ricerca può restituire i dati direttamente (solo lookup) o meno (reference alla risorsa). 

### 1.1 Centralized search (Napster)
I dati rimangono solo sui peer, ma la ricerca è centralizzata. Dopo che la ricerca è stata effettuata, i dati vengono scambiati direttamente tra peer. Il server centrale mantiene un mapping risorse-nodi, bisogna contattarlo per pubblicare qualcosa. Grazie a questa centralizzazione della ricerca si può implementare qualche forma di search (aggiungi metadati ai file durante la pubblicazione). 
Ha il vantaggio di essere semplice ed efficiente per la ricerca, ma il server deve mantenere uno stato pesante ed è single point of failure. 

### 1.2 Query flooding (Gnutella)
I nuovi nodi si connettono in un punto random (anchor node) di una overlay network: per robustezza la richiesta di join viene propagata dall'anchor node ad altri nodi che rispondono se hanno spazio e si connettono al nuovo. 
Ogni ricerca viene inizialmente inviata solo ai vicini che se non sanno rispondere propagano ulteriormente (flooding). 
Si può fare search anche meglio che con il sistema centralizzato, perché ognuno può valutare meglio la query con i propri dati (non servono per forza i metadati). 
Ha il vantaggio di essere semplice e totalmente distribuito, ma il traffico va gestito: la propagazione viene limiatata dopo un tot di hops, questo introduce trade-off tra garanzia di ricerca e performance. Il tempo di ricerca è molto maggiore che nel centralizzato, e magari non ottieni la risorsa (troppo lontana o nodi disconnessi). 

### 1.3 Hierarchical topology (Kazaa)
Questo metodo è una via di mezzo tra gli estremi dei primi due. 
Si basa su una gerarchia: ci sono super-nodes per la ricerca e nodi normali con dati (ogni supernode gestisce un sottoinsieme dei nodi normali). Il flooding avviene solo tra i super nodes, che esternamente appaiono come un sistema centralizzato. La mappa risorsa-nodo può essere distribuita nell'overlay di supernodes ma non è detto. 
Può gestire il trade-off tra performance (tiene in considerazione eterogeneità tra i nodi) e garanzie di ricerca. 

## 2 Downloading resources (BitTorrent)
> per trovare i nodi che hanno una risorsa si utilizza un protocollo esterno

L'idea è suddividere il file in chunks e scaricarli da diversi nodi (usi la bandwidth di tutti). Non appena hai qualche chunk puoi contribuire alla rete (risorve il problema dei "free riders"). Viene data la priorità ai chunk rari per mantenere la disponibilità. 
Ogni nodo ha un numero limitato di connessioni entranti, la scelta di chi far connettere viene fatta dinamicamente in base alla vista locale (faccio connettere chi ha contribuito molto a me). L'effetto globale è conveniente per tutti. 
Ogni tanto le connessioni vengono date a random per permettere ai nuovi nodi di entrare nella rete. All'inizio la performance in download è bassa e poi cresce quando si diventa dei contributori.
Problema: on ci sono bounds sull'efficienza del servizio. 

## 3 Secure storage (Freenet)
> per scoprire l'overlay network si utilizza un protocollo esterno

L'idea è creare un internet non centralizzato senza censure e anonimo. Quando ti connetti offri dello storage che conterrà una parte dei file della rete (criptati). Quando pubblichi un file, sai che viene distribuito sulla rete in qualche modo. Quando cerchi un file, questo viene propagato ulteriormente lungo il path (favorisce le risorse popolari), non c'è connessione diretta tra i due nodi. Un file che non viene mai cercato viene cancellato dopo un po'. 
Oltre a propagare queries, ora anche risorse. Per gestire questo, ogni nodo mantiene informazioni limitate sulla mappa risorse-nodi e sul routing tra nodi (non c'è bisogno di flooding). 

    - ricevo una query: mando una risposta se possiedo la ricerca o conosco chi la possiede
    - ricevo la risposta a una query: memorizzo la mappa risorsa-nodo e procedo con la ricerca del nodo
    - devo cercare un nodo: inizio a chiedere dai nodi che hanno una chiave simile a quella che sto cercando (probabilisticamente conveniente)
    - ricevo una richiesta di routing: rispondo direttamente affermativamente o propago la richiesta. Durante questo processo aggiorno anche le mie informazioni sulla rete (mappa e routing). 
    
Questa struttura si sviluppa naturalmente in modo che dopo un po' certi nodi diventano più informati, e alla fine le richieste passeranno per lo più attraverso questi pochi nodi rilevanti. Le risorse si sposteranno verso l'area in cui sono più richieste. 
Non ci sono ancora bounds sulle garanzie del servizio. 

## 4 Blockchains (Bitcoin)
Sono protocolli di consenso non centralizzati. La risorsa condivisa è potenza computazionale, i dati distribuiti costituiscono un log (diviso in blocchi consecutivi) che tipicamente memorizzano transazioni. 
I nodi validano le transazioni consultando il log, per farlo devono risolvere un problema complicato da risolvere ma facile da verificare. I nodi competono per risolvere il problema, con l'incentivo di poter aggiunere una transazione-premio personale al log. I blocchi sono considerati affidabili quando sono seguiti da un po' di blocchi, per questo vengono tenute diverse versioni parallele del log finché non si possono scartare in quanto troppo corte. 
Funziona quando la potenza computazionale è ben distribuita sulla rete. 
