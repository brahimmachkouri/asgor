# ASGOR

*Automated Security Grading Of Repositories* : ASGOR analyse un dépôt GitHub public et lui donne une note de sécurité (de **A** à **F**, inspirée de SSL Labs) sur trois axes — **secrets**, **dépendances** et **code** — avec un rapport détaillé et des pistes de correction.

![Page d'accueil d'ASGOR : champ pour l'URL du dépôt, options d'analyse et liste des analyses récentes](https://raw.githubusercontent.com/brahimmachkouri/asgor-app/refs/heads/main/docs/apercu-accueil.png)

Ce dépôt sert à **installer et exécuter ASGOR en local sur son poste** avec Docker. Le code source, les tests, le barème (`BAREME.md`), le guide d'exploitation et la construction de l'image sont dans [`asgor-app`](https://github.com/brahimmachkouri/asgor-app). Pour corriger les dépendances signalées par un rapport, voir [`asgor-tools`](https://github.com/brahimmachkouri/asgor-tools).

---

## Langages et dépendances pris en charge

* **Code analysé (SAST)** : **PHP en priorité** (notamment **Symfony** et **Laravel**) et **Python**. Les autres langages présents dans un dépôt ne sont pas couverts avec le même niveau d'analyse statique.
* **Dépendances (CVE / GHSA)** : vérifiées à partir des fichiers de verrouillage (*lockfiles*) pour **npm**, **pnpm**, **yarn**, **Composer** (`composer.lock`) et **pip**.
* **Secrets** : recherche de clés, jetons et mots de passe dans les fichiers et l'historique Git de la branche par défaut (les branches jamais fusionnées ne sont pas inspectées).
* **Fonctionnement prudent** : aucun code du dépôt analysé n'est exécuté, aucune configuration du dépôt cible n'est appliquée et chaque étape tourne dans un environnement isolé.

---

## Prérequis

* **Docker** : Docker Desktop récent (macOS) ou Docker Engine 26+ avec Docker Compose (Linux). Compatible **amd64** et **arm64** (dont Apple Silicon). Windows n'a pas été testé.
* **Au moins 8 Go de mémoire alloués à Docker** (par exemple dans *Settings › Resources* sur Docker Desktop), requis notamment pour l'analyse PHP.
* **Environ 5 Go d'espace disque** : image Docker (~1,3 Go), bases de vulnérabilités téléchargées au premier scan (~2 Go), dépôts clonés temporairement (500 Mo maximum chacun, supprimés après analyse) et rapports.
* **Accès Internet** : nécessaire pour télécharger l'image, les règles d'analyse, les bases de failles et les dépôts GitHub publics à auditer.

---

## Installation

```bash
git clone [https://github.com/brahimmachkouri/asgor.git](https://github.com/brahimmachkouri/asgor.git)
cd asgor
```

**Sur Linux uniquement**, renseignez le GID propriétaire du socket Docker :

```bash
echo "DOCKER_GID=$(stat -c %g /var/run/docker.sock)" > .env
```

Téléchargez l'image, vérifiez son attestation de provenance (voir section dédiée), puis démarrez les services :

```bash
docker compose pull
docker compose up -d
```

Au premier démarrage, ASGOR récupère les règles Semgrep et vérifie leur empreinte SHA256. L'interface web est ensuite disponible sur **[http://127.0.0.1:8000](http://127.0.0.1:8000)** (écoute locale uniquement).

Pour vérifier l'isolation des analyses sur le poste :

```bash
docker compose exec worker python3 -m asgor_moteur.autotest_docker
```

---

## Première analyse

1. Ouvrez **[http://127.0.0.1:8000](http://127.0.0.1:8000)**.
2. Collez l'URL d'un dépôt GitHub public (par exemple `https://github.com/brahimmachkouri/depot-test-analyse` pour tester sur un dépôt contenant des vulnérabilités volontaires).
3. Lancez l'analyse, suivez sa progression puis consultez le rapport (les analyses sont traitées séquentiellement). La page **Aide** détaille le barème et la lecture des rapports.

En ligne de commande :

```bash
docker compose exec worker asgor-scan [https://github.com/brahimmachkouri/depot-test-analyse](https://github.com/brahimmachkouri/depot-test-analyse)
```

> **Quota GitHub API** : sans jeton d'authentification, l'API GitHub est limitée à **60 requêtes par heure et par adresse IP**. Sur un réseau partagé ou lors de scans répétés, des limitations temporaires peuvent survenir.

---

## Vérifier la provenance de l'image

Chaque version est construite et testée par GitHub Actions dans [`asgor-app`](https://github.com/brahimmachkouri/asgor-app/actions/workflows/image.yml) puis signée avec une attestation SLSA. Vérifiez-la avec GitHub CLI (`gh`) :

```bash
gh attestation verify \
  oci://ghcr.io/brahimmachkouri/asgor@sha256:7db9bcc28540af06be6014bc844e79681e14681ac7ce8a2c7d97fac010 \
  --repo brahimmachkouri/asgor-app \
  --signer-workflow brahimmachkouri/asgor-app/.github/workflows/image.yml \
  --source-ref refs/tags/v0.1.0
```

Un code de retour `0` confirme l'origine et l'intégrité de l'image référencée dans `compose.yaml`.

---

## Licence des règles d'analyse (Semgrep)

L'analyse statique s'appuie sur Opengrep avec les règles de [`semgrep/semgrep-rules`](https://github.com/semgrep/semgrep-rules), publiées sous **Semgrep Rules License v1.0** (licence non libre) :

* Les règles ne sont pas embarquées dans l'image : chaque installation les télécharge au premier démarrage et doit respecter cette licence.
* L'usage est prévu pour les **besoins internes de votre organisation**.
* La **redistribution ou mise à disposition sous forme de service à des tiers extérieurs** à votre organisation est interdite : évitez d'exposer publiquement une instance ASGOR à des utilisateurs externes.
* Les rapports reprenant des extraits, explications ou références de règles ne doivent pas être publiés.

*Ce résumé ne remplace pas la lecture du texte officiel de la Semgrep Rules License v1.0.*

---

## Mise à jour et désinstallation

### Mise à jour

Vérifiez d'abord l'attestation SLSA de la nouvelle version avant de lancer le conteneur :

```bash
git pull
docker compose pull
docker compose up -d
```

La base, les rapports et le cache sont conservés dans les volumes Docker.

### Désinstallation

```bash
docker compose down              # arrête les conteneurs, conserve les volumes
docker compose down --volumes    # arrête les conteneurs et supprime toutes les données persistées
docker image rm ghcr.io/brahimmachkouri/asgor@sha256:7db9bcc28540af06be6014bc844e79681e14681ac7ce8a2c7d97fac010
```

---

## Versions publiées

| Version | Date | Image |
| --- | --- | --- |
| `0.1.0` | 20/11/2026 | `ghcr.io/brahimmachkouri/asgor:0.1.0@sha256:7db9bcc28540af06be6014bc844e79681e14681ac7ce8a2c7d97fac010` |

---

## Licence

Ce dépôt et ASGOR sont distribués sous licence **MIT** (voir [`LICENSE`](LICENSE)). Les outils embarqués et les règles Semgrep restent soumis à leurs licences respectives.
