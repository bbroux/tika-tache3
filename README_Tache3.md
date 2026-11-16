## Ajout de Semgrep

J'ai commencé en ajoutant Semgrep directement à partir des Github actions. En faisant "New Workflow", j'ai pu direcement ajouter Semgrep et y mettre mes configurations. Voici ce que j'ai choisi et pourquoi: 

    1. Faire l'évaluation sur les "push": Je ne voulais pas le faire à chaque pull request et ensuite le faire au push suivant aussi. De plus, je ne voulais que l'action roule à chaque jour selon un horaire, car je ne l'utilise pas dans un contexte qui le nécéssite et donc ca ne me servirait pas.

    2. Évaluation utilisant scan et p/java: Je n'avais pas besoin de me créer un compte Semgrep et comme nous sommes en java, on évalue les règles propres à ce language. 

    3. Utilisation de la mention "--error": Pour signaler des erreurs et faire échouer le build si des problèmes sont détectés. 

Pour vérifier l'ajout, une première analyse est faite lors du commit du fichier de configuration Semgrep. Pour le moment, on fait une analyse baseline. Par la suite, on va modifier le fichier de configuration semgrep.yml pour que l'analyse touche seulement aux nouvelles modifications du push en question. 

# Rapport initial 

Le premier run nous affiche 11 problèmes détectés. On lui demande de générer un fichier .json avec le log du run. (voir tika/semgrep-report-initial.json) 
Les erreurs détectées par semgrep initialement sont de 8 types différents:

    1. java.lang.security.audit.xxe.saxparserfactory-disallow-doctype-decl-missing.             saxparserfactory-disallow-doctype-decl-missing
    
    2. java.lang.security.audit.crypto.unencrypted-socket.unencrypted-socket

    3. java.lang.security.xmlinputfactory-possible-xxe.xmlinputfactory-possible-xxe

    4. java.lang.security.audit.formatted-sql-string.formatted-sql-string

    5. java.lang.security.audit.crypto.use-of-aes-ecb.use-of-aes-ecb

    6. java.lang.security.audit.xxe.documentbuilderfactory-disallow-doctype-decl-missing.documentbuilderfactory-disallow-doctype-decl-missing

    7. java.lang.security.audit.crypto.use-of-sha1.use-of-sha1

    8. java.lang.security.audit.crypto.unencrypted-socket.unencrypted-socket
    
    On update aussi la configuration semgrep avec --baseline-commit <SHA-commit-initial> pour que par la suite, semgrep détecte seulement les nouvelles vulnérabilités et ne fasse pas échouer le build pour ces 11 vulnérabilités déja trouvées. On peut vérifier que ceci fonctionne lorsqu'on fait le commit du changement dans semgrep.yml. 

## Ajout de vulnérabilités

    


