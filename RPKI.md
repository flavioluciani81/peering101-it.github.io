# Resource Public Key Infrastructure (RPKI)

## Sommario

- [Cos'è](RPKI.md#cosè)
- [Architettura RPKI](RPKI.md#architettura-rpki)
- RPKI e prefix Hijacking (esempio con e senza RPKI)
- [Route Origin Authorization (ROA)](RPKI.md#route-origin-authorization-roa)
- [Creazione dei ROA](RPKI.md#creazione-dei-roa)
- ROA con AS Origine = 0
- Route Origin Validation (ROV)
- Aspetti di configurazione
- Verifica della configurazione
- RPKI e processo di selezione BGP
- Software di validazione

## Cos’è

Molti degli incidenti nell’Internet sono causati dalla propagazione di incorrette informazioni di routing. 
Le più comuni minacce, come ad esempio route hijacking o route leak, sfruttano una vulnerabilità di fondo del 
protocollo BGP: l’impossibilità di verificare se gli *Autonomous System (AS)* che propagano gli annunci sono realmente 
legittimati nel farlo. Infatti, le informazioni sulle proprietà delle risorse Internet (ASN e prefissi) sono contenute 
all’interno di database pubblici chiamati *Internet Routing Registry (IRR)*, gestiti e mantenuti dai *Regional 
Internet Registry (RIR)*. Non risiedono quindi all’interno del protocollo BGP.

Il BGP non ha alcuno strumento per verificare se un AS che annuncia un determinato prefisso nell’Internet sia autorizzato a farlo, ossia se il prefisso annunciato gli è stato assegnato effettivamente da un RIR. Dato che ogni prefisso può essere annunciato e originato da ogni AS, indipendentemente dal suo diritto nel farlo, è necessario un meccanismo out-of-band per aiutare il BGP a verificare *quale AS può annunciare quale prefisso*.

Questo meccanismo esiste. E’ parte del sistema IRR. Come accennato in precedenza esistono dei database pubblici che contengono le informazioni sulla proprietà dei prefissi, alcuni gestiti dai grandi operatori, altri gestiti dai RIR stessi. E’ ormai ampiamente diffuso generare i filtri a partire dalle informazioni presenti negli IRR. Esiste però un limite a questo sistema: possiamo fidarci di queste informazioni? Purtroppo il sistema IRR è ben lontano dall’essere completo, gli oggetti che rappresentano le informazioni <Prefissi; AS autorizzati> non sono del tutto affidabili, questo perchè spesso non vengono aggiornati o contengono informazioni errate.

Per ovviare a questo problema, il gruppo SIDR (Secure Inter Domain Routing) di IETF ha sviluppato nel 2012 un framework (RFC 6481) basato su una struttura pubblica (Resource Public Key Infrastructure) con dei database distribuiti (RPKI repository) dove sono contenute le associazioni <Prefissi; AS autorizzati>. A ciascuna associazione è legato un Certificato Digitale, che consente a chi consulta i database, di verificare che le associazioni siano corrette. Tali attestati (prefissi, ASN e certificati digitali) vengono denominati ROA (Route Origin Authorization).

RPKI utilizza il formato dei certificati digitali X.509 con l’estensione per indirizzi IP e ASN (RFC 3779). I certificati non includono le informazioni di identità ma il loro scopo è solo quello di trasferire il diritto all’uso delle risorse Internet. I RIR svolgono la funzione di Certification Authority (CA) e si occupano dell’emissione dei certificati digitali. L’utilizzo principale dei certificati digitali è quello di validare le chiavi pubbliche e la legittimità di un AS a iniettare nel BGP un determinato blocco di prefissi e di utilizzare un determinato numero di AS.

## Architettura RPKI

![architettura rpki](architettura-rpki.png "architettura rpki")

L’architettura è basata su dei database *(RPKI repository)*, dove sono contenute le informazioni sui ROA, informazioni che possono essere immesse direttamente dai RIR o anche dai NIR/LIR/ISP (verificate comunque dai RIR) attraverso un particolare *Publication Protocol*. 

Ciascun RPKI repository, oltre ai ROA, mantiene anche una lista di *sub-RPKI repository (anche detti Trust Anchors)* che vengono utilizzati per sincronizzare le informazioni sui ROA e quindi realizzare un unico RPKI repository virtuale a livello mondiale.

Sulla base dei ROA contenuti nei RPKI repository, un AS può validare la correttezza degli annunci ricevuti e scegliere in modo selettivo gli annunci che vuole accettare o meno. 


Affinché tutto ciò sia efficace, è necessaria per un AS la certezza che le informazioni dei RPKI repository siano corrette e sincronizzate a livello mondiale. L’architettura RPKI prevede quindi una gerarchia di RPKI repository e quindi di Certificali Digitali, che segue la stessa gerarchia prevista per l’assegnazione degli indirizzi IP, che può essere semplificata come segue: *IANA→RIR→NIR/LIR*. I 5 RIR mondiali hanno il compito di gestire ciascuno il RPKI repository dei ROA della propria area, e di mettere queste informazioni a disposizione degli altri RIR e degli ISP di tutto il mondo.  

Lato ISP, l’architettura prevede l’utilizzo di server denominati *RPKI Validator*, che a loro volta si interfacciano con i RPKI repository dei vari RIR per effettuare un download locale dei ROA. I router di Edge dell’ISP, attraverso il protocollo standard *RPKI-to-Router Protocol* (RFC 6810 - The Resource Public Key Infrastructure to Router Protocol, Gennaio 2013), scaricano localmente i ROA presenti nel RPKI Validator, inserendoli in una *RPKI Table*. All’arrivo di un annuncio BGP, un router può così inferire sulla validità o meno degli annunci che riceve, confrontando il contenuto degli annunci con quello dei ROA presenti nella propria RPKI Table.

## Route Origin Authorization (ROA)

I ROA sono oggetti firmati digitalmente. Accoppiano un AS a uno o più prefissi IP. Possono essere visti come degli oggetti che forniscono un mezzo per verificare se un AS è autorizzato ad annunciare un determinato prefisso IP.

Formato di un ROA IPv4:

| Prefisso originato | 172.16.0.0/16 |
| ----------- | ------ |
| Lunghezza massima della maschera | 24 |
| AS Origine | 12345 |
| Firma Digitale | qç!r5@eX!%89?@cv! |

Formato di un ROA IPv6:

| Prefisso originato | 2001:db8:1::/48 |
| ----------- | ------ |
| Lunghezza massima della maschera | 48 |
| AS Origine | 12345 |
| Firma Digitale | qç!r5@eX!%89?@cv! |

Un ROA ha 4 componenti:

1) __Prefisso originato__: Un prefisso IPv4 o IPv6, con una determinata lunghezza della maschera. Tipicamente è un prefisso assegnato da un RIR a un NIR/LIR/ISP

2) __Lunghezza massima della maschera__: specifica quali subnet IP del prefisso originato possono essere annunciate

3) __AS Origine__: il numero dell’AS che origina il prefisso IP o una sua subnet ammessa, ossia con lunghezza della maschera inferiore o al più uguale a quella specificata nel punto precedente

4) __Firma Digitale__: basata sul sistema chiave pubblica/chiave privata

## Creazione dei ROA

I RIR forniscono una soluzione hosted nascondendo tutti i dettagli relativi alla complessità crittografica. Un operatore deve focalizzare la propria attenzione solo sulla creazione e pubblicazione dei ROA per i prefissi annunciati.

La prima cosa da fare è accedere al portale del RIR di riferimento (in questa guida faremo riferimento al portale di RIPE NCC, https://www.ripe.net). Una volta autenticati, cliccare sul tab "My Resources".

![Ripe NCC Portal my resources](ripe_my_resources.png "My Resoures")

Nella schermata successiva, cliccare nel menu a sinistra sulla voce "RPKI dashboard" e quindi sul tab "Route Origin Authorisations (ROAs)"

![RPKI dashboard](rpki_dashboard.png "RPKI dashboard")

Da qui è possibile visualizzare l'elenco dei ROA creati, aggiungerne di nuovi oppure cancellarli.
