# Modulo Hooking - Dettagli Tecnici

1.  [Operazioni relative ad un arco-identità](#Operazioni_arco_identita)

## <a name="Operazioni_arco_identita"></a>Operazioni relative ad un arco-identità

Il modulo Hooking inizia le sue attività relativamente ad una certa identità nel sistema.
Cioè una istanza di `HookingManager` viene costruita quando si crea una identità nel
sistema e a questa viene associata.  
L'utilizzatore del modulo Hooking comunica poi a questa istanza di `HookingManager` la
nascita e la rimozione di ogni arco-identità associato a quella identità nel sistema. Questo
chiamando i metodi pubblici `add_arc` e `remove_arc` dell'istanza di `HookingManager`.

Il modulo Hooking, quando aggiunge un arco-identità, avvia una nuova tasklet. In essa eseguirà
tutte le operazioni relative a quell'arco-identità. Quando rimuove un arco-identità abortisce
la tasklet relativa.

### Esame delle identità

La tasklet per prima cosa esamina il tipo di identità su cui è in esecuzione. Se si tratta di una
identità *di connettività* la tasklet termina. Se si tratta di una identità *principale* (quindi
con indirizzo *reale*) allora prosegue. Questa verifica viene fatta all'inizio
e in seguito ogni volta che la tasklet intende chiamare il metodo remoto `retrieve_network_data`.

La tasklet chiama il metodo remoto `retrieve_network_data` sul suo arco-identità. La risposta può essere
l'eccezione `NotPrincipalError` oppure informazioni sulla rete
di appartenenza del vicino.

Se rileva che l'identità vicina è *di connettività*, allora la tasklet termina.

Se l'identità vicina appartiene alla nostra stessa rete, allora la tasklet emette un segnale `same_network` indicando
questo arco-identità. L'utilizzatore gestirà questo segnale memorizzando che quel dato arco-identità congiunge
due nodi della stessa rete e quindi aggiungendo un IQspnArc al modulo QSPN.  
Poi la tasklet termina. Infatti l'identità vicina potrebbe in seguito diventare di un'altra rete, ad
esempio a causa di uno split di g-nodo. Ma in questo caso l'utilizzatore del modulo procederebbe
a rimuovere l'arco-identità e eventualmente a ricostruirne un altro.

Se l'identità vicina appartiene ad una rete diversa che ha una topologia diversa, allora la tasklet termina.  
Le due reti, infatti, non potranno in alcun modo fondersi.

Se l'identità vicina appartiene ad una rete diversa che ha la stessa topologia, allora la tasklet procede
con la valutazione.

### Valutazione dell'ingresso

Se la valutazione suggerisce di non fare ingresso nell'altra rete, allora attende un tempo lungo (10 minuti) prima
di ripetere le operazioni di esame dell'identità.

Se invece la valutazione suggerisce di fare ingresso nell'altra rete, allora la tasklet interroga il nodo
Coordinator della sua rete chiamando ripetutamente il metodo `evaluate_enter` del modulo Coordinator
con le modalità descritte nell'analisi, cioè finché riceve l'eccezione `AskAgainError`.

Se infine riceve l'eccezione `IgnoreNetworkError` allora attende un tempo molto lungo (20 volte il
`global_timeout` calcolato sulla base della dimensione della rete) prima di ripetere di nuovo le operazioni
di esame dell'identità.

Se invece alla fine riceve il valore `int first_ask_lvl`, allora la tasklet ha il compito di tentare
l'ingresso del suo g-nodo di quel livello tramite il suo arco-identità.

La tasklet per prima cosa pone `ask_lvl = first_ask_lvl`. Poi inizia le operazioni di esecuzione
dell'ingresso al livello `ask_lvl`.

### Esecuzione dell'ingresso al livello ask_lvl

La tasklet interroga il nodo Coordinator del suo g-nodo di livello `ask_lvl` chiamando il metodo
`begin_enter` del modulo Coordinator con le modalità descritte nell'analisi.

Se riceve l'eccezione `AlreadyEnteringError` allora attende un tempo molto lungo (20 volte il
`global_timeout` calcolato sulla base della dimensione della rete) prima di ripetere di nuovo le operazioni
di esame dell'identità.

Altrimenti la tasklet chiama sul suo arco-identità il metodo remoto `search_migration_path(ask_lvl)`.

Se riceve l'eccezione `MigrationPathExecuteFailureError` allora subito riprova con la stessa chiamata
del metodo remoto `search_migration_path` con lo stesso livello.

Se riceve l'eccezione `NoMigrationPathFoundError` allora dovrà fare più operazioni. Per prima cosa
comunica al nodo Coordinator del suo g-nodo di livello `ask_lvl` il fatto che questo tentativo è abortito,
chiamando il metodo `abort_enter` del modulo Coordinator con le modalità descritte nell'analisi.  
Poi decrementa di 1 il valore di `ask_lvl`. Se adesso è minore di 0, allora non è proprio possibile
entrare nell'altra rete. Quindi la tasklet attende un tempo molto lungo (20 volte il
`global_timeout` calcolato sulla base della dimensione della rete) prima di ripetere di nuovo le operazioni
di esame dell'identità.  
Se invece il nuovo valore di `ask_lvl` non è minore di 0, allora ripete dall'inizio le operazioni di
esecuzione dell'ingresso al livello `ask_lvl`, ripartendo dalla chiamata di `begin_enter`.

Se invece il metodo remoto `search_migration_path(ask_lvl)` non rilancia eccezioni, allora la tasklet
riceve una istanza di `EntryData entry_data` con le informazioni necessarie all'ingresso del suo g-nodo di
livello `ask_lvl` nell'altra rete.

Ora la tasklet inventa un identificativo `migration_id`. Poi fa uso della collaborazione con il modulo
Coordinator per avviare una *propagazione con ritorno* a tutti i singoli nodi del suo g-nodo di
livello `ask_lvl`. Essi avviano la prima parte delle operazioni di duplicazione dell'identità usando
l'identificativo `migration_id` comunicato dalla nostra tasklet.

La tasklet di nuovo fa uso della collaborazione con il modulo Coordinator per avviare una *propagazione senza ritorno*
a tutti i singoli nodi del suo g-nodo di livello `ask_lvl`. Essi avviano la seconda parte delle operazioni
di duplicazione dell'identità e la costruzione di un g-nodo isomorfo dentro la nuova rete.  
Ricordiamo che in questa operazione non diamo nessuna importanza al mantenimento della connettività
interna dei g-nodi della vecchia rete: cioè le precedenti identità vengono dismesse immediatamente senza
dover assumere una posizione *di connettività*.

Le informazioni che vanno comunicate ai singoli nodi in questa propagazione sono l'identificativo `migration_id`
e quelle contenute in `entry_data`.

Dopo aver dato il via alla *propagazione senza ritorno* la tasklet fa in modo che nella sua stessa
identità venga operata questa migrazione. Poi, essendo la sua identità destinata a venire dismessa, termina.
