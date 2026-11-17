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
