# Docker: Architerrura a tre livelli 

---

## LEMP MySQL Database - Laravel Backend - Vue.js Frontend

La scelta di strutturare un'applicazione separandola su tre livelli, livello presentazione (frontend), livello dell'applicazione (backend) e livello persistenza ( database),
nasce dall'idea dhe ogni parte possa esser gestibile in modo indipendente, distribuibile, scalabile e facilmente sostituibile la dove necessario.

Prima di modificare la nostra configurazione, eliminiamo i container ed i volumi precendenti eseguendo l'istruzione:

        $ docker-compose down -v

`down` distrugge i container, mentre `-v` elimina i volumi associati.

Rispetto alla configurazione precedente useremo un servizio `backend`,
andando ad eliminando le configurazioni PHP, composta dalla directory `.docker/php` e dai i file `.docker/nginx/conf.dphp.conf` e `src/index.php`.

    docker-laravel-test/
    ├── .docker/
    │   ├── mysql/
    │   │   └── my.cnf
    │   └── nginx/
    │       └── conf.d/
    │           └── phpmyadmin.conf
    ├── src/
    ├── .env
    ├── .env.example
    ├── .gitignore
    └── docker-compose.yml


Modifichiamo la configurazione attuale del `docker-compose.yml`, 
rimuovendo il servizio `php` e montando nel servizio `nginx` un nuovo volume `- ./src/backend:/var/www/backend` 
e la dipenzenza al servizio che andremmo a creare, `- backend`

    # Nginx Service
    nginx:
      image: nginx:1.19-alpine
      ports:
        - 80:80
      volumes:
        - ./src/backend:/var/www/backend
        - ./.docker/nginx/conf.d:/etc/nginx/conf.d
        - phpmyadmindata:/var/www/phpmyadmin
      depends_on:
        - backend
        - phpmyadmin

    # Servizio Backend
    backend:
      build: ./src/backend
      working_dir: /var/www/backend
      volumes:
        - ./src/backend:/var/www/backend
      depends_on:
        mysql:
          condition: service_healthy

Il nuovo servizio `Backend` cerca un `Dokerfile` situato nella directory dell'applicazione `backend` ( `src/backend` ), montato come volume sul container.
L'applicazione di backend verrà costruita con **Laravel**, quindi creiamo una configurazione del server Nginx basata su quella fornita dalla
[documentazione ufficiale](https://laravel.com/docs/8.x/deployment#nginx) di Laravel.

In `.docker/nginx/conf.d` creiamo il file `backend.conf` e configuriamo il server.

    server {
    listen      80;
    listen      [::]:80;
    server_name backend.test;
    root        /var/www/backend/public;
    
        add_header X-Frame-Options "SAMEORIGIN";
        add_header X-XSS-Protection "1; mode=block";
        add_header X-Content-Type-Options "nosniff";
    
        index index.html index.htm index.php;
    
        charset utf-8;
    
        location / {
            try_files $uri $uri/ /index.php?$query_string;
        }
    
        location = /favicon.ico { access_log off; log_not_found off; }
        location = /robots.txt  { access_log off; log_not_found off; }
    
        error_page 404 /index.php;
    
        location ~ \.php$ {
            fastcgi_pass  backend:9000;
            fastcgi_index index.php;
            fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
            include       fastcgi_params;
        }
    
        location ~ /\.(?!well-known).* {
            deny all;
        }
    }

Aggiornare il local `host`.

`127 .0.0.1 backend.test frontend.test phpmyadmin.test`

Il file `host` su distribuzioni Linux e macOS si trova in
`/etc/hosts`, per Windows dovrebbe esser in `c:\windows\system32\drivers\etc\hosts`.

Se non è stata già creata, va aggiunta la directory `src/backend` ed al suo interno aggiungere il file `Dockerfile`, con all'interno l'istruzione

    FROM php:8.0-fpm-alpine

Struttura attuale:

    docker-laravel-test/
    ├── .docker/
    │   ├── mysql/
    │   │   └── my.cnf
    │   └── nginx/
    │       └── conf.d/
    │           └── backend.conf
    │           └── phpmyadmin.conf
    ├── src/
    │   └── backend/
    │       └── Dockerfile
    ├── .env
    ├── .env.example
    ├── .gitignore
    └── docker-compose.yml

Al fine di aver un corretto funzionamento, **Laravel** richiede alcune estensioni PHP, quindi dobbiamo assicurarci che esse siano installate.
La versione dell'immagine ***PHP-Alpine*** è fornita con una serie di estensioni preinstallate che possono esser elencate eseguendo l'istruzione (dalla root del progetto).

`$ docker-compose run --rm backend php -m`

La differenza tra il comando `exec` ed il comando `run` è che mentre il primo ci permette semplicemente di eseguire un comando su un container **gia in esecuzione**,
il secondo, `run`, lo esegue su un nuovo container che viene immediatamente fermato al termine del comando. Tuttavia non elimina il container per impostazione predefinita;
per ottenere tale risultato dibbiamo specificare l'istruzione `--rm` dopo `run`.

Il comando che vien e eseguito è `php -m` sul container `backend`, e formisce come risultato:


            [PHP Modules]
            Core
            ctype
            curl
            date
            dom
            fileinfo
            filter
            ftp
            hash
            iconv
            json
            libxml
            mbstring
            mysqlnd
            openssl
            pcre
            PDO
            pdo_sqlite
            Phar
            posix
            readline
            Reflection
            session
            SimpleXML
            sodium
            SPL
            sqlite3
            standard
            tokenizer
            xml
            xmlreader
            xmlwriter
            zlib
            
            [Zend Modules]

I manutentori dell'immagine PHP hanno scelto di semplificare la vita dei propri utenti preinstallando un bel pacchetto di estensioni.
Tuttavia ciò non incide particolarmente sul peso complessivo, in quanto siano sull'ordine degli 80MB complessivi;

Osservando le estensioni presenti possiamo installare quelle mancanti, andando ad implementarle nel `Dokerfile`, 
insieme a [Composer](https://hub.docker.com/_/composer), necessario per **Laravel**;


            FROM php:8.0-fpm-alpine
            
            # Install extensions
            RUN docker-php-ext-install pdo_mysql bcmath
            
            # Install Composer
            COPY --from=composer:latest /usr/bin/composer /usr/local/bin/composer

In questo modo si sfrutta un'immagine esterna per ottenere l'ultima versione di Composer (buil multi-stage).

Andiamo ad aggiornare il Dockerfile, e quindi a ricostruire l'immagine:

`docker-compose build backend`

Successivamente eseguiamo l'istruzione
`docker-compose run --rm backend composer create-project --prefer-dist laravel/laravel tmp "8.*"`

Questo comando utilizzerà la versione di Composer installata sul containe **backend** (non sarà necessario installare Composer localmente)
 per creare un nuovo progetto Larave 8 nella cartella del container `var/www/backend/tmp` (nome progetto ***tmp***).

Come per `docker-compose.yml`, la directory di lavoro del container è `/var/www/backend` su cui è stata montata la cartella locale `src/backend`, ora in quest'ultima c'è il 
nuovo progetto Laravel nella cartella `tmp` che sarà l'allocazione temporanea del progetto.

Uno dei motivi per il quale viene creata una nuova cartella è perchè dietro le quinte il comando `composer create-project` esegue un `git clone` che a meno che la directory di
destinazione non sia vuota, non andrà a buon fine. 
Di fatto se avessimo provato a creare il progetto direttamente nella cartella **backend**, poichè già presente il file `Dockerfile`, no sarebbe andato a buon fine.

Quindi una volta creato il progetto nella cartella temporanea `tmp`, lo riporteremo nella directory di destinazione `backend`, mediante l'istruzione:

`docker-compose run --rm backend sh -c "mv -n tmp/.* ./ && mv tmp/* ./ && rm -Rf tmp"`

Questo esegue l'istruzione tra virgolette sul container, e per poter eseguire più di un singolo comando contemporaneamente sul containre sfruttiamo l'istruzione `sh -c`,
(in questo modo si evita che venga eseguita solo la prima istruzione `mv` sul container e le restanti sulla macchina locale).

Possiamo ora configurare il file `.env` generato da Laravel

    APP_NAME=demo
    APP_ENV=local
    APP_KEY=base64:BcvoJ6dNU/I32Hg8M8IUc4M5UhGiqPKoZQFR804cEq8=
    APP_DEBUG=true
    APP_URL=http://backend.test
    
    LOG_CHANNEL=single
    
    DB_CONNECTION=mysql
    DB_HOST=mysql
    DB_PORT=3306
    DB_DATABASE=demo
    DB_USERNAME=root
    DB_PASSWORD=root
    
    BROADCAST_DRIVER=log
    CACHE_DRIVER=file
    QUEUE_CONNECTION=sync
    SESSION_DRIVER=file

Se si dovessero verificare problemi in merito ai permessi di scrittura, una possibile soluzione è entrare nella directori `src` e modificare i permessi della cartella `backend` con:

`sudo chown -R $USER:$USER backend `

Proviamo la nuova configurazione, eseguiamo

`docker-compose up -d`

e visitiamo [backend.test](http://backend.test/);
Se tutto è andato a buonfine avremo accesso alla views `welcome.blade.php`. 

Ora possiamo verificare che vengano correttamente eseguite le migrazioni predefinite di Laravel al nostro database eseguendo l'istruzione:

`docker-compose exec backend php artisan migrate`

Possiamo accedere a [phpmyadmin.test](http://phpmyadmin.test/), (con le credenziali `root / root`) e confermare la presenza del database `demo` e le relative tabelle di nuova creazione
(`failed_job`,`migrations`,`password_resets` e `users`).

Le istruzioni da utilizzare quando il container backend è in esecuzione per eseguire i comandi **Artisan** e **Composer** hanno la sintassi:

    docker-compose exec backend php artisan
    docker-compose exec backend compositore

Mentre se non è in esecuzione:

    docker-compose run --rm backend php artisan
    docker-compose run --rm backend composer


##OPcache

>OPcache migliora le prestazioni di PHP memorizzando 
il bytecode di script precompilati nella memoria condivisa, 
eliminando così la necessità per PHP di caricare e analizzare 
gli script su ogni richiesta.
---
Per poter abilitare **OPcache**, creiamo nella directory `src/backend` una nuova cartella `.docker`,
 e creamo il file `php.ini`, dove andremo a definire la configurazione di opcache

    [opcache]
    opcache.enable=1
    opcache.revalidate_freq=0
    opcache.validate_timestamps=1
    opcache.max_accelerated_files=10000
    opcache.memory_consumption=192
    opcache.max_wasted_percentage=10
    opcache.interned_strings_buffer=16
    opcache.fast_shutdown=1

Posizioniamo questo file qui e non alla radice del progetto perché 
questa configurazione è specifica dell'applicazione di backend e dobbiamo farvi riferimento 
dal suo Dockerfile

Quindi andremo ad aggiornare il `Dockerfile` in `src/backend`

    FROM php:8.0-fpm-alpine
    
    # Installare estensioni
    RUN docker-php-ext-install pdo_mysql bcmath opcache
    
    # Installa Composer
    COPY --from=composer:latest /usr/bin/composer /usr/local/bin/composer
    
    # Configura PHP
    COPY .docker/php.ini $PHP_INI_DIR/conf.d/opcache.ini
    
    # Usa la configurazione di sviluppo predefinita
    RUN mv $PHP_INI_DIR/php.ini-development $PHP_INI_DIR/php.ini

In questo modo abbiamo copiato il `php.ini` nella directory dove ci si aspetta
che le configurazioni personalizzate vadano inserite nbel container, e la cui posizione
è data dalla `$PHP_INI_DIR` che è una variabile d'ambiente. Inoltre abbiamo utilizzato 
le impostazioni di sviluppo predefinite, fornite dai manutentori dell'immagine, che impostano
i parametri per la segnalazione degli errori.

Ricostruiamo di nuovo l'immagine e riavviamo i container

    docker-compose build backend
    docker-compose up -d
Dovremmo notare dei miglioramenti nella reattività del backend.