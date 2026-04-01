Dopo essermi connesso, inizio la scansione con nmap per trovare le porte aparte, per fare cio usero sudo nmap -sV e l'ip, sV serve appunto a trovare le porte aperte per determinare service e version info.
Il risultato porta a trovare la porta 23/tcp open telnet Linux telnetd, che sappiamo telnet non essere sicura, non criptando i dati.
Con il comando sudo telnet e l'ip, si connette alla rete e chiede le credenziali.

Ora dobbiamo trovare le credenziali, quando non abbiamo ancora le capacita di trovare questi dati, si puo provare un brute force diciamo manuale, ovvero provando le credenziali base, spesso per altro, per queste connessioni, oltre a username base come root, admin, o nomi propri, viene lasciato senza password, quindi ora proviamo vari username e a lasciare senza password.

Provando con root, siamo riusciti ad accedere senza che chiedesse alcuna password.
Ora siamo connessi, e con un semplice ll possiamo vedere i file e troviamo flag.txt, che con un semplice cat flag.txt possiamo trovare la risposta alla domandaa.

Prima box fatta, molto facile, ma grande soddisfazione.