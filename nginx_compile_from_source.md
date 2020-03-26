title: Οδηγίες για compile του Nginx από τον πηγαίο κώδικα 
author: Theodoros Deligiannidis
date: 15 Mar 2020

Ο παρακάτω οδηγός θα σας βοηθήσει να μεταγλωττίσετε(compile) τον διακομιστή HTTP
NGINX. Ο οδηγός γράφτηκε για Ubuntu server 18.04.4 αλλά προσαρμόζεται εύκολα και σε άλλες εκδόσεις.

## Για ποιο λόγο να εγκαταστήσω τον NGINX με αυτό τον τρόπο και όχι μέσω της διανομής μου; ##
Οι λόγοι είναι αρκετοί και διαφέρουν από χρήστη σε χρήστη. Ένας λόγος 
είναι ότι θα έχετε πάντα την τελευταία έκδοση η οποία περιέχει και τις πιο πρόσφατες ενημερώσεις
ασφαλείας.

Επίσης, πολλές φορές τα έτοιμα πακέτα των διανομών, υστερούν σε ορισμένα χαρακτηριστικά τα οποία για μερικούς είναι αναγκαία, πχ υποστήριξη για streaming αρχεία MP4 (ngx_http_mp4_module) ή όπως θα δείτε πιο κάτω υποστήριξη brotli compression. Ένα άλλο πλεονέκτημα είναι μια μικρή αύξηση στις επιδόσεις του nginx αλλά και το μικρότερο memory footprint.

## Βήμα 1ο: Πηγαίος κώδικας και ρυθμίσεις περιβάλλοντος ##
Κατεβάστε τον πηγαίο κώδικα. Ο nginx είναι διαθέσιμος σε 2 εκδόσεις: **Mainline** και **Stable**. [Πατήστε εδώ](https://www.nginx.com/blog/nginx-1-6-1-7-released) για να δείτε ποια έκδοση σας ταιριάζει.

    wget https://nginx.org/download/nginx-1.16.1.tar.gz
    tar xfv nginx-1.16.1.tar.gz
θα χρειαστούμε άλλες 3 βιβλιοθήκες τις οποίες χρησιμοποιεί ο nginx:  

    wget https://www.openssl.org/source/openssl-1.1.1d.tar.gz && tar xzvf openssl-1.1.1d.tar.gz  
    wget https://www.zlib.net/zlib-1.2.11.tar.gz && tar xzvf zlib-1.2.11.tar.gz  
    wget https://ftp.pcre.org/pub/pcre/pcre-8.44.tar.gz && tar xzvf pcre-8.44.tar.gz  

Για να μεταγλωττίσουμε τον nginx είναι αναγκαία μερικά πακέτα όπως gcc, make κτλ:

    sudo apt install build-essential -y

Τέλος, έμεινε να ρυθμίσουμε τις παραμέτρους μεταγλώττισης για τον compiler gcc(προαιρετικό).

    export CFLAGS='-Ofast -pipe -fomit-frame-pointer -march=native -fPIE -fstack-protector-strong -Wformat -Werror=format-security -Wdate-time -D_FORTIFY_SOURCE=2'  
    export CXXFLAGS="${CFLAGS}"
    export LDFLAGS='-Wl,-Bsymbolic-functions -fPIE -pie -Wl,-z,relro -Wl,-z,now'

Για περισσότερες πληροφορίες, μπορείτε να διαβάσετε περισσότερα εδώ https://developers.redhat.com/blog/2018/03/21/compiler-and-linker-flags-gcc και εδώ https://wiki.gentoo.org/wiki/GCC_optimization .  

## Βήμα 2ο: Μεταγλώττιση ##
Πρώτα κάνουμε configure που θα θέσει τις απαραίτητες οδηγίες για την μεταγλώττιση. Αυτό το βήμα είναι το πιο σημαντικό
μιας και εδώ θα ορίσουμε τι θα υποστηρίζει ο ngixn, ποια modules θα είναι διαθέσιμα(στατικά ή δυναμικά - static / dynamic),
ποιοι θα είναι οι φάκελοι εγκατάστασης και άλλα:

    ./configure 
        --with-openssl=../openssl-1.1.1d 
        --with-pcre=../pcre-8.44 
        --with-zlib=../zlib-1.2.11 
        --user=www-data 
        --group=www-data 
        --build=my-nginx 
        --with-openssl-opt=no-nextprotoneg 
        --with-openssl-opt=no-weak-ssl-ciphers 
        --with-pcre-jit 
        --with-file-aio  
        --with-threads 
        --with-http_gzip_static_module 
        --with-debug 
        --with-http_ssl_module 
        --with-http_geoip_module 
        --with-http_v2_module

Η πλήρης λίστα με τις διαθέσιμες επιλογές βρίσκεται online https://nginx.org/en/docs/configure.html ή δίνοντας 

    ./configure --help

Τα παραπάνω είναι ενδεικτικά και εξαρτάται από την χρήση που θα κάνετε, τα εξωτερικά modules που θέλετε να είναι διαθέσιμα.
Για παράδειγμα αν θέλουμε συμπίεση brotli τότε προσθέτουμε ``--add-module=../ngx_brotli`` ή ``--add-dynamic-module=../brotli`` για δυναμικά modules που τα φορτώνουμε με την παράμετρο ``load_module``

    load_module /path/to/modules/ngx_http_execute_module.so;

Αφού τελειώσει χωρίς σφάλματα προχωράμε στην μεταγλώττιση:

    make -j 4
Η παράμετρος ``-j 4`` είναι προαιρετική και έχει να κάνει με τους διαθέσιμους πυρήνες του CPU σας επιταχύνοντας έτσι την μεταγλώττιση. Μπορείτε να το βρείτε δίνοντας:
    
    lscpu  |grep 'CPU(s)'
    CPU(s):              4

Όταν το make τελειώσει χωρίς σφάλματα, κάνουμε εγκατάσταση δίνοντας:

    sudo make install

Ο προεπιλεγμένος φάκελος εγκατάστασης είναι ``/usr/local/nginx/`` αλλά αυτό αλλάζει με την παράμετρο ``--prefix=/path``.
Εκεί μέσα θα βρείτε και το αρχείο ``nginx.conf`` που βρίσκονται οι ρυθμίσεις του nginx. Οι προεπιλεγμένες ρυθμίσεις είναι αρκέτε για να ξεκινήσετε
και μπορείτε να βεβαιωθείτε ότι όλα δουλεύουν σωστά με την εντολή:

    sudo /usr/local/nginx/sbin/nginx -t

Τέλος, ξεκινάμε τον nginx:

    sudo /usr/local/nginx/sbin/nginx

## Βήμα 3ο(προαιρετικό): Ενσωμάτωση module κατά την μεταγλώττιση

Σαν παράδειγμα θα προσθέσουμε το brotli(συμπίεση δεδομένων) στον nginx ως στατικό module με την παράμετρο ``--add-module=`` . Στο κεντρικό φάκελο κατεβάστε με git το project:

    git clone https://github.com/eustas/ngx_brotli.git
    cd ngx_brotli && git submodule update --init

Προσθέτουμε την παράμετρο ``--add-module=../ngx_brotli`` στο configure του nginx και επαναλαμβάνουμε το 2ο βήμα.

Στο άρχειο ρυθμίσεων του nginx μέσα στο http{..} θα πρέπει να προσθέσουμε τα εξής:

    sudo nano /usr/local/nginx/conf/nginx.conf
    brotli on;
    brotli_comp_level 11;    # τιμές μεταξύ 1 και 11
    brotli_types text/plain text/css application/javascript application/json image/svg+xml application/xml+rss;

Εδώ μπορείτε να δείτε ποιοι browsers υποστηρίζουν την συμπίεση brotli: https://caniuse.com/#feat=brotli.

Για να δείτε αν δουλεύει η συμπίεση brotli ανοίξτε την ιστοσελίδα σας και μέσα από τα developers tools ψάξτε για τη λέξη ``br`` στα
response headers.

Happy compiling!

Θεόδωρος Δεληγιαννίδης
thiodor@gmail.com
