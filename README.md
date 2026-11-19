# TP3 : DAST avec OWASP ZAP

Le rendu du TP (configuration, qualification des alertes ZAP, priorisation, CI) est dans [RAPPORT.md](RAPPORT.md).

L'application scannée est [TIWAP](https://github.com/tombstoneghost/TIWAP), un lab volontairement vulnérable : à ne lancer qu'en local.

## Lancer le projet

```bash
podman compose up -d --build
```

TIWAP répond sur https://localhost:5000 (certificat autosigné), avec le compte `admin` / `admin`.

Scan ZAP de référence, rapports écrits dans `reports/` :

```bash
podman compose exec zap zap-baseline.py -t https://app:5000 -m 2 -r zap-report.html -J zap-report.json
```
