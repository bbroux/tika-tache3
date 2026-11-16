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

    1. Désérialisation dangereuse

        On veut construire un objet java à partir d'un fichier sérializé donné par un utilisateur, mais on ne connait pas les intentions de cet utilisateur. À partir du fichier sérialisé, un attaquant pourrait accéder à l'application et, par exemple, exécuter du code ou accéder à des informations privilégiées. 
        Avec l'objet ObjectInputStream de Java, on a ce danger, mais tika n'utilise pas cette façon de désérialiser. On va donc l'introduire dans tika-serialization/src/main/java/org/apache/tika/config/loader/TikaObjectMapperFactory.java, où l'on fait déjà de la désérialisation avec d'autres méthodes. On ajoute la méthode deserializeObject, dans laquelle on va créér un ObjectInputStream et tenter de le lire afin de déclencher l'erreur suivante de Semgrep: java.lang.security.audit.object-deserialization.object-deserialization 

        Le run Semgrep #6 fonctionne, et on reconnait l'erreur. 

    2. Manque de validation pour certificats

        On peut implémenter un gestionnaire de certificats qui ne fait aucune/pas assez de vérifications et donc qui pourrait permettre a un adversaire d'intercepter de l'information ou de faire des partages dangereux. Dans tika-core/src/main/java/org/apache/tika/parser/NetworkParser.java, on fait déjà des connexions réseau, donc il est naturel d'ajouter la vulnérabilité ici. On ajoute une méthode qui retourne un OK pour toutes les demandes, donc qui valide tous les certificats. Ceci devrait lancer une vulnérabilité de type java.lang.security.audit.crypto.ssl.insecure-trust-manager.insecure-trust-manager dans semgrep. 






