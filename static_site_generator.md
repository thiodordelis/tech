title: Γράφοντας ένα static page generator από το μηδέν
author: Theodoros Deligiannidis
date: 25 Mar 2020

Σίγουρα static page generators υπάρχουν πάρα πολλοί, αλλά μιας και μένουμε σπίτι αυτό τον καιρό, είπα να πιάσω κάτι καινούργιο.

Οι επιλογές για blog πλατφόρμες είναι πάρα πολλές και σίγουρα ο #1 προορισμός είναι το wordpress το οποίο είναι too much για το σκοπό που θέλω. Θα έλεγα ότι μου αρμόζει περισσότερο η πλατφορμα [ghost](https://ghost.org) αλλά και πάλι τα ~30$/μήνα πολλά. Στο παρελθόν είχα δοκιμάσει το [Jekyll](https://jekyllrb.com/) αλλά αν θυμάμαι καλά, είχα ένα πρόβλημα με τα αρχεία markdown και να πω την αλήθεια, ποτέ δεν μου άρεσε η ruby!

## Concept ##
Το concept είναι απλό: Γράφεις τα άρθρα σου σε markdown, τρέχεις μια CLI εφαρμογή και σου φτιάχνει ένα αρχείο HTML με όλα τα άρθρα σε μορφή blog.

Για το σκοπό αυτό, το Node.js πιστεύω είναι το καταλληλότερο.

## Πως δουλεύει ##
Η εφαρμογή βασίζεται κυρίως στο [showdown.js](https://github.com/showdownjs/showdown) , του οποίου η δουλειά είναι να μετατρέπει markdown σε html. Επειδή στην πορεία είδα ότι χρειάζομαι έναν τρόπο να αποθηκεύω και metadata μέσα στο markdown αρχείο αλλά να μην φαίνονται στο τελικό αποτέλεσμα, αποφάσισα να τα συμπεριλάβω σε μορφή κλειδιών / τιμής στην αρχή του αρχείου και να τα επεξεργάζομαι με το parse-markdown-metadata. Τα metadata αυτά θα μπορούν να είναι οτιδήποτε πχ title, author, date, tags κτλ.

Τέλος χρειαζόμαστε μια templating γλώσσα, ώστε να γράψουμε τον σκελετό της σελίδας σε html και να ορίσουμε πως θα φαίνονται οι τίτλοι, το κείμενο, η ημερομηνία και φυσικά το layout(css). Τη δουλειά αυτή θα την αναλάβει η [Handlebars](https://handlebarsjs.com) .

## Κώδικας ##
Πρώτα θα πρέπει να κάνουμε εγκατάσταση μέσω του npm τα πακέτα από τα οποία εξαρτάται η εφαρμογή.

    npm install showdown jsdom js-beautify glob handlebars parse-markdown-metadata

και φυσικά να τα χρησιμοποιήσουμε:

    const fs = require('fs');
    const glob = require('glob');
    const beautify = require('js-beautify').html;
    const showdown = require('showdown');
    const ParseMarkdownMetadata = require('parse-markdown-metadata');
    const Handlebars = require("handlebars");

H function που μετατρέπει το markdown σε html είναι:

    function toHtml(input)
    {
        converter.setFlavor('vanilla');
        return converter.makeHtml(input);
    }

Ανάλογα με το spec του markdown που θέλουμε, το ορίζουμε με την αντίστοιχη επιλογή ``setFlavor('vanilla')`` (περισσότερα [εδώ](https://github.com/showdownjs/showdown#flavors) ).

Έπειτα, θα αρχικοποιήσουμε τη showdown με το αρχείο html που θέλουμε να έχουμε σαν οδηγό.

    let html = fs.readFileSync('index.html').toString();
    let template = Handlebars.compile(html);
    let data = {"posts":[]};

Το αρχείο html μπορεί να μοιάζει κάπως έτσι:

    <!DOCTYPE html>
    <html lang="en">
        <head>
            <title>Renderview - Just a static page generator</title>
        </head>
        <body>
            {{#posts}}
            <div id="block">
                <h1 id="title">{{title}}</h1>
                <div id="author">{{author}}</div>
                <div id="date">{{date}}</div>
                <div id="content">{{{content}}}</div>
                <div>{{email}}</div>
                <hr>
            </div>
            {{/posts}}
        </body>
    </html>

Έδω στην ουσία λέμε την handlebars οτι μέσα στο αρχείο html, θα πρέπει να αντικαταστήσεις οτιδήποτε υπάρχει μέσα σε αγκύλες με τιμές τις οποίες θα τις βρείς στη μεταβλητή data. Επειδή προκειται για blog, τότε ο κώδικας HTML που περικλείεται μέσα στα ``{{#posts}} {{/posts}}`` , θα πρέπει να τον επαναλάμβάνεται(loop) για όσα αρχεία md υπάρχουν μέσα στο root φάκελο. Μια μικρή λεπτομέρεια είναι ότι το content, δηλαδή το κείμενο του άρθρου θα πρέπει να είναι ανάμεσα σε τριπλές αγκύλες ``{{{ }}}`` διότι δε θελουμε να γίνει escaped απο την handlebar, το θέλουμε άθικτο(περισσότερα εδώ)

Τέλος έμεινε να γράψουμε τον κώδικα που θα βρίσκει όλα τα αρχεία που τελειώνουν σε .md μέσα στο φάκελο μας, να τραβήξει τα metadata που υπάρχουν αλλά και το κυρίως κείμενο το οποίο θα το μετατρέψει σε html. Όλα αυτά θα αποθηκευτούν στο πίνακα(array) data.

Ο πίνακας για παράδειγμα θα έχει αυτά τα περιεχόμενα:

    data = {"posts":[
    {"title":"This is a title", "author":"Theodoros Deligiannidis", "date":"11/01/2020", "content":"<html>...</html>"},
    {...}
    ]};

To loop για όλη την παραπάνω διαδικασία είναι:

    glob(__dirname + '/*.md', {}, (err, files)=>{
        let mdfile,md;
        files.forEach(function (file) {
            mdfile = fs.readFileSync(file).toString();
            md = new ParseMarkdownMetadata(mdfile);
            md.props.content = toHtml(md.content);
            data.posts.push(md.props);
        });
        let result = template(data);
        let stream = fs.createWriteStream(outputFilename);

        stream.once('open', function(fd) {
            stream.end(beautify(result, { indent_size: 2, space_in_empty_paren: true }));   
        });
    });

Το νέο αρχείο html που δημιουργείται, βρίσκεται στο φάκελο _public και είναι έτοιμο να το ανεβάσουμε στο github ή όπου αλλού θέλουμε! Στο τέλος θα παρατηρήσατε ότι έκανα και ένα beautify στο αρχείο html για να διαβάζεται πιο εύκολα αλλά και για να έχει σωστή στοίχιση.

Τρέχουμε την εφαρμοφή δίνοντας:

    node app.js

Όπως είδαμε, με node.js, λίγα πακέτα και λίγο φαντασία, γράφουμε ένα ισάξιο κλώνο του Jekyll. Βέβαια, λείπουν αρκετά χαρακτηριστικά όπως tags και η σειρά των άρθρων αλλά αυτά αργότερα, σε ένα άλλο άρθρο!

Ακολουθούν όλα τα αρχεία.

## app.js ##
    const fs = require('fs');
    const glob = require('glob');
    const beautify = require('js-beautify').html;
    const showdown  = require('showdown');
    const ParseMarkdownMetadata = require('parse-markdown-metadata');
    const Handlebars = require("handlebars");

    let publicDir = './_public';
    let outputFilename = publicDir+'/index.html'

    const conf = {};
    let converter = new showdown.Converter(conf);

    if (!fs.existsSync(publicDir)){
        fs.mkdirSync(publicDir);
    }

    function toHtml(input)
    {
        converter.setFlavor('vanilla');
        return converter.makeHtml(input);
    }   

    let html = fs.readFileSync('index.html').toString();
    let template = Handlebars.compile(html);
    let data = {"posts":[]};

    glob(__dirname + '/*.md', {}, (err, files)=>{
        let mdfile,md;
        files.forEach(function (file) {
            mdfile = fs.readFileSync(file).toString();
            md = new ParseMarkdownMetadata(mdfile);
            md.props.content = toHtml(md.content);
            data.posts.push(md.props);
        });
        let result = template(data);
        let stream = fs.createWriteStream(outputFilename);

        stream.once('open', function(fd) {
            stream.end(beautify(result, { indent_size: 2, space_in_empty_paren: true }));   
        });
    });

## myfirstpost. md ##

    title: My first post
    author: Theodoros Deligiannidis
    date: 20/03/2020
    email: thiodor@gmail.com
    
    #Hello word

## index_skeleton.html ##
    <!DOCTYPE html>
    <html lang="en">
        <head>
            <meta charset="UTF-8">
            <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
            <meta http-equiv="X-UA-Compatible" content="ie=edge">
            <link rel="stylesheet" href="https://unpkg.com/sakura.css/css/sakura.css" type="text/css">
            <title>Just a blog</title>
        </head>
        <body>
            {{#posts}}
            <div id="block">
                <h1 id="title">{{title}}</h1>
                <div id="author">{{author}}</div>
                <div id="date">{{date}}</div>
                <div id="content">{{{content}}}</div>
                <div>{{email}}</div>
                <hr>
            </div>
            {{/posts}}
        </body>
    </html>
    
Happy compiling!