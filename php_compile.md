title: Οδηγίες για compile της PHP από τον πηγαίο κώδικα 
author: Theodoros Deligiannidis
date: 18 Mar 2020

Ο δεύτερος οδηγός μου είναι η μεταγλώττιση της PHP από τον πηγαίο κώδικα που στην ουσία είναι 
συμπληρωματικός στον πρώτο οδηγό μιας και php(και συγκεκριμένα php-fpm) και 
nginx συνήθως πάνε πακέτο.

Όταν τελειώσετε με τα παρακάτω βήματα θα έχετε στη διάθεση σας την τελευταία έκδοση 
της PHP και του PHP-FPM.

## Βήμα 1ο: Πηγαίος κώδικας και ρυθμίσεις περιβάλλοντος ##
Κάνουμε ένα φάκελο μέσα στο home όπου θα είναι εγκαταστημένα τα εκτελέσιμα αρχεία και τα 
αρχεία ρυθμίσεων.

    mkdir ~/php

Κατεβάζουμε την τελευταία έκδοση PHP 7.4
    
    wget https://www.php.net/distributions/php-7.4.3.tar.gz
    tar xzvf php-7.4.3.tar.gz
    cd php-7.4.3

θα χρειαστείτε μερικά αναγκαία προγράμματα(και μερικά προαιρετικά όπως webp):

    sudo apt-get install -y libargon2-0-dev libzip-dev libzip4 libsqlite3-dev libonig-dev webp

## Βήμα 2ο: Μεταγλώττιση ##
Για τη δικιά σας ευκολία αλλά και σε περίπτωση που θέλετε να πειραματιστείτε πολλές φορές, μπορείτε 
το configure να το ενσωματώσετε σε ένα σκριπτάκι bash που μπορείτε να το ονομάσετε όπως θέλετε:

    #!/bin/sh
    INSTALL_DIR=$HOME/bin/php7
    mkdir -p $INSTALL_DIR

    ./configure 
        --prefix=$INSTALL_DIR \
        --enable-bcmath \
        --enable-fpm \
        --with-fpm-user=www-data \
        --with-fpm-group=www-data \
        --disable-cgi \
        --enable-mbstring \
        --enable-shmop \
        --enable-sockets \
        --enable-sysvmsg \
        --enable-sysvsem \
        --enable-sysvshm \
        --with-zlib \
        --with-curl \
        --without-pear \
        --with-openssl \
        --enable-pcntl \
        --with-password-argon2 \
        --with-sodium \
        --enable-mysqlnd \
        --with-pdo-mysql \
        --with-pdo-mysql=mysqlnd \
        --enable-gd \
        --with-jpeg  \
        --enable-opcache \
        --enable-zip \
        --enable-exif

Στο παραπνω σετ ρυθμίσεων, εκτός των άλλων, ενεργοποιούμε το opcache το οποίο είναι must για την PHP
και φυσικά την υποστηριξη mysql(``--enable-mysqlnd``), επεξεργασία εικόνων(``--enable-gd``) και άλλα.

Όταν είμαστε έτοιμοι, διαλέγουμε τις παραμέτρους για τον gcc:

    export CFLAGS='-Ofast -pipe -fomit-frame-pointer -march=native -fPIE -fstack-protector-strong -Wformat -Werror=format-security -Wdate-time -D_FORTIFY_SOURCE=2'  
    export CXXFLAGS="${CFLAGS}"
    export LDFLAGS='-Wl,-Bsymbolic-functions -fPIE -pie -Wl,-z,relro -Wl,-z,now'

και τρέχουμε το σκριπτάκι:

    sh buildPHP.sh

Αφού τελειώσει χωρίς σφάλματα προχωράμε στην μεταγλώττιση:

    make -j 4
Η παράμετρος ``-j 4`` είναι προαιρετική και έχει να κάνει με τους διαθέσιμους πυρήνες του CPU σας επιταχύνοντας έτσι την μεταγλώττιση. Μπορείτε να το βρείτε δίνοντας:
    
    lscpu  |grep 'CPU(s)'
    CPU(s):              4

Όταν το make τελειώσει χωρίς σφάλματα, κάνουμε εγκατάσταση δίνοντας:

    sudo make install

## Βήμα 3ο - Παραμετροποίηση της PHP
Αρχικά κάνουμε αντιγραφή του php.ini που θα χρησιμοποιήσουμε:

    cp php.ini-development ~/php/php7/lib/php.ini
    nano ~/php/php7/lib/php.ini

Θα χρειαστεί να ρυθμίσουμε μερικές παραμέτρους της PHP.

Ζώνη ώρας

    [Date]
    date.timezone=Europe/Athens

Ο οδηγός της Mysql(PDO) αν αυτή τρέχει τοπικά

    pdo_mysql.default_socket=/var/run/mysqld/mysqld.sock

Απενεργοποίηση του pathinfo για λόγους ασφαλείας:

    cgi.fix_pathinfo=0:

Αφού τελειώσουμε με τις ρυθμίσεις της PHP, προχωράμε στις ρυθμίσεις του PHP-FPM. Θα χρειαστούμε ένα πρότυπο για να 
δουλέψουμε, το οποίο βρίσκεται στο φακελο ``~/php/php7/etc``:

    cd ~/php/php7/etc/; mv php-fpm.conf.default php-fpm.conf
    cd ~/php/php7/etc/php-fpm.d/; mv www.conf.default www.conf

Ανοίγουμε το αρχείο ``www.conf`` και ελέγχουμε αν ο user και το group είναι το σωστό(``www-data``).
    
    user = www-data
    group = www-data
    listen.owner = www-data
    listen.group = www-data
    listen.mode = 0660

Σε περίπτωση που θέλουμε να χρησιμοποιήσουμε sockets αντί για TCP(είναι πιο γρήγορα αλλά με μερικούς περιορισμούς) τότε αλλάζουμε
την επιλογή ``listen``

    listen = /var/run/php-fpm.sock

Επίσης, για λόγους ταχύτητας, καλό είναι να ενεργοποιήσουμε την επέκταση opcache που είναι διαθέσιμη γιατί κατά το configure δώσαμε 
``--enable-opcache``. Ανοίγουμε το αρχείο ``~/php/php7/lib/php.ini`` και προσθέτουμε την γραμμή 

    zend_extension=~/php/php7/lib/php/extensions/no-debug-non-zts-20190902/opcache.so

Τέλος ξεκινάμε τo php-fpm με την εντολή:

    sudo $HOME/php/php7/sbin/php-fpm

## Βήμα 4ο - Σύνδεση με τον NGINX ##
Αν και δεν είναι στο πλαίσιο αυτού του οδηγού το πως θα ρυθμίσετε τη PHP με τον NGINX, παρακάτω θα βρείτε ένα παράδειγμα 
για το πως ρυθμίζεται η PHP μέσα στο server block του NGINX:

    location ~ \.php$ {
        include fastcgi_params;
        fastcgi_intercept_errors on;
        fastcgi_pass unix:/var/run/php-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root/$fastcgi_script_name;
    }

Βλέπουμε οτι στη παράμετρο fastcgi_pass δώσαμε socket αντί για TCP μιας και στο 3ο βήμα το αλλάξαμε από ``listen = 127.0.0.1:9000
`` που ήταν το default.


Happy compiling!

Ο οδηγός βασίστηκε πάνω σε αυτό το άρθρο: https://github.com/gitKearney/php7-from-scratch

Θεόδωρος Δεληγιαννίδης
thiodor@gmail.com
