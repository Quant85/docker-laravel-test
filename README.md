# Snellire le immagini Docker

---

## Alpine Linux

[Alpine](https://alpinelinux.org/) è una distribuzione Linux focalizzata sulla sicurezza e con un ingombro ridotto,
presenta il minimo indispensabile per impostazione predefinita e
consente di installare ciò di cui si ha effettivamente bisogno per 
l' applicazione. La versione *dockerizzata* di Alpine è di appena 4 MB
e la maggior parte delle immagini Docker ufficiali fornisce una versione basata
su questa distribuzione.

Prima di modificare la nostra configurazione, sbarazziamoci di quella attuale:

        $ docker-compose down -v --rmi all --remove-orphans
Questa istruzione fermerà e / o distruggerà i contenitori, 
così come rimuoverà i volumi e le immagini, 
permettendoci di ricominciare da capo.

Modifichiamo la configurazione attuale del `docker-compose.yml` utilizzando le immagini dockerizzate di Alpine
`nginx: 1.19-alpine` e `phpmyadmin / phpmyadmin: 5-fpm-alpine`

Abbiamo montato un nuovo volume denominato `phpmyadmindata`(dichiarato in fondo al file) e utilizzato `depends_onper` 
indicare che il contenitore **phpMyAdmin** dovrebbe essere avviato per primo.

*Nginx* ora servirà *phpMyAdmin* così come la nostra applicazione *PHP*,
dove in precedenza l'immagine *phpMyAdmin* presentava il proprio server HTTP (Apache).
Come suggerisce il nome, il `5-fpm-alpine` tag è la versione basata su Alpine dell'immagine,
il cui contenitore esegue PHP-FPM come processo e si aspetta che i file PHP vengano gestiti da un server HTTP esterno.

Approfondimento al riguardo [GitHub issue](https://github.com/phpmyadmin/docker/issues/253)

Ora abbiamo bisogno di una nuova configurazione Nginx per ***phpMyAdmin***.

Creiamo un nuovo file `phpmyadmin.conf` in `.docker/nginx/conf.d`
 e configuriamo il server.

Una configurazione del server piuttosto standard che assomiglia molto a quella adottata in `php.conf`,
tranne per il fatto che punta alla porta `9000` del servizio `phpMyAdmin`.

Aggiornare il local `host`.

`127 .0.0.1 php.test phpmyadmin.test`

Il file `host` su distribuzioni Linux e macOS si trova in
`/etc/hosts`, per Windows dovrebbe esser in `c:\windows\system32\drivers\etc\hosts`.

Avendo aggiunto nel file `docker-compose.yml` nell'applicazione `phpmyadmin` il volume
  
    volumes:
    - phpmyadmindata: /var/www/html

Esso ci garantisce che i file phpMyAdmin siano disponibili
per Nginx. La differenza con l'applicazione PHP
è che invece di montare una cartella locale, lasciamo che Docker Composer scelga una cartella locale su cui montare entrambi i container
Nginx e phpMyAdmin, rendendo disponibile il contenuto di ques'ultimo al primo.

Inoltre abbiamo eliminato la mappatura della porta `8080`, 
perchè utilizzeremo direttamente la porta 80 di Nginx.

Poiche abbiamo modificato l'immagine PHP che stiamo utilizando, andiamo ad apportare una modifica anche
nel `Dockerfile` in `.docker/php`
    
    FROM php:8.0-fpm-alpine
    
    RUN docker-php-ext-install pdo_mysql

Unica differenza è l'aggiunta del suffisso alla fine della versione, `-alpine`.

Per quanto riguarda il servizio MySQL al momento non è disponibile una versione Alpine, quindi viene lasciato invariato.

Siamo ora pronti per testare la nostra nuova configurazione.
Esegui `docker-compose up -d`, 
seguito da `docker-compose images` per verificare il peso della nuova configurazione.

Se tutto è andato a buonfine dovremmo aver un peso complessivo di circa 700MB (dai 1.5 GB iniziali), di cui 470 MB
associati a mysql, che è rimasto invariato.

Ora per accedere a phpMyAdmin invece di accedere mediante `localhost:8080` possiamo accedere mediante [phpmyadmin.test](http://phpmyadmin.test/)
