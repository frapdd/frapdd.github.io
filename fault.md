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

Se è il client a crashare, la computazione si definisce "orphan". Il problema in questi casi è eliminare i client crashati per evitare spreco di risorse. Per questo abbiamo visto diverse strategie a lezione:

## 2 Protection against failures

## 3 Reliable group communication

## 4 Distributed Commit

## 5 Recovery techniques
