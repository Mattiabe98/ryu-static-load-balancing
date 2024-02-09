# Ryu OpenFlow static load balancing project for PoliMi [BONSAI Lab](https://www.bonsai.deib.polimi.it/)

## Caratteristiche degli host e della rete 

![image](https://github.com/Mattiabe98/ryu-static-load-balancing/assets/49247389/c0996be1-ef06-4a92-aa1f-1e472a79f13e)

Gli host sono dispositivi virtuali realizzati con Mininet, connessi alla rete realizzata mediante un unico 
switch. Non abbiamo bisogno dell’argomento “--arp” di Mininet dato che il controller gestisce tutte le 
richieste ARP tramite proxy.


I client possono possedere qualsiasi indirizzo IP della subnet 10.0.0.0/8 ad eccezione del range 10.0.1.0/24, 
il quale è dedicato ai server. 
Per quanto riguarda gli indirizzi MAC dei client non vi è alcuna restrizione, mentre gli indirizzi MAC dei 
server devono essere progressivi. I server inoltre devono essere connessi in modo progressivo dalla porta 
numero uno in poi. 

## Load Balancer 

Il load balancer deve possedere un indirizzo MAC e un indirizzo IP prestabilito e deve conoscere la quantità 
di server disponibili. L’indirizzo IP deve essere presente nella stessa subnet dei client e server. 
È possibile impostare quanto detto dichiarando queste variabili nel codice Python. 

``VIRTUAL_IP = '10.0.1.100' 
VIRTUAL_MAC = '00:00:00:00:01:00' 
SERVER_NUMBER = M ``

Dove il valore “M” va sostituito con il numero di server scelti. 

## Obiettivo primario del progetto 
Il controllore e lo switch devono lavorare sinergicamente per bilanciare il carico sulla rete senza 
memorizzare alcuno stato. 
In particolare, ogni volta che riceve una nuova richiesta il controllore deve: 
• Scegliere dove dirottare il traffico mediante l’algoritmo statico: consistent hashing.  
Esso consiste in: 
o Calcolare l’hash con i parametri che identificano la connessione: 
▪ IP sorgente; 
▪ Porta sorgente. 
o Scegliere l’indirizzo fisico del server di destinazione mediante l’hash calcolato. 
• Installare la regola sullo switch che instrada il flusso verso il server scelto. 
Una volta installata la regola, essa sarà valida per tutti gli altri pacchetti del medesimo flusso, senza 
bisogno di ricalcolare l’hash. 

## Obiettivo secondario  
Instradamento deterministico dei flussi futuri. Una volta che un client viene assegnato ad un determinato 
server, tutte le sue richieste future arriveranno alla medesima destinazione. 
Il procedimento è simile a quanto già visto ma l’hashing verrà calcolato solo sull’IP sorgente. Non 
differenzieremo i diversi flussi del medesimo client, per ogni instradamento terremo in considerazione 
unicamente l’IP sorgente. 


## Architettura 
Usiamo un modello di controllo reattivo, con una sola regola installata nello switch durante la fase di 
CONFIG_DISPATCHER, subito dopo l’evento Features Reply. La regola ha priorità minima, ha match su ogni 
tipo di traffico e ha come istruzioni quelle di inviare ogni pacchetto al controllore. 
Una volta che il primo pacchetto di un flusso entra nello switch viene inoltrato al controllare, il quale 
controlla se il tipo di pacchetto è di nostro interesse (secondo le specifiche) e, in caso di esito positivo, si 
occupa dell’hashing dello stesso tramite la funzione hash. 
Tale hash ci servirà per calcolare il server verso il quale instradare il flusso, che verrà specificato 
modificando i campi di IP e MAC destinazione del pacchetto. 
A questo punto una regola verrà installata nello switch, che dopo un certo tempo (hard_timeout, 120s nel 
nostro caso) verrà eliminata. 
Ovviamente viene creata una corrispondente regola per il percorso server–host modificando i campi IP e 
MAC sorgente del pacchetto in arrivo dal server. 

I messaggi che ci interessano sono i tre principali: 
• Packet-In 
lo switch manda al controllore unicamente il primo pacchetto di un flusso di cui non conosce la 
destinazione. 
• Modify-State 
Dopo aver calcolato l’hash e aver fatto altre verifiche, il controllore aggiunge una riga nella tabella 
dello switch con la destinazione di tutti i pacchetti di quel flusso. Ovviamente la regola nella tabella 
è provvista di un timeout. 
• Packet-Out 
Il controllore dice allo switch dove mandare quel pacchetto. 
Il controllore si comporterà anche da proxy ARP, occupandosi di identificare e rispondere alle ARP request 
dei vari host e server, facendo sembrare “trasparente” il load balancer all’utente finale. 

## Risultati

![image](https://github.com/Mattiabe98/ryu-static-load-balancing/assets/49247389/a29d60c6-19af-49f0-9dbe-5ed265cfd8e3)
![image](https://github.com/Mattiabe98/ryu-static-load-balancing/assets/49247389/d5d9b834-a993-4136-b9d3-4e7f52fe5c0a)


## Developed By

* [Mattia Bevilacqua](https://github.com/Mattiabe98)
* Patatone
* lucio97

## License

Apache, see the [LICENSE](LICENSE) file.
