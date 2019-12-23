# Resource Public Key Infrastructure (RPKI)

## sommario

- `cos'è`_;

## Cos’è

Molti degli incidenti nell’Internet sono causati dalla propagazione di incorrette informazioni di routing. 
Le più comuni minacce, come ad esempio route hijacking o route leak, sfruttano una vulnerabilità di fondo del 
protocollo BGP: l’impossibilità di verificare se gli Autonomous System (AS) che propagano gli annunci sono realmente 
legittimati nel farlo. Infatti, le informazioni sulle proprietà delle risorse Internet (ASN e prefissi) sono contenute 
all’interno di database pubblici chiamati Internet Routing Registry (IRR), gestiti e mantenuti dai Regional 
Internet Registry (RIR). Non risiedono quindi all’interno del protocollo BGP.

Il BGP non ha alcuno strumento per verificare se un AS che annuncia un determinato prefisso nell’Internet sia autorizzato a farlo, ossia se il prefisso annunciato gli è stato assegnato effettivamente da un RIR. Dato che ogni prefisso può essere annunciato e originato da ogni AS, indipendentemente dal suo diritto nel farlo, è necessario un meccanismo out-of-band per aiutare il BGP a verificare quale AS può annunciare quale prefisso.

Questo meccanismo esiste. E’ parte del sistema IRR. Come accennato in precedenza esistono dei database pubblici che contengono le informazioni sulla proprietà dei prefissi, alcuni gestiti dai grandi operatori, altri gestiti dai RIR stessi. E’ ormai ampiamente diffuso generare i filtri a partire dalle informazioni presenti negli IRR. Esiste però un limite a questo sistema: possiamo fidarci di queste informazioni? Purtroppo il sistema IRR è ben lontano dall’essere completo, gli oggetti che rappresentano le informazioni <Prefissi; AS autorizzati> non sono del tutto affidabili, questo perchè spesso non vengono aggiornati o contengono informazioni errate.

Per ovviare a questo problema, il gruppo SIDR (Secure Inter Domain Routing) di IETF ha sviluppato nel 2012 un framework (RFC 6481) basato su una struttura pubblica (Resource Public Key Infrastructure) con dei database distribuiti (RPKI repository) dove sono contenute le associazioni <Prefissi; AS autorizzati>. A ciascuna associazione è legato un Certificato Digitale, che consente a chi consulta i database, di verificare che le associazioni siano corrette. Tali attestati (prefissi, ASN e certificati digitali) vengono denominati ROA (Route Origin Authorization).

RPKI utilizza il formato dei certificati digitali X.509 con l’estensione per indirizzi IP e ASN (RFC 3779). I certificati non includono le informazioni di identità ma il loro scopo è solo quello di trasferire il diritto all’uso delle risorse Internet. I RIR svolgono la funzione di Certification Authority (CA) e si occupano dell’emissione dei certificati digitali. L’utilizzo principale dei certificati digitali è quello di validare le chiavi pubbliche e la legittimità di un AS a iniettare nel BGP un determinato blocco di prefissi e di utilizzare un determinato numero di AS.
