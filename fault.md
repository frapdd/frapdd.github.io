# <span style="color:white">Fault Tolerance</span>

I fault si classificano in base alla frequenza: 
- Transient: succedono una volta e poi scompaiono
- Intermittent: appaiono e scompaiono senza apparente motivo
- Permanent: rimangono finché non vengono fixati

Per i sistemi distribuiti esiste un'ulteriore classificazione:
- Omission: crash dei processi / perdita di un pacchetto sul canale
- Timing: violazione dei bound temporali (si applica solo a sistemi sincroni)
- Byzantine: un processo salta qualche istruzione o ne aggiunge / i canali consegnano copie dei messaggi, pacchetti corrotti o non esistenti

## 1 Reliable client-server
Nella comunicazione point-to-point, i fault di omission sui canali sono coperti da TCP. I crash dei processi invece devono essere maskati con qualche protocollo. In RPC, se il server risulta irraggiungibile allora si può gestire come eccezione, ma se invece semplicemente non risponde bisogna capire quando considerare il server come crashato. 
Si può implementare un protocollo per cui c'è la sicurezza di consegnare un messaggio "at least once" o "at most once". 

Consideriamo il caso in cui un client richiede al server di svolgere una certa operazione: quando la richiesta viene ricevuta, il server può decidere di mandare un ack prima o dopo lo svolgimento dell'operazione. Per tenere in considerazione il possibile crash del server si possono implementare delle "reissue strategies" per re-inviare la richiesta nel caso il client non riceva l'ack. A lezione abbiamo visto tutte le possibili combinazioni (tutte si posizionano in un punto dello spettro tra "at most once" e "at least once"), ma risultano tutte subottimali. 
Se il protocollo prevede la possibilità di mandare più volte la stessa richiesta, e questa non è idempotente, il server deve memorizzare l'hash della stessa per evitare i duplicati (problema: come gestire questo buffer). 

Se è il client a crashare, la computazione si definisce "orphan". Il problema in questi casi è eliminare i client crashati per evitare spreco di risorse. Per questo abbiamo visto diverse strategie a lezione (omesse qui, non penso siano importanti). 

## 2 Protection against failures
> Sistema sincrono (come se il tempo fosse discreto e suddiviso in time steps)

> I processi possono terminare silenziosamente ad ogni time step

L'idea di base è sfruttare ridondanza per mascherare il crash di processi: ogni operazione è svolta da un gruppo di processi, e non più uno solo, in modo che se uno o più falliscono viene comunque portata a termine. 
Un gruppo può essere flat (tutti alla pari, in comunicazione con tutti) o hierarchical (coordinator gestisce un pool di processi workers). 
Per gestire un gruppo flat: 
> Il protocollo di comunicazione offre un multicast reliable

Bisogna inoltre implementare la rilevazione dei crash e l'eventuale scioglimento di un gruppo nel caso di troppi crash. 
Per decidere il numero di membri facciamo delle assunzioni ragionate del tipo: al 99% 5 processi su 6 rimangono sempre attivi. 

Una delle sfide principali nell'utilizzo di gruppi è il consenso tra i processi (non crashati) di un gruppo. 
Formalizzazione del problema:
- I processi non crashati devono accordarsi su un valore soggetto a una condizione di validità (non può essere un valore qualsiasi)
- Tutti i processi hanno un valore iniziale prima di iniziare la procedura di consenso
- L'algoritmo deve garantire la terminazione
- Se tutti i processi vengono inizializzati con lo stesso valore, allora l'algoritmo deve prevedere la convergenza su quel valore

Hanno dimostrato che non esiste soluzione a questo problema (ne crash ne bizantino) se rimuoviamo l'assunzione del sistema sincrono. 

### 2.1 FloodSet
Algoritmo per il consenso nel caso di crash dei processi. 
> I crash saranno al massimo una quantità f

> Tutti i processi sono a conoscenza di un valore di default v0
 
- Ogni processo mantiene un SET W inizializzato con un valore random
- Per f+1 rounds: ogni processo manda il proprio W in broadcast e poi aggiunge i valori ricevuti a W (aggiunta idempotente per un set)
- Dopo il ciclo: se la cardinalità di W è 1 (caso poco probabile: tutti si sono inizializzati con lo stesso valore) allora si decide per quel valore, altrimenti si decide v0

Può essere ottimizzato dal secondo ciclo del loop in poi inviando in broadcast solo le nuove aggiunte. 

### 2.2 Byzantine generals
Algoritmo per il consenso nel caso di errori bizantini dei processi: i processi possono comportarsi in modo arbitrario.  
Viene modellato sulla storia dei generali che devono concordare la "potenza" di tutti i loro battaglioni. Ogni processo viene inizializzato con un valore e possiede una memoria (un vettore) dove salvare quello degli altri. 
- Ogni processo manda in broadcast il proprio valore
- Ogni processo raccoglie i valori ricevuti in un vettore
- Ogni processo manda il vettore costruito in broadcast
- Ogni processo fa un max() per ogni posizione del vettore

Lamport ha dimostrato che con m processi bizantini si può raggiungere un accordo se ci sono altri 2m+1 processi funzionanti. 

## 3 Reliable group communication
Si tratta di un requisito importante per ottenere resilienza del sistema tramite replicazione, l'obiettivo è ottenere un multicast affidabile. 

### 3.1 Non hierarchical feedback control
> I gruppi sono fissati

> I processi sono funzionanti

> Il multicast all'interno del gruppo funziona

L'ultima assunzione si può implementare con ack positivi (problema: rete sovraccarica) o negativi (mandati se manca un pacchetto, ci si accorge numerando).

L'idea è che ogni nack viene spedito in multicast e quindi tutti gli altri processi ne sono al corrente. Se devono mandare anche loro un nack per la stessa cosa devono prima aspettare la scadenza di un timer di durata random che viene resettato ogni volta che si osserva un nuovo nack. 
Si evita di mandare nack replicati ma ogni processo deve processare tutti i nack, e questo ha un costo. 

### 3.2 Hierarchical feedback control
> I gruppi sono fissati

> I processi sono funzionanti

Questa variante è pensata per quando il numero di processi è elevato. 
I nodi sono organizzati in gruppi gestiti da un coordinatore e organizzati in un albero (che bisogna immaginare con il gruppo del mittente a fare da root). All'interno del gruppo si può adottare il protocollo precedente. Costruire e mantenere questo albero è un problema di questo protocollo. 
Per mandare un messaggio a un esterno, il pacchetto viene instradato tramite i coordinatori. 
> Le connessioni unicast tra coordinatori sono reliable

Ogni coordinatore è dotato di un buffer in cui vengono memorizzati i messaggi in entrata verso il gruppo in uscita verso altri coordinatori. Un messaggio viene rimosso dal buffer quando tutti i ricevitori nel gruppo hanno ricevuto il messaggio o quando tutti i coordinatori figli hanno ricevuto il messaggio (entrambe le cose vengono verificate con ack). 
Un coordinatore può richiedere la ritrasmissione di un pacchetto al suo coordinatore padre se necessario. 

### 3.3 Synchrony
La variante "close synchrony" ha due requisiti: 
- due elementi ricevono gli stessi messaggi multicast o osservano gli stessi cambiamenti nella composizione del gruppo => i due elementi vedono gli stessi eventi nello stesso ordine. 
- gli invii e le ricezioni dovrebbero apparire come un unico evento istantaneo

Tutto questo non può essere ottenuto in presenza di failures, quindi cerchiamo la variante più debole "virtual synchrony":
- I processi che crashano sono esclusi dal gruppo e devono joinnare di nuovo 
- I messaggi dei processi funzionanti sono visti da tutti gli altri processi funzionanti
- I messaggi dei processi che crashano sono visti da tutti i processi funzionanti o da nessun processo
- La group view deve essere consistente con i messaggi multicast osservati, e deve evolvere in modo consistente

Segue che tutti i messaggi devono essere scambiati tra due cambi di view, per far si che questo avvenga i messaggi devono essere virtualmente "atomici" (virtualmente sincroni). 
La virtual synchrony non impone nessun constraint sull'ordinamento dei messaggi multicast provenienti dallo stesso server, richiede solo un global order. 
Possibili strategie: 
- Atomic multicast: garantisce solo global order e atomicità
- FIFO atomic multicast: tutti vedono i messaggi nell'ordine in cui li ha mandati il mittente (magari sono out of order tra di loro, dipende dal mittente)
- Causal atomic multicast: tutti vedono i messaggi nell'ordine causale 

#### 3.3.1 ISIS
> Canali FIFO affidabili
> Mentre si sta processando un cambiamento di gruppo non se ne verifica un altro

Implementazione nota di virtual synchrony. L'idea è che ogni processo tiene un buffer con i messaggi inviati finchè non viene ricevuto un ack (messaggio di flush) dai destinatari. 
Ogni volta che un processo riceve la notifica di un cambiamento nel gruppo, smette di inviare messaggi e aspetta di aver ricevuto gli ack da tutti (aspetta che il buffer sia vuoto) per applicare il cambiamento alla propria view. 


## 4 Distributed Commit
Modelliamo il commit con "decidere 1" e l'abort con "decidere 0". 
Un protocollo di commit distribuito deve rispettare 3 caratteristiche:
- Agreement: tutti i processi prendono la stessa decisione finale
- Validity: se tutti i processi hanno deciso 1 per sè, allora la decisione finale deve essere 1, altrimenti 0. 
- Termination: se non ci sono crash tutti i processi concordano (debole) o tutti i processi non crashati concordano (forte). 

### 4.1 Two-phase commit 
Questo protocollo richiede un coordinatore. 

Macchina a stati del coordinatore: 
- Init: quando è il momento, manda "vote request" a tutti e passa in Wait
- Wait: se tutti mandano "vote commit" manda un "global commit" a tutti e passa in Commit, ma se ricevi anche solo un "vote abort" allora manda a tutti un "global abort" a passa in Abort
- Abort: stato finale --> transazione abortita
- Commit: stato finale --> transazione committata 

Se un votante non risponde, dopo un timeout assume che abbia votato abort. 

Macchina a stati del processo semplice: 
- Init: quando ricevi "vote request" decidi tra committare (manda "vote commit" e spostati in Ready) o abortire (manda "vote abort" e spostati in Abort)
- Ready: se ricevi "global commit" vai in Commit, se ricevi "global abort" vai in Abort. In ogni caso manda un ack al coordinator
- Abort: stato finale --> transazione abortita 
- Commit: stato finale --> transazione committata

Se il coordinator non manda la richiesta di voto, dopo un timeout il processo abortisce. 
Se il coordinator crasha prima di dare la comunicare globale, i processi provano a decidere tra di loro chiedendosi gli stati tra di loro. 
- almeno un Init: abort
- tutti in Ready: aspetta che ritorni attivo il coordinator ("blocking protocol")

### 4.2 Three-phase commit 
Questo protocollo cerca di risolvere i problemi della versione a due fasi, rendendolo "non-blocking". 

Macchina a stati del coordinatore: 
- Init: quando è il momento, manda "vote request" a tutti e passa in Wait
- Wait: se tutti mandano "vote commit" manda un "prepare commit" a tutti e passa in Precommit, ma se ricevi anche solo un "vote abort" allora manda a tutti un "global abort" a passa in Abort
- Abort: stato finale --> transazione abortita
- Precommit: Quando tutti mandano "ready commit" manda a tutti "global commit" e vai in Commit
- Commit: stato finale --> transazione committata 

Se un votante non risponde alla vote request, dopo un timeout assume che abbia votato abort. Se un votante non manda il ready commit allora prosegue lo stesso (quando tornerà attivo verrà committato).

Macchina a stati del processo semplice: 
- Init: quando ricevi "vote request" decidi tra committare (manda "vote commit" e spostati in Ready) o abortire (manda "vote abort" e spostati in Abort)
- Ready: se ricevi "prepare commit" vai in Precommit e manda "ready commmit". Se ricevi "global abort" vai in Abort e manda un ack
- Abort: stato finale --> transazione abortita 
- Precommit: Quando ricevi "global commit" vai in Commit e manda un ack
- Commit: stato finale --> transazione committata

Se il coordinator non manda la richiesta di voto, dopo un timeout il processo abortisce. 
Se il coordinator crasha prima di dare la comunicare globale, i processi provano a decidere tra di loro chiedendosi gli stati tra di loro:
- almeno un Init: abort
- almeno un Commit: commit
- almeno un Abort: abort
- maggioranza di Precommit: commit (??)
- maggioranza di Ready: abort

## 5 Recovery techniques
Ci sono due approcci per ritornare a uno stato corretto:
- Forward: vai a un nuovo stato corretto 
- Backward: ritorna a uno stato backuppato. Serve un sistema di checkpointing periodico (costoso) o di log dei messaggi (da ripetere in caso di recovery per ricostruire lo stato). Nella pratica si fa un backup una volta ogni tanto e nel mentre si logga, trade-off. 

### 5.1 Independent checkpointing
Differentemente dallo snapshot distribuito, qua ogni processo salva il proprio stato indipendendentemente periodicamente (ad ogni checkpoint viene assegnato un numero che si incrementa man mano). Trovare un checkpoint significa trovare un taglio consistente. 
Siccome tutti vanno per conto proprio, non posso cancellare i vecchi checkpoint quando ne faccio di nuovi (differentemente da come si fa con gli snapshot) perché potrebbe essere necessario un vecchio checkpoint per trovare un taglio consistente. 
Nell'implementazione di un sistema del genere, si aggiunge ad ogni messaggio scambiato un timestamp (numero progressivo del checkpoint del mandante) per tenere conto delle dipendenze. 

Quando c'è bisogno di un recovery, tutti i processi stoppano mandano le proprie dipendenze a un processo apposito che ricostruisce il grafo delle dipendenze, trova il taglio e manda le istruzioni di rollback a tutti gli altri processi. 
Ci sono due algoritmi (centralizzati) per trovare il taglio a partire dal grafo:
- Rollback-dependency
- Checkpoint-dependency

Entrambi sono algoritmi grafici spiegati da 1:36:00 qui:
https://politecnicomilano.webex.com/recordingservice/sites/politecnicomilano/recording/6fa3ab0e0735103ab7fd005056829580/playback

### 5.2 Coordinated checkpointing
L'idea è svilupare una versione che limiti il numero di checkpoint inutili. Esistono diverse possibili soluzioni:

- Semplice: un coordinatore chiede periodicamente richiede a tutti di salvare un checkpoint (mettendo i messaggi in entrata in coda per il momento). Quando tutti i processi hanno mandato un ack per dire che hanno finito, il coordinatore comunica a tutti che il checkpoint è finito e si riprende il normale funzionamento
- Incrementale: un coordinatore chiede periodicamente checkpoint solo quando un processo diventa (magari anche solo indirettamente) dipendente dal coordinatore
- Snapshot globale: ha il vantaggio di lasciare eseguire i processi per tutto il tempo, ma è un algoritmo più complesso
- Communication-induced: nei messaggi vengono inserite delle informazioni che ogni processo elabora per capire se è venuto il momento di fare un checkpoint

### 5.3 Logging
> Il sistema è "piecewise deterministic", cioè esegue in modo deterministico tra un messaggio e l'altro

Per implementarlo bisogna aggiungere all'header del messaggio tutte le informazioni necessarie per fare il replay. 
Ogni processo classifica internamente ogni messaggio come stabile (in quanto scritto su memoria stabile permanente) o instabile. Per ogni messaggio instabile "m" vengono mantenuti due set: 
- Dep: tutti i processi destinatari di m (e quindi dipendenti) o i processi destinatari di un messaggio dipendente da m
- Copy: tutti i processi che hanno m in memoria (non stabile), potrebbero servire se si perde m

In seguito a uno o più crash, un processo si definisce orfano quando dipende da un messaggio m e tutti gli elementi di Copy(m) hanno crashato. In questo caso non c'è modo di recuperare m, e quindi il processo orfano va killato (con tutte le conseguenze del caso). Ogni processo dovrebbe mantenere una copia di ogni messaggio da cui dipende. 

Esistono due approcci al logging: 
- Pessimistico: ogni volta che ricevo un messaggio devo scriverlo in memoria stabile prima di poterne mandare un altro. Assicurazione di non creare orfani. 
- Ottimistico: il salvataggio è asincrono. Se tutti i processi in Copy(m) crashano, allora si rollbacka in uno stato in cui non c'è più la dipendenza da m. Ci sono gli orfani ma vengono costretti a rollbackare finché non lo sono più. 
