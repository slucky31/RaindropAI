# RaindropAI

Outil .NET 10 qui trie automatiquement le backlog de la collection **« Non trié »** de Raindrop.io. Il apprend vos collections et tags réels (via l'API Raindrop), classifie chaque nouvel article avec Claude Haiku en s'appuyant sur cette taxonomie, puis applique directement le résultat : tags fusionnés et déplacement vers la collection identifiée si elle correspond — sans étape de validation manuelle. Une notification Discord signale en plus les articles jugés prioritaires à tester, et un digest email quotidien récapitule tout le reste.

Tout ce qui se trouve **en dehors** de « Non trié » est considéré comme déjà classé par vos soins et n'est jamais retouché.

Voir [docs/adr/](docs/adr/) pour le détail des décisions d'architecture, notamment [0007](docs/adr/0007-apprentissage-taxonomie-non-trie.md) pour la logique d'apprentissage de la taxonomie. Pour contribuer, voir [docs/versioning.md](docs/versioning.md) : le titre des Pull Requests suit SemVer via Conventional Commits et pilote la version publiée automatiquement.

## Architecture

```
src/RaindropAI.Core            modèles, enums, interfaces (zéro dépendance externe)
src/RaindropAI.Infrastructure  Raindrop API (raindrops + collections + tags), classification Anthropic, persistance SQLite, notifications
src/RaindropAI.Worker          Worker Service (Coravel pour la planification, Serilog pour les logs)
tests/                         xUnit.v3 + NSubstitute + WireMock.Net
docs/adr/                      Architecture Decision Records
```

## Roadmap

L'outil classe et range, mais n'exploite pas encore ce qu'il a accumulé. [docs/roadmap.md](docs/roadmap.md) cartographie les scénarios envisagés pour synthétiser, nettoyer et relire ce contenu (serveur MCP, revues thématiques, détection de doublons, file de lecture), priorisés valeur/effort et découpés en lots livrables.

## Prérequis

- .NET 10 SDK (dev local)
- Docker + Docker Compose (déploiement, notamment sur Raspberry Pi 64-bit / arm64)
- Un token API Raindrop.io (App Management Console → Test token)
- Une clé API Anthropic
- Un webhook Discord
- Un compte SMTP pour l'envoi du digest

## Configuration

Copier `.env.example` en `.env` et renseigner les valeurs (voir commentaires dans le fichier). Les clés suivent la convention .NET `Section__Propriete` (ex. `Raindrop__Token`, `Email__SmtpHost`).

⚠️ Vérifiez que `Raindrop__CollectionId` vaut bien **`-1`** (« Non trié »). C'est ce réglage qui garantit que l'outil ne touche pas aux raindrops que vous avez déjà rangés : `0` viserait **toute votre bibliothèque**.

## Lancer en local

```bash
dotnet build RaindropAI.slnx
dotnet test RaindropAI.slnx
dotnet run --project src/RaindropAI.Worker
```

En local, `appsettings.Development.json` pointe vers un fichier SQLite `raindropai.dev.db` dans le dossier courant et des logs dans `logs/`.

Le worker **refuse de démarrer** si la configuration est incomplète (token Raindrop, clé Anthropic, webhook Discord, SMTP) et indique précisément le champ fautif — par exemple `DataAnnotation validation failed for 'RaindropApiOptions' members: 'Token'`. Renseignez les valeurs via `dotnet user-secrets` ou `.env` avant de lancer.

## Déploiement sur Raspberry Pi

Rien n'est compilé sur le Pi : l'image est construite et publiée par la CD GitHub sur `ghcr.io`, en multi-arch (`linux/amd64` + `linux/arm64`).

Pour un déploiement de zéro sur un Raspberry Pi fraîchement installé (installation de Docker incluse, récupération des secrets, dépannage), voir [docs/deploiement-raspberry-pi.md](docs/deploiement-raspberry-pi.md). Version courte ci-dessous pour une mise à jour ou un Pi déjà équipé de Docker :

```bash
uname -m                                        # doit renvoyer aarch64 (Raspberry Pi OS 64-bit)
mkdir -p data && sudo chown -R 1654:1654 data   # le conteneur tourne en non-root (uid 1654)
docker compose pull
docker compose up -d
docker compose logs -f
```

Pour épingler une version plutôt que de suivre `latest` : `RAINDROPAI_TAG=0.3.0 docker compose up -d` (ou la variable dans le `.env`).

Le conteneur s'exécute sous l'utilisateur applicatif non-root de l'image .NET (`uid 1654`), à partir d'une image « chiselée » sans shell ni gestionnaire de paquets. Sur un bind mount, c'est la propriété côté hôte qui prime : sans le `chown` ci-dessus, SQLite échoue à créer sa base avec un `Permission denied` sur `/data`. Si vous mettez à jour une installation existante qui tournait en root, appliquez le `chown` sur le dossier `data/` déjà présent.

Pour reconstruire l'image localement malgré tout (mise au point) :

```bash
docker build -f src/RaindropAI.Worker/Dockerfile -t raindropai-worker:local .
```

Le fichier SQLite (`/data/raindropai.db`) et les logs (`/data/logs/`) sont persistés via le volume `./data`.

## ⚠️ Premier lancement

Sans état de polling préexistant, l'outil remonte **tout l'historique** de la collection « Non trié » au premier cycle (aucun webhook natif disponible côté API, cf. [ADR 0003](docs/adr/0003-strategie-polling-raindrop.md)). Si elle contient déjà beaucoup d'articles, cela peut être long et générer un volume d'appels LLM important, **et modifier automatiquement un grand nombre de raindrops d'un coup** (tags + déplacements). Pour éviter un traitement massif au premier lancement, insérez manuellement une ligne dans `PollingState` avant de démarrer :

```sql
INSERT INTO PollingState (Id, LastRaindropId, LastCreatedUtc, UpdatedAtUtc)
VALUES (1, <id_du_dernier_raindrop_a_ignorer>, '<date_ISO8601_UTC>', '<date_ISO8601_UTC>');
```

## Comment le tri est appliqué

À chaque cycle (`Worker__PollingCronExpression`, toutes les 15 min par défaut) :
1. Apprentissage de la taxonomie réelle : collections (`GET /collections` + `/collections/childrens`) et tags (`GET /tags`) existants.
2. Pour chaque nouvel article de « Non trié », le LLM propose une collection existante (ou aucune), des tags, une action (à lire / à tester / référence) et une priorité.
3. Les tags proposés sont **toujours** appliqués (fusionnés avec les tags déjà présents, jamais de perte). Le raindrop n'est **déplacé** que si la collection proposée correspond exactement à une collection existante ; sinon il reste dans « Non trié » avec juste les tags. La note existante est complétée, jamais écrasée.

`Worker__WriteBackToRaindrop=false` désactive cette application automatique (mode classification + rapport seulement, rien n'est modifié dans Raindrop) — utile pour observer le comportement avant de laisser l'outil toucher à vos données. Actif (`true`) par défaut.

## Tests automatisés

```bash
dotnet test RaindropAI.slnx
```

Aucune vraie clé nécessaire : les appels Raindrop/Anthropic/Discord sont simulés via WireMock.Net, la base est un fichier SQLite temporaire par test.

## Vérification manuelle bout-en-bout (avec de vraies clés)

À faire avant un déploiement réel, pour observer le comportement sur votre compte Raindrop :

1. Renseigner dans `src/RaindropAI.Worker/appsettings.Development.json` (ou via `dotnet user-secrets`, jamais commité) : `Raindrop__Token`, `Classifier__ApiKey`, `Discord__WebhookUrl`, `Email__Smtp*`.
2. Pour ne pas attendre 15 min / 24h pendant les tests, surcharger temporairement les expressions cron :
   ```json
   "Worker": {
     "PollingCronExpression": "* * * * *",
     "DigestCronExpression": "*/2 * * * *",
     "WriteBackToRaindrop": false
   }
   ```
   Garder `WriteBackToRaindrop` à `false` le temps d'observer les suggestions dans les logs avant de laisser l'outil modifier vos raindrops.
3. Lancer :
   ```bash
   dotnet run --project src/RaindropAI.Worker
   ```
4. Observer les logs (console + `logs/raindropai-*.log`) : nombre de nouveaux articles détectés dans « Non trié », nombre de collections/tags appris, notifications envoyées.
5. Ajouter un raindrop test dans « Non trié » via l'app Raindrop pendant que le worker tourne, attendre le prochain cycle.
6. Inspecter `raindropai.dev.db` (créé à la racine du projet en dev) avec DB Browser for SQLite ou `sqlite3` — vérifier les colonnes `SuggestedCollection`/`SuggestedTags`/`RecommendedAction`/`Priority`/`Reason`.
7. Repasser `WriteBackToRaindrop` à `true` pour valider l'application réelle (tags + déplacement) sur un raindrop de test, puis vérifier dans l'app Raindrop que le résultat correspond à la ligne SQLite.
8. Vérifier la réception Discord (si le raindrop test matche le seuil ATester/Haute) et le digest email (regroupement par collection puis action).

⚠️ Ce test réel consomme de vrais appels à l'API Anthropic (coût minime mais réel) et modifie votre vrai compte Raindrop dès que `WriteBackToRaindrop=true`.
