# ASGOR

*Automated Security Grading Of Repositories* : ASGOR analyse un dépôt GitHub public et lui donne une note de sécurité (secrets, dépendances vulnérables, code), avec un rapport détaillé.

![Page d'accueil d'ASGOR : champ pour l'URL du dépôt, options d'analyse et liste des analyses récentes](https://raw.githubusercontent.com/brahimmachkouri/asgor-app/refs/heads/main/docs/apercu-accueil.png)

Ce dépôt sert à **installer ASGOR sur son poste** avec Docker. Le code source, les tests et la construction de l'image sont dans [asgor-app](https://github.com/brahimmachkouri/asgor-app). Pour corriger les dépendances signalées par un rapport, voir [asgor-tools](https://github.com/brahimmachkouri/asgor-tools).

---

## Prérequis

- **Docker** : Docker Desktop récent (Mac), ou Docker Engine 26 ou plus récent avec Docker Compose (Linux). Machines Intel/AMD (amd64) et ARM (arm64, dont les Mac Apple Silicon). Windows n'a pas été testé.
- **Au moins 8 Go de mémoire donnés à Docker** (Docker Desktop : *Settings › Resources*). L'analyse PHP (Psalm) en a besoin.
- **Environ 5 Go de place disque** : image (1,3 Go), bases de failles téléchargées à la première analyse (2 Go environ), dépôts analysés (500 Mo au plus chacun, effacés après l'analyse) et rapports.
- Un accès à Internet : ASGOR télécharge les dépôts à analyser, les bases de failles et les règles d'analyse.

---

## Installation

```bash
git clone https://github.com/brahimmachkouri/asgor.git
cd asgor
```

**Linux seulement** : donner à ASGOR le groupe propriétaire du socket Docker.

```bash
echo "DOCKER_GID=$(stat -c %g /var/run/docker.sock)" > .env
```

Télécharger l'image, vérifier sa provenance (voir plus bas), puis démarrer :

```bash
docker compose pull
docker compose up -d
```

Au premier démarrage, ASGOR télécharge les règles d'analyse de Semgrep (quelques Mo) et vérifie leur somme SHA256. L'interface est ensuite sur **http://127.0.0.1:8000**, accessible depuis le poste seulement.

Pour vérifier l'isolation des analyses sur le poste (autotest) :

```bash
docker compose exec worker python3 -m asgor_moteur.autotest_docker
```

---

## Première analyse

Ouvrir http://127.0.0.1:8000, coller l'adresse d'un dépôt GitHub public (par exemple `https://github.com/brahimmachkouri/depot-test-analyse`, un dépôt rempli de failles volontaires) et lancer l'analyse. Suivre l'avancement sur la page, puis lire le rapport. Les analyses passent une par une ; la page « Aide » du site explique les rapports et les notes.

La première analyse est plus longue : ASGOR télécharge les bases de failles.

En ligne de commande :

```bash
docker compose exec worker asgor-scan https://github.com/brahimmachkouri/depot-test-analyse
```

ASGOR interroge l'API de GitHub sans jeton : 60 requêtes par heure et par adresse IP. Si beaucoup d'analyses partent de la même adresse (même réseau), une analyse peut être refusée ; réessayer plus tard.

---

## Vérifier la provenance de l'image

Chaque version est construite et testée par GitHub Actions dans [asgor-app](https://github.com/brahimmachkouri/asgor-app/actions/workflows/image.yml), puis signée par une attestation de provenance (SLSA). Pour vérifier que l'image vient bien de ce workflow, à cette étiquette, avec le [CLI GitHub](https://cli.github.com/) :

```bash
gh attestation verify \
  oci://ghcr.io/brahimmachkouri/asgor@sha256:7db9bcc28540af06be6014bc844e79681e14681ac7ce8ad7ce8a2c7d97fac010 \
  --repo brahimmachkouri/asgor-app \
  --signer-workflow brahimmachkouri/asgor-app/.github/workflows/image.yml \
  --source-ref refs/tags/v0.1.0
```

La commande doit réussir (code de retour 0). L'empreinte est celle de `compose.yaml` ; Docker refuse toute autre image.

---

## Règles d'analyse de Semgrep : licence

Pour l'analyse du code, ASGOR utilise Opengrep avec les règles du dépôt [semgrep/semgrep-rules](https://github.com/semgrep/semgrep-rules). Ces règles sont publiées sous la [Semgrep Rules License v1.0](https://semgrep.dev/legal/rules-license), qui n'est pas une licence libre :

- **l'image ne contient pas les règles** : chaque installation les télécharge elle-même, depuis le dépôt de Semgrep, à son premier démarrage. Chaque personne qui installe ASGOR obtient donc sa propre copie et **doit respecter cette licence** ;
- la licence permet d'utiliser les règles **pour les besoins internes de son organisation** (« for your own internal business purposes ») ;
- elle **interdit de les distribuer et de les mettre à disposition d'autres personnes sous forme de service**. Ouvrir une instance d'ASGOR à des utilisateurs extérieurs à son organisation pourrait entrer dans ce cas ;
- les rapports reprennent le texte des règles (explication, références) : ne pas les publier.

Ceci est un résumé, pas un avis juridique : seul le texte de la licence fait foi.

---

## Mise à jour

```bash
git pull
docker compose pull
docker compose up -d
```

Vérifier la provenance de la nouvelle image avant `up` (même commande, avec la nouvelle empreinte et la nouvelle étiquette). La base, les rapports et le cache sont gardés dans des volumes Docker.

---

## Désinstallation

```bash
docker compose down              # arrête ASGOR, garde les données
docker compose down --volumes    # arrête ASGOR et supprime toutes ses données (base, rapports, cache, règles)
docker image rm ghcr.io/brahimmachkouri/asgor@sha256:7db9bcc28540af06be6014bc844e79681e14681ac7ce8ad7ce8a2c7d97fac010
```

---

## Versions publiées

| Version | Date | Image |
|---|---|---|
| 0.1.0 | 20/11/2026 | `ghcr.io/brahimmachkouri/asgor:0.1.0@sha256:7db9bcc28540af06be6014bc844e79681e14681ac7ce8ad7ce8a2c7d97fac010` |

---

Licence : MIT (voir [`LICENSE`](LICENSE)). Elle couvre ce dépôt et ASGOR ; les outils d'analyse et les règles de Semgrep ont leurs propres licences (voir plus haut).
