# Rendu TP3 Zap

## Configuration de départ

Pas besoin d'exposer le service db, théoriquement le container app peut communiquer avec db sans que ce dernier ouvre ses ports. En pratique, cela se vérifie : localhost:27017 n'est pas accessible depuis l'host; le site localhost:5000 peut se connecter à la DB (logs de connexion visible et authentifications possible).

## Ajout de Zap

La commande fournie échoue car http n'est pas supporté par le serveur, et les chemins de sortie sont incorrects.

```diff
pc exec zap \     
  zap-baseline.py \
-  -t http://app:5000 \
+  -t https://app:5000 \
  -m 2 \
-  -r /zap/wrk/zap-report.html \
+  -r zap-report.html \
-  -J /zap/wrk/zap-report.json
+  -J zap-report.json
```

`app.py` lance Flask avec un `ssl_context` (ligne 913) : le port 5000 ne parle que HTTPS. Quant aux chemins, `zap-baseline.py` écrit ses rapports relativement à `/zap/wrk`, donc `-r /zap/wrk/zap-report.html` part dans un sous-dossier qui n'existe pas.

Scan du 16/11/2026, ZAP 2.17.0 : 18 alertes, dont 4 de risque moyen, 8 faibles et 6 informatives. Rapports : [zap-report.html](reports/zap-report.html), [zap-report.json](reports/zap-report.json).

## Interprétation des résultats

### Tableau de qualification

J'ai gardé les alertes qui disent quelque chose sur l'application. Les vérifications sont faites à la main avec curl, sorties dans [verifications-manuelles.txt](reports/preuves/verifications-manuelles.txt).

| Alerte constatée | Risque indiqué (confiance) | Vérification nécessaire | Action envisagée |
| --- | --- | --- | --- |
| 10099 Source Code Disclosure - SQL, `POST /login` | Moyen (moyenne) | Rejouée : un login avec un utilisateur inconnu renvoie une 500 et la page du debugger Werkzeug, avec la pile d'appels et le code de `check_login` (`db_manager.py:45`). La requête affichée est paramétrée : ce n'est pas une injection, c'est une fuite d'information. **Confirmée.** | Corriger : désactiver `debug=True` (`app.py:913`) et gérer le cas `fetchone() is None` dans `check_login` |
| 90022 Application Error Disclosure, `POST /login` | Faible (moyenne) | Même cause que la précédente. En plus, la page contient `EVALEX = true` : la console Python interactive du debugger est activée, et `/console` répond 200 (verrouillée par un PIN). **Confirmée.** | Corriger avec la précédente. Une console Python derrière un PIN reste une porte vers l'exécution de code |
| 10202 Absence of Anti-CSRF Tokens, formulaire de `/` | Moyen (faible) | Le formulaire de login n'a pas de jeton. Le risque réel est un « login CSRF », faible ici. SonarQube avait déjà relevé la protection CSRF désactivée (`app.py:28`). **Confirmée, impact limité.** | Corriger : réactiver Flask-WTF `CSRFProtect` |
| 10038 CSP Header Not Set et 10020 Missing Anti-clickjacking Header | Moyen (haute / moyenne) | Aucun en-tête de sécurité dans la réponse de `/` (voir curl). Observation directe, sans doute possible. L'impact dépend des XSS présentes : le TP2 en a montré au moins une (`\| safe`). **Confirmée.** | Corriger : ajouter les en-têtes dans l'application (`after_request`) ou au reverse proxy |
| 10035 Strict-Transport-Security Header Not Set | Faible (haute) | Confirmée. Le certificat est auto-signé et l'application n'est pas exposée : HSTS n'a de sens qu'avec un vrai certificat. | Justifier pour le labo, corriger en production (au reverse proxy) |
| 10036 Server Leaks Version Information | Faible (haute) | `Server: Werkzeug/2.0.3 Python/3.6.15`. Confirmée. Révèle surtout que le serveur de développement tourne et que Python 3.6 n'est plus maintenu. | Corriger : servir l'application avec gunicorn derrière un proxy, mettre Python à jour |
| 10027 Suspicious Comments | Info (moyenne) | Une occurrence est le commentaire JS du debugger (« Prevent page from refreshing »), l'autre est la trace d'erreur déjà vue. **Faux positif** pour la première. | Aucune action propre, traitée avec 10099 |

Le reste (Cross-Origin-*-Policy, Permissions-Policy, X-Content-Type-Options, cache) relève du même correctif d'en-têtes. Les URL en `/usr/local/lib/python3.6/...` viennent des liens présents dans la page d'erreur que le spider a suivis : ce ne sont pas de vraies routes.

### Pourquoi une alerte doit être vérifiée

L'alerte 10099 en est un bon exemple. Son titre parle de SQL, et on pourrait la lire comme une injection. En rejouant la requête, on voit autre chose : ZAP a reconnu un `SELECT` dans le texte d'une page d'erreur. La requête est paramétrée, donc pas injectable. Le vrai problème est le mode debug, qui affiche le code source et propose une console Python. Le niveau « moyen » de ZAP sous-estime ce risque : sur une application exposée, c'est le constat le plus grave du rapport.

À l'inverse, une partie de l'alerte 10027 est un faux positif, et HSTS n'a pas de sens sur un labo en certificat auto-signé. Trier ces cas demande de connaître le contexte, ce que l'outil n'a pas.

Pour l'en-tête CSP (question 8.3) :

1. Toutes les réponses HTML, dont la page de login `/`.
2. Dans le labo, non : seul l'hôte y accède (`127.0.0.1:5000`). Pour une application NOVA exposée aux clients, oui.
3. Sans CSP, une XSS peut charger et exécuter n'importe quel script. L'en-tête ne corrige pas la XSS, il en limite l'impact.
4. Les deux sont possibles. Le reverse proxy couvre toutes les applications d'un coup ; l'application connaît mieux ses propres besoins (scripts inline, CDN).
5. Relancer le scan de référence : l'alerte 10038 doit disparaître. Un `curl -I` suffit pour un contrôle rapide.

### Lien avec l'injection SQL du TP2

**Le scan n'a détecté aucune injection SQL.** C'était prévisible :

- le Baseline Scan est passif : il n'envoie pas de charge `' OR '1'='1` et ne compare pas les réponses ;
- le scan n'est pas authentifié, alors que les pages vulnérables (`/sqli/low`, etc.) sont derrière le login. Le spider n'a vu que `/`, `/login` et les fichiers statiques.

SonarQube ne l'avait pas trouvée non plus (pas d'analyse de propagation dans l'édition Community). C'est la revue de code du TP2 qui l'a confirmée, avec un test unitaire. Les deux outils se complètent quand même : ZAP a montré ce que SonarQube ne voit pas (le comportement réel du serveur en debug, les en-têtes renvoyés), et SonarQube voit le code que ZAP n'atteint pas. Pour confirmer une injection en dynamique, il faudrait un scan actif authentifié, ou une approche IAST qui observe l'application de l'intérieur pendant les tests.

## Priorisation pour NOVA SERVICES

L'ordre de ZAP ne convient pas tel quel. Le mode debug arrive en « moyen » et « faible », alors que c'est le seul constat qui mène à l'exécution de code ; les en-têtes manquants, eux, ont un impact limité tant qu'il n'y a pas de XSS à exploiter. Les priorités ci-dessous supposent une application NOVA exposée à ses clients, avec des données personnelles.

| Champ | Debug Werkzeug actif | En-têtes de sécurité absents | Pas de jeton anti-CSRF |
| --- | --- | --- | --- |
| Référence | ZAP 10099 + 90022, `POST /login` | ZAP 10038, 10020, 10021, 90004, 10063 | ZAP 10202, formulaire `/` (et SonarQube python:S4502) |
| Application | TIWAP, cible de laboratoire | TIWAP, cible de laboratoire | TIWAP, cible de laboratoire |
| Actif métier analogue | Portail client NOVA, même stack Flask | Toutes les applications web NOVA | Portail client et back-office des prestations |
| Impact | Fuite du code et des chemins internes ; la console Python, si son PIN est deviné ou calculé, donne l'exécution de code sur le serveur (confidentialité, intégrité, disponibilité) | Une XSS existante devient exploitable sans restriction ; la page peut être encadrée pour du clickjacking | Un site tiers peut faire envoyer des formulaires par un utilisateur connecté (modification de données) |
| Priorité | **Critique** : exploitable sans compte, depuis la page de login, correction triviale | **Moyenne** : pas exploitable seule, mais amplifie les XSS déjà relevées au TP2 | **Élevée** sur les formulaires authentifiés, faible sur le login seul |
| Responsable | Développement (code), exploitation (configuration de lancement) | Exploitation (reverse proxy), développement pour la CSP | Développement |
| Échéance | 18/11/2026 (24 h) | 01/12/2026 | 15/12/2026 |
| Action | Retirer `debug=True`, servir avec gunicorn ; traiter `fetchone() is None` dans `check_login` | Ajouter les en-têtes au reverse proxy ; écrire une CSP adaptée aux scripts de l'application | Réactiver `CSRFProtect` de Flask-WTF et ajouter `{{ csrf_token() }}` aux formulaires |
| Vérification | Nouveau scan : 10099 et 90022 absents ; `curl` du login inconnu doit renvoyer « Invalid Credentials » en 200 | Nouveau scan : alertes d'en-têtes absentes ; `curl -I` | Nouveau scan (10202 absent) et test : un POST sans jeton est refusé (400) |
| Statut | Ouvert | Ouvert | Ouvert |

Dans le labo lui-même, ces trois défauts sont acceptés : TIWAP est vulnérable exprès et n'écoute que sur `127.0.0.1`.

## Intégration continue

Le sujet laisse le choix entre GitLab CI et GitHub Actions. J'ai pris GitHub Actions, comme pour le TP2 : [.github/workflows/dast-zap.yml](.github/workflows/dast-zap.yml).

Le job `dast_zap` tourne à chaque push, sur toutes les branches, et sur les pull requests :

1. il démarre `app`, `db` et `zap` avec le même `docker-compose.yml` qu'en local ;
2. il attend que `https://localhost:5000` réponde (30 essais de 5 s), et affiche les journaux de l'application sinon ;
3. il lance le scan de référence avec `-I` : les avertissements ne cassent pas le build, seules des règles passées en FAIL le feraient ;
4. il écrit le tableau des alertes dans le résumé du run et publie les rapports HTML et JSON en artefact `zap-rapports` ;
5. il arrête tout avec `docker compose down -v`, même en cas d'échec.

Le runner ne publie rien : le port 5000 reste lié à `127.0.0.1` de la machine virtuelle, détruite à la fin du job.

Limites :

- le job ne bloque rien pour l'instant. Pour bloquer, il faudrait un fichier de règles (`-c`) qui passe en FAIL les alertes qualifiées, par exemple 10099 et 90022 ;
- l'artefact n'alimente aucun tableau de bord de vulnérabilités. GitHub attend du SARIF pour son onglet *Code scanning*, GitLab un rapport DAST dans son propre format ;
- le scan n'est pas authentifié, comme en local.

## Indicateurs de pilotage

Valeurs de départ tirées de ce TP (un seul scan, une seule application). Le nombre brut d'alertes n'y figure pas : 18 alertes aujourd'hui, 25 demain avec un scan authentifié, cela voudrait dire une meilleure couverture, pas une application moins sûre.

| Indicateur | Définition et calcul | Source | Fréquence | Responsable | Décision permise | Valeur TIWAP |
| --- | --- | --- | --- | --- | --- | --- |
| Vulnérabilités confirmées ouvertes, par priorité | Nombre de fiches au statut « ouvert » ou « en cours », regroupées par priorité | Fiches de traitement (tickets) | Hebdomadaire | Équipe sécurité | Réaffecter des développeurs quand le stock critique ou élevé ne baisse pas | 1 critique, 1 élevée, 1 moyenne |
| Vulnérabilités prioritaires en retard | Fiches critiques ou élevées dont l'échéance est dépassée, sur le total des critiques et élevées | Fiches de traitement | Hebdomadaire | Responsable sécurité applicative | Escalader au management ; bloquer la mise en production concernée | 0 / 2 (échéances non atteintes) |
| Corrections vérifiées par un nouveau scan | Fiches « corrigé » dont l'alerte a disparu au scan suivant, sur toutes les fiches « corrigé » | Fiches + rapports JSON de la CI | À chaque pipeline | Équipe de développement | Rouvrir une fiche dont la correction n'a pas tenu | Pas encore mesurable (aucune correction) |
| Délai médian de correction | Médiane de (date de vérification − date de détection) sur les fiches fermées du trimestre | Fiches de traitement | Mensuelle | Responsable sécurité applicative | Ajuster les échéances par priorité, ou les moyens si elles sont irréalistes | Pas encore mesurable |
| Couverture DAST des applications critiques | Applications critiques avec au moins un scan réussi dans les 30 derniers jours, sur toutes les applications critiques | Inventaire des applications + historique des pipelines | Mensuelle | Équipe sécurité | Prioriser l'ajout du job DAST dans les projets non couverts | 1 / 1 (le labo) |
| Taux d'échec technique des scans | Jobs `dast_zap` en échec hors alertes (application qui ne démarre pas, ZAP qui plante), sur tous les jobs lancés | Historique GitHub Actions | Mensuelle | Exploitation | Fiabiliser le pipeline avant d'en tirer des chiffres | À mesurer sur les prochains runs |

## Réponses aux questions

### Partie 1

**Q1. Pourquoi démarrer et vérifier l'application avant de lancer ZAP ?** ZAP ne lit pas le code, il interroge le serveur. Si l'application ne répond pas, ou répond par une erreur, le spider ne trouve rien et le rapport est vide ; on risquerait de prendre ce rapport vide pour un résultat propre. Ici, la vérification a aussi montré que le serveur ne parlait que HTTPS, ce qui a changé la cible du scan.

**Q2. Pourquoi ne pas exposer le port MongoDB à l'hôte ?** L'application joint `db` par le réseau interne de Compose, elle n'a pas besoin du port publié. Publier 27017 ajoute une surface d'attaque, avec des identifiants triviaux (`username` / `password` dans le Compose), et sur certaines configurations Docker le port contourne le pare-feu de l'hôte. J'ai vérifié que `localhost:27017` est fermé et que l'application se connecte toujours à la base.

### Questions de compréhension

1. **SAST et DAST.** Le SAST (SonarQube) lit le code sans l'exécuter : il voit tous les chemins, y compris ceux qu'aucun utilisateur n'atteint, mais pas la configuration réelle du serveur. Le DAST (ZAP) parle à l'application en marche, comme un client HTTP : il voit ce que le serveur renvoie vraiment, mais seulement sur les pages qu'il atteint.
2. **Pourquoi une application en fonctionnement ?** Parce que ZAP n'a pas d'autre matière que les requêtes et les réponses. Le mode debug de TIWAP, par exemple, ne se voit qu'à l'exécution : rien dans le code ne dit si on le lance en production avec `debug=True`.
3. **Pourquoi `http://app:5000` ?** Dans un conteneur, `localhost` désigne le conteneur lui-même. Compose crée un réseau où chaque service est joignable par son nom, et le DNS interne résout `app` vers le conteneur de TIWAP. Ici c'était `https://app:5000`, le serveur ne parlant que TLS.
4. **Passif et actif.** Le scan passif observe le trafic généré par l'exploration et en tire des constats (en-têtes, cookies, fuites). Le scan actif envoie des charges d'attaque (injections, XSS) ; il peut modifier des données ou faire tomber l'application, d'où un périmètre autorisé et un environnement préparé.
5. **Pourquoi le Baseline ne garantit pas de trouver une injection SQL ?** Il n'injecte rien : une injection ne se révèle que si l'on envoie une entrée piégée et qu'on compare la réponse. Il faudrait en plus atteindre la page vulnérable ; dans TIWAP, elles sont derrière le login et le scan n'était pas authentifié. Résultat : aucune alerte d'injection dans notre rapport.
6. **Alerte et vulnérabilité confirmée.** Une alerte est un constat de l'outil, avec son niveau de confiance. Elle devient une vulnérabilité confirmée quand quelqu'un l'a rejouée et a établi l'impact. L'alerte 10099 parlait de « SQL » ; la vérification a montré une fuite d'information par le debugger, pas une injection.
7. **Prioriser sur une application critique.** On part du risque ZAP, puis on le corrige selon le contexte : exposition (Internet ou interne), données touchées, facilité d'exploitation (faut-il un compte ?), mesures compensatoires (WAF, proxy), coût de correction. Le debug Werkzeug, « moyen » pour ZAP, passe en critique : il est exploitable sans compte et se corrige en une ligne.
8. **Pourquoi conserver les rapports en CI ?** Pour comparer deux scans et prouver qu'une correction a tenu, pour garder une trace d'audit liée à un commit, et pour ne pas mettre les rapports dans le dépôt : ils décrivent les failles de l'application et doivent rester réservés à l'équipe.
9. **Éviter de publier une application vulnérable.** Lier les ports à `127.0.0.1`, ne pas publier la base, faire tourner les scans sur des runners ou des réseaux isolés, détruire l'environnement après le job (`docker compose down -v`), et ne jamais déployer TIWAP avec un pipeline de production.
10. **Indicateurs d'efficacité.** Ceux qui mesurent le traitement plutôt que le volume : délai médian de correction, vulnérabilités prioritaires en retard, pourcentage de corrections vérifiées par un nouveau scan, couverture des applications critiques. Voir le tableau ci-dessus.

## Conclusion

Le scan de référence a trouvé en deux minutes un défaut que SonarQube n'avait pas classé comme grave : le serveur de développement Flask tourne en debug et affiche son code, avec une console Python, à quiconque rate son login. Il a aussi listé les en-têtes absents, des constats sûrs mais de faible impact seuls.

Il n'a vu ni l'injection SQL ni la XSS du TP2. Il était passif, non authentifié, et il n'a exploré que la page de login. Une absence d'alerte ne dit donc rien sur les pages derrière le login.

Les deux outils couvrent des angles différents : le SAST lit tout le code mais pas le serveur, le DAST voit le serveur réel mais seulement ce qu'il atteint. Pour NOVA SERVICES, aucun des deux ne garantit la sécurité ; leurs rapports servent à ouvrir des fiches, à les prioriser et à vérifier les corrections. La suite logique serait un scan authentifié, puis un scan actif sur un environnement de recette.

## Nettoyage

```bash
podman compose down
podman compose ps
```

Le volume MongoDB peut être gardé : il ne contient que les données de démonstration de TIWAP.
