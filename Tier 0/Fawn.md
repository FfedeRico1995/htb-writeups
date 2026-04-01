Per prima cosa ci connettiamo a openvpn. poi controlliamo con ping e ip che sia connesso, dopodiche possiamo procedere alla scansione con nmap.
Il comando sara sudo nmap -sV {ip} per trovare le porte aperte, e la versione e notiamo che e aperta la porta 21/tcp ftp vsftpd 3.0.3
Sapere la versione ci puo essere utile perche possiamo capire se e una versione non aggiornata quindi vulnerabile.
Se non presente, installiamo ftp.
Ora tramite il comando ftp -help vediamo le possibilita che abbiamo, e da cio notiamo che con l'ip possiamo connetterci, quindi usiamo ftp {ip}
Ora ci chiede l'username, e spesso una missconfiguration di ftp permette di connetterci con anonymous, proviamo.
Funziona, alla richiesta di password lasciamo vuoto, e siamo dentro.
Con help possiamo vedere tutti i comandi possibili.
Dopo tot tempo senza comandi, si disconette, quindi usiamo open {ip} per riconnetterci e ricominciamo la nostra scansione.
Per rispondere alla domanda what os type is running, usiamo system.
Ora passiamo a vedere con ls cosa c'e e troviamo un flag.txt. Essendo via ftp, per scaricarlo e averlo nel nostro sistema, usiamo get flat.txt, una volta fatto cio possiamo uscire con bye, e ora lo troveremo nel nostro sistema.

