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

| Alerte constatée | Risque indiqué | Vérification nécessaire | Action envisagée |
| --- | --- | --- | ---|





