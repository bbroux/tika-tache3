## Ajout de Semgrep

J'ai commencé en ajoutant Semgrep directement à partir des Github actions. En faisant "New Workflow", j'ai pu direcement ajouter Semgrep et y mettre mes configurations. Voici ce que j'ai choisi et pourquoi: 

    1. Faire l'évaluation sur les "push": Je ne voulais pas le faire à chaque pull request et ensuite le faire au push suivant aussi. De plus, je ne voulais que l'action roule à chaque jour selon un horaire, car je ne l'utilise pas dans un contexte qui le nécéssite et donc ca ne me servirait pas.

    2. Évaluation utilisant scan et p/java et p/security-audit: Comme nous sommes en java, on évalue les règles propres à ce language. De plus, le security-audit nous permet d'évaluer les risques à la sécurité. 

    3. Utilisation de la mention "--error": Pour signaler des erreurs et faire échouer le build si des problèmes sont détectés. 

Pour vérifier l'ajout, une première analyse est faite lors du commit du fichier de configuration Semgrep. Pour le moment, on fait une analyse baseline. Par la suite, on va modifier le fichier de configuration semgrep.yml pour que l'analyse touche seulement aux nouvelles modifications du push en question. 

# Rapport initial 

Le premier run nous affiche 11 problèmes détectés (Semgrep #2 dans Github actions). On lui demande de générer un fichier .json avec le log du run. (voir tika/semgrep-report-initial.json) 
Les erreurs détectées par semgrep initialement sont de 8 types différents:

    1. java.lang.security.audit.xxe.saxparserfactory-disallow-doctype-decl-missing.             saxparserfactory-disallow-doctype-decl-missing
    
    2. java.lang.security.audit.crypto.unencrypted-socket.unencrypted-socket

    3. java.lang.security.xmlinputfactory-possible-xxe.xmlinputfactory-possible-xxe

    4. java.lang.security.audit.formatted-sql-string.formatted-sql-string

    5. java.lang.security.audit.crypto.use-of-aes-ecb.use-of-aes-ecb

    6. java.lang.security.audit.xxe.documentbuilderfactory-disallow-doctype-decl-missing.documentbuilderfactory-disallow-doctype-decl-missing

    7. java.lang.security.audit.crypto.use-of-sha1.use-of-sha1

    8. java.lang.security.audit.crypto.unencrypted-socket.unencrypted-socket
    
    On update aussi la configuration semgrep avec --baseline-commit <SHA-commit-initial> pour que par la suite, semgrep détecte seulement les nouvelles vulnérabilités et ne fasse pas échouer le build pour ces 11 vulnérabilités déja trouvées. On peut vérifier que ceci fonctionne lorsqu'on fait le commit du changement dans semgrep.yml (Semgrep #3 dans Github actions). Le run Semgrep est vert, c'est-à-dire qu'on a pas trouvé de nouvelle vulnérabilité en comparant avec la version initiale, qui en avait 11. C'est un bon signe, mais on va valider que le tout fonctionne en ajoutant des nouvelles vulnérabilités. 

## Ajout de vulnérabilités

    1. Désérialisation dangereuse - Vulnérabilité 1

        On veut construire un objet java à partir d'un fichier sérializé donné par un utilisateur, mais on ne connait pas les intentions de cet utilisateur. À partir du fichier sérialisé, un attaquant pourrait accéder à l'application et, par exemple, exécuter du code ou accéder à des informations privilégiées. 
        Avec l'objet ObjectInputStream de Java, on a ce danger, mais tika n'utilise pas cette façon de désérialiser. On va donc l'introduire dans tika-serialization/src/main/java/org/apache/tika/config/loader/TikaObjectMapperFactory.java, où l'on fait déjà de la désérialisation avec d'autres méthodes. On ajoute la méthode deserializeObject, dans laquelle on va créér un ObjectInputStream et tenter de le lire afin de déclencher l'erreur suivante de Semgrep: java.lang.security.audit.object-deserialization.object-deserialization 

        Le run Semgrep #6 fonctionne, et on reconnait l'erreur. 

    2. Manque de validation pour certificats - Vulnérabilité 2

        On peut implémenter un gestionnaire de certificats qui ne fait aucune/pas assez de vérifications et donc qui pourrait permettre a un adversaire d'intercepter de l'information ou de faire des partages dangereux. Dans tika-core/src/main/java/org/apache/tika/parser/NetworkParser.java, on fait déjà des connexions réseau, donc il est naturel d'ajouter la vulnérabilité ici. On ajoute une méthode qui retourne un OK pour toutes les demandes, donc qui valide tous les certificats. Ceci devrait lancer une vulnérabilité de type java.lang.security.audit.crypto.ssl.insecure-trust-manager.insecure-trust-manager dans semgrep. 

        Ceci fonctionne, on a 2 erreurs flaggées ici, mais on ne peut pas en retirer une car sinon le tout ne compilera pas. On laisse donc les 2 erreurs et on devrait, a la fin, se retrouver avec 8 erreurs rapportées par semgrep. 

    3. Injection dans une commande système 

        Lorsqu'une entrée utilisateur est ensuite passée à une commande système, un adversaire pourrait y glisser des commandes selon ce qu'il veut faire. Dans tika-parsers\tika-parsers-standard\tika-parsers-standard-modules\tika-parser-ocr-module\src\main\java\org\apache\tika\parser\ocr\TesseractOCRParser.java, il y a déjà des commandes systèmes pour faire rouler tesseract. Il est donc possible de faire des modifications ici pour introduire une vulnérabilité. 
        Je vais donc, dans la méthode doOCR, rajouter un paramètre d'entrée utilisateur et concatener ceci avec la commande qui est utilisée pour créer le processus système. 

        Après quelques essais différents, semgrep ne semble pas reconnaître le problème, même lorsque mon implémentation est très similaire à l'exemple donné dans le Registry Semgrep. Je vais tenter d'introduire une prochaine vulnérabilité pour l'instant. 

        Après plusieurs recherches, j'ai remarqué que ce que j'essayait de faire apparaître viens d'une règle de Semgrep "pro", donc il faut essayer autre chose. Je laisse quand même le fichier avec les modifications afin de garder une trace du travail. 

    4. Génération aléatoire prévisible - Vulnérabilité 3

        Ici, on va essayer d'introduire un générateur aléatoire, qui ne sera pas vraiment aléatoire. La règle java.lang.security.audit.crypto.weak-random.weak-random reconnait l'utilisation de Math.Random() et de java.util.Random(). Ces fonctions peuvent être introduites dans plusieurs classes différentes, mais la classe tika-parsers\tika-parsers-standard\tika-parsers-standard-modules\tika-parser-microsoft-module\src\main\java\org\apache\tika\parser\microsoft\onenote\fsshttpb\util\SequenceNumberGenerator.java semble être une classe candidate car elle sert déjà à générer un nombre. On va retourner un résultat de java.util.Random().nextInt() à la sortie de la méthode "getCurrentToken()". 

        L'erreur est bien reconnue par Semgrep au moment de l'exécution. 

    5. Ajout d'information privilégiée dans le code

        Lorsqu'on met de l'information qui devrait rester secrète dans notre code, le module secrets de semgrep le reconnait, et signale un danger. Il ne faut pas stocker cette information dans le code, car un adversaire pourrait la récupérer et l'utiliser de façon malveillante. Il est préférable de stocker ces informations dans une variable environnement, par exemple. On peut faire ceci dans la classe NetworkParser.java, qu'on a modifié plus tôt pour la vulnérabilité 2. On ajoute une propriété à la connection avec un URL, et cette propriété définit une authentification "hard-coded". 

        La vulnérabilité n'est pas détectée, et je pense que c'est parce que les règles sont incluses dans Semgrep Pro, qui n'est pas la version que j'utilise. Je ne sais pas si ça aurait fonctionné de toute manière, car les règles de "secrets" sont ciblées à des contextes spécifiques et dans le Registry il semble juste avoir une règle pour HttpUrlConnection. On avait modifié un objet UrlConnection. 
        Passons à une prochaine vulnérabilité. 

    6. Parse non sécuritaire de PDF -> XML - Vulnérabilité 4

        Dans la classe tika-parsers\tika-parsers-standard\tika-parsers-standard-modules\tika-parser-pdf-module\src\main\java\org\apache\tika\parser\pdf\XFAExtractor.java, 
        on extrait d'un PDF un formulaire XFA. On parse ceci en XML. Par contre, comme le formulaire XFA pourrait être compromis, il faut faire des validations de sécurité avant de pouvoir créer notre XML. Si on ne fait pas de vérifications, Semgrep reconnait ceci avec la mention java.lang.security.xmlinputfactory-possible-xxe.xmlinputfactory-possible-xxe. On va donc modifier la méthode "extract()", pour introduire ce type de vulnérabilité. On remplace le "getXMLInputFactory()" par "newFactory()", et on va ajouter des propriétés à cette new factory qui laisseront passer les fichiers externes, ce qui devrait créer notre alerte semgrep. 

        Ça fonctionne, et on a maintenant 6 erreurs semgrep détectées (voir run semgrep #13)

    7. Fonction de hachage faible - Vulnérabilité 5

        Ici, on veut introduire une fonction de hachage faible que Semgrep saura reconnaître. Dans la classe, tika-core\src\main\java\org\apache\tika\digest\InputStreamDigester.java, on utilise un MessageDigest pour créer un hash selon un algorithme donné. Si on lui passe un algorithme de hachage faible, Semgrep reconnaît le problème. On va donc essayer de lui passer SHA-1, qui est reconnu comme une fonction de hachage faible. Il devrait nous retourner l'erreur java.lang.security.audit.crypto.use-of-sha1.use-of-sha1

        Ça fonctionne, et on obtient un total de 7 détections avec Semgrep #14. 

    8. CBC padding oracle - Vulnérabilité 6 

        Dans le protocole de cryptographie CBC, on ajoute parfois un padding à la fin pour pouvoir chiffrer des messages de longueur aléatoire avec une clé de taille fixe. Si le programme donne de l'information sur la validité du padding avant que la clé soit acceptée, un adversaire pourrait accéder à de l'information sur la clé secrète en sachant si le padding est valide/invalide. On va donc introduire ce genre de vulnérabilité. Dans la classe tika-core\src\main\java\org\apache\tika\parser\CryptoParser.java, on utilise déjà des méthodes de chiffrement, donc c'est un environnement logique pour l'introduction de cette vulnérabilité. On devrait avoir une détection de type: cbc-padding-oracle.
        Après 2 essais, 1 avec CBC et un avec ECB, on n'a pas d'erreur détectée par Semgrep. 
        Lorsqu'on compare avec le rapport initial, qui trouvait cette erreur mais dans une autre classe (HwpTextExtractor.java), on voit que la mention /NoPadding donnait une erreur détectée par Semgrep. En ajoutant ceci, on peut faire apparaître la nouvelle vulnérabilité dans la classe CryptoParser.java. On va donc garder cette version qui fait apparaître la vulnérabilité dans le run semgrep #19. 

        On a maintenant 9 détections avec Semgrep #19 (celle-ci introduit 2 vulnérabilités, à cause des 2 lignes modifiées - donc c'est normal)
    
    9. Script Engine Injection

        Dans la classe tika-parsers\tika-parsers-standard\tika-parsers-standard-modules\tika-parser-pdf-module\src\main\java\org\apache\tika\parser\pdf\AbstractPDF2XHTML.java

        On donne un PDF a un interpréteur de script, et donc si un attaquant donne quelque chose de non-sécuritaire, c'est exécuté comme un script. On devrait avoir l'erreur java.lang.security.audit.script-engine-injection.script-engine-injection avec cet ajout. 

        Ça fonctionne, et on a maintenant 10 vulnérabilités détectées avec Semgrep! 

# Rapport final 

Pour produire un rapport final, on va retirer le check de baseline-SHA pour le moment, afin de calculer la différence entre le rapport initial et final (avec changements). 
Le rapport final devrait compter 10 erreurs de plus que celui qu'on avait au début, pour un total de 21 détections. On va donc modifier le semgrep.yml pour retirer ceci (on pourra le remettre par la suite) 




## Tests et humour

Pour ajouter quelque chose de simple et humoristique, j'ai décidé de déclencher un RickRoll quand Semgrep ne passe pas, car on sait déjà que ce sera activé pour ce projet. Pour faire ceci, on ajoute un github actions qui s'appelle Rickroll, et on voit que la configuration est assez simple. 

 


