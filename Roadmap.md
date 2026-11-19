# SambaPME — Roadmap Purple Team (7 jours)

Plan de travail unique à suivre, jour par jour. Les détails fins (commandes exactes, payloads) sont affinés au fil de l'eau ; ce document fixe le périmètre, l'ordre et les livrables de chaque journée.

**Principe général** : chaque finding suit la boucle *attaque → télémétrie observée → règle de détection → correctif → re-test*. Les branches sont : `main` (baseline durcie) → `vuln` (régressions volontaires + attaques) → `hardened` (correctifs + rejeu des attaques).

---

## J1 — Déploiement baseline durci

**Objectif** : rendre l'environnement crédible avant toute attaque. Ubuntu et Kali sont déjà provisionnés, réseau privé en place — pas de recréation de VM.

### À faire

- Centraliser les configs dans `configs/` (source unique) et les référencer depuis `setup_samba.sh` via une fonction `push_conf` (scp + install + `testparm -s` avant restart).
- Firewall sur la VM (nftables/ufw) : 445/139 et 5000 limités au LAN du lab, 22 restreint à un hôte. Attention à Docker, qui publie ses ports en contournant ufw → filtrer via la chaîne `DOCKER-USER`.
- **Durcir `vagrant`** : remplacer le `NOPASSWD: ALL` par un sudoers restreint aux commandes réellement utilisées par l'app (liste dans `admin_logic.py`). Remplacer les `grep`/`tail` composés par des scripts dédiés whitelistés — un `grep` générique autorisé en sudo peut lire n'importe quel fichier, donc un sudoers restreint sans ce nettoyage ne protège rien.
- **Mots de passe** : individualiser — garder le mot de passe faible commun (`Samba2026!`) sur la moitié des comptes de démo (réalisme), mots de passe forts et distincts sur l'autre moitié. Ne pas tous varier : ça casserait le scénario de spraying de J2.
- `.env` : vérifier qu'il est dans `.gitignore`, fournir un `.env.example`, générer un vrai `FLASK_SECRET_KEY` aléatoire pour la baseline (remis à `change_me` uniquement sur `vuln`).
- Télémétrie minimale : `vfs_full_audit` dans `smb.conf`, `auditd` avec watches sur `/etc/samba`, `/usr/local/bin`, `/srv/samba`, logs Flask JSON avec IP cliente réelle (`request.remote_addr`).
- **Corriger le bug de bannissement** : fail2ban voit l'IP du bridge Docker/VM au lieu de l'attaquant réel. Ajouter `ignoreip` pour ce sous-réseau côté jail Samba, et créer un second jail fail2ban sur les logs Flask JSON (`LOGIN_FAIL` avec IP réelle) pour bannir côté web aussi.
- Remplacer le volume nommé des logs par un bind mount (ex. `/var/log/sambapme`) pour que fail2ban puisse les lire depuis l'hôte.

### Relance propre

```
make infra   # reprovisionne via setup_samba.sh
make sync    # copie le projet dans la VM
make up      # docker compose up -d --build (--force-recreate si .env a changé)
docker compose ps   # doit être "healthy"
```

Ne jamais faire `down -v` (perte des volumes sessions/audit).

### Tests de validation (depuis Kali)

1. `nmap` sans filtre → confirmer que seuls les ports voulus répondent une fois le firewall posé.
2. `smbclient -L -N` → `ACCESS_DENIED` attendu (contrôle déjà validé, à documenter comme test de non-régression, pas comme vulnérabilité).
3. Brute force SMB → IP Kali bannie (`fail2ban-client status samba`).
4. Échecs de login répétés sur le portail web → IP Kali bannie, pas le conteneur.
5. Pendant le ban de Kali, un compte légitime (`direction1`) se connecte toujours normalement.
6. `fail2ban-regex` sur `log.*` confirme que le filtre matche bien le format des logs.

### Snapshot & tag

Snapshot libvirt `v1-baseline` après validation, puis :

```
git add configs/ infra/ .gitignore .env.example
git commit -m "baseline: firewall, sudoers restreint, auditd, fix fail2ban"
git tag -a v1-baseline -m "J1 validated"
```

---

## J2 — Surface Samba (branche `vuln`)

```
git checkout -b vuln v1-baseline
```

### Régressions volontaires (commits dédiés, revertables individuellement)

- `F-S01` : ouvrir `hosts allow` sur `commun` (sans restriction IP).
- `F-H01` : revenir temporairement à `NOPASSWD: ALL` pour `vagrant` (lab only — nécessaire pour démontrer H-01/H-02).
- Option : repasser `log level` sans format JSON `auth_json_audit`, pour montrer que fail2ban ne détecte alors plus rien (bon contraste pour le rapport).
- *Ne pas* recréer d'énumération anonyme : `restrict anonymous = 2` fonctionne déjà et doit rester tel quel — c'est un contrôle validé, pas un finding.

### Findings à démontrer

- **S-01 (révisé)** : test de contrôle — énumération anonyme bloquée. Documenté comme preuve de robustesse, pas comme vulnérabilité.
- **S-02 — Spraying distribué** : identifiants faibles partagés, attaque répartie sur plusieurs IP (alias sur l'interface Kali) pour rester sous le seuil fail2ban par source. Finding principal : détection comportementale manquante (fail2ban ne voit qu'une IP à la fois).
- **Énumération authentifiée** : une fois un compte obtenu, `smbclient -L` révèle tous les partages y compris ceux hors périmètre de l'utilisateur (fuite d'organisation).
- **S-03 — Upload piégé dans `commun`** : dépôt via `smbclient` d'un fichier à double extension ou script (ex. `facture.pdf.exe`). Cible directe du FIM/YARA de J5.
- **S-04 — Exfiltration massive** : `mget *` récursif sur un partage. Sert à calibrer un seuil de détection volumétrique (lectures/minute par utilisateur via `full_audit`).

### Preuves à collecter par finding

Capture Kali (commande + sortie), état fail2ban avant/après, extrait `log.smbd`/`full_audit`, horodatage (pour corrélation avec FIM/YARA).

### Fiches & tag

`docs/findings/S-01.md` (contrôle validé), `S-02.md`, `S-03.md`, `S-04.md`, `H-01.md` (sudo excessif — bascule ici ou documentée pleinement en J4).

```
git tag -a v2a-samba -m "Samba findings done"
```

---

## J3 — Surface web (branche `vuln`)

Les failles sont déjà présentes dans le code (pas de création nécessaire, sauf `F-W00`).

### Régression volontaire

- `F-W00` : `FLASK_SECRET_KEY=change_me` (lab only).

### Findings à démontrer

- **W-01 — Bypass admin** : le mot de passe saisi est ignoré pour le compte `admin` (seul le SSH `.env` compte) → session admin obtenue sans connaître le vrai mot de passe.
- **W-02 — XSS stockée via upload** : upload SMB d'un `.html` piégé dans `commun`, exécuté via `/preview/commun/<fichier>` sans sandboxing MIME. Chaîne complète : upload SMB → XSS web → vol de session (idéalement la session admin volée en W-01).
- **W-03 — Log poisoning → XSS admin** : nom d'utilisateur piégé dans une tentative de login SMB échouée, stocké tel quel dans `log.smbd`, injecté en `innerHTML` par `admin.js` → exécution dans le navigateur admin.
- **W-04 — Path traversal** : tentative sur `/preview/<share>/../../etc/passwd`. Si `smbprotocol` normalise déjà le chemin côté serveur, documenter l'échec comme contrôle validé plutôt que de forcer un faux positif.
- **W-05 — CSRF** : absence de token sur les actions sensibles (ex. `/delete`), page piégée hébergée sur Kali déclenchant une suppression à l'insu d'une victime connectée.

### Fiches & tag

`docs/findings/W-01..W-05.md`, puis :

```
git tag -a v2b-web -m "Web findings done"
```

---

## J4 — Surface hôte & chaîne complète (branche `vuln`)

### Findings à démontrer

- **H-01 — Sudo excessif** : `vagrant` avec `NOPASSWD: ALL` (régression F-H01 de J2) → root immédiat depuis des identifiants SSH par défaut.
- **H-02 — Chaîne complète web → hôte → root** (scénario vitrine pour la démo finale) : W-01 (devenir admin web) → lecture du `.env` via une faille annexe (session mal protégée dans `/tmp/flask_sessions_samba`, ou traversal) → récupération de `SSH_PASSWORD` → SSH direct → sudo sans mot de passe → root.
- **H-03 — Persistance** : création d'un utilisateur Samba via le formulaire admin légitime (`/admin/users/add`) une fois l'accès admin obtenu — persistance sans exploit, juste un abus de fonctionnalité.
- **H-04 — Altération des preuves** : depuis root, édition/suppression de `audit.log`, des logs Samba, voire du script de sauvegarde. Justifie le FIM côté configs/logs (pas seulement côté partages) en J5.

### Tag de fin de branche

```
git tag -a v2-vuln -m "all attacks demonstrated"
```

---

## J5 — Blue : détection, FIM, YARA

### Composants

- **FIM temps réel** : `watchdog` (inotify) sur `/srv/samba`, scan YARA à chaque création/modification de fichier → détecte S-03.
- **Baseline de hashes** (AIDE ou script sha256 maison) sur `/etc/samba`, `/etc/fail2ban`, `/usr/local/bin`, `audit.log` → détecte H-04.
- **Règle comportementale spraying** : lecture de `log.smbd`/`full_audit`, comptage des échecs distincts par utilisateur toutes IP confondues sur une fenêtre glissante → détecte S-02 même distribué, ce que fail2ban seul rate.
- **Règle web** : alerte sur tout login admin (volume attendu faible, donc chaque occurrence est significative) → détecte W-01.
- **YARA** : intégration du code déjà testé par ailleurs, branché sur le hook FIM ; règles pour webshells, double extension, macros auto-exécutées.
- **Canary files** : un fichier leurre par partage, alerte sur toute lecture.
- Sortie unique : `alerts.jsonl`, consultable en terminal (`jq`/`lnav`). Onglet admin « Détections » (lecture seule, échappé) uniquement si le temps le permet — ne pas sacrifier le reste pour ça.

---

## J6 — Branche `hardened`

```
git checkout -b hardened v2-vuln
```

Un commit de correctif par finding, avec revert explicite des régressions de J2/J3 :

- sudoers restreint (retour à l'état baseline J1) ;
- mots de passe individualisés sur tous les comptes ;
- vrai contrôle du mot de passe admin (comparaison effective, pas juste tentative de connexion SSH) ;
- sanitation de l'upload (whitelist d'extensions/MIME, `Content-Disposition: attachment` forcé pour tout type non confirmé sûr) ;
- échappement des logs dans `admin.js` (texte, jamais `innerHTML`) ;
- CSRF token (Flask-WTF ou équivalent maison) ;
- `FLASK_SECRET_KEY` aléatoire obligatoire, permissions strictes sur le dossier de sessions ;
- `ignoreip` et jail web restaurés (déjà posés en J1, à revalider).

**Rejeu** : chaque script d'attaque de J2–J4 doit être rejoué. Critère de succès : l'alerte apparaît dans `alerts.jsonl` **avant** que l'attaque échoue (preuve de détection, pas seulement de blocage).

```
git tag -a v3-hardened -m "fixes + detections validated"
```

---

## J7 — Documentation & démo finale

- Finaliser toutes les fiches `docs/findings/*.md` (preuve, impact, mapping ATT&CK, correctif, statut détecté/bloqué).
- `REPORT.md` de synthèse avec tableau avant/après : détecté (oui/non), délai de détection, attaque bloquée (oui/non) — un tableau par surface (Samba/Web/Hôte).
- Mapping MITRE ATT&CK consolidé (T1135, T1110.003, T1105, T1020/T1048, etc.).
- Script de démo rejouable (`make attack-S02`, `make attack-W01`, etc.).
- Nettoyage : retirer les `__pycache__` du dépôt, corriger le README (arborescence `infra/`, `.env.example` manquant).
- Marge de sécurité pour les dépassements de J4 (jour historiquement le plus à risque).

---

## Repères rapides

| Jour | Branche | Tag de sortie |
| --- | --- | --- |
| J1 | `main` | `v1-baseline` |
| J2 | `vuln` | `v2a-samba` |
| J3 | `vuln` | `v2b-web` |
| J4 | `vuln` | `v2-vuln` |
| J5 | `vuln` (outils blue, mode observe) | — |
| J6 | `hardened` | `v3-hardened` |
| J7 | `main` (merge final) | — |
