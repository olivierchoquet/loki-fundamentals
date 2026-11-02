# Démo Loki – Carnivorous Greenhouse

Montrer en 15 minutes comment Loki permet de trouver l’origine d’une erreur dans une application en microservices.

**Fil conducteur :** « une application a un problème, comment le trouve-t-on grâce aux logs ? ». Chaque composant de l’architecture apparaît au moment où il sert : l’agent collecte (Alloy), Loki stocke et interroge, Grafana affiche.

## Préparation (30 minutes avant)

Tout doit tourner avant l’arrivée du public, pour que les images soient téléchargées et que des logs existent déjà.

1. Lancer la stack d’observabilité :
   ```bash
   docker compose up -d
   ```
2. Lancer l’application :
   ```bash
   docker compose -f greenhouse/docker-compose-micro.yml up -d --build
   ```
3. Vérifier que les conteneurs `greenhouse-*`, `loki`, `grafana` et `alloy` sont lancés :
   ```bash
   docker ps --format '{{.Names}}'
   ```
4. Ouvrir les onglets du navigateur dans l’ordre de la démo :
   - Application : http://localhost:5005
   - Alloy : http://localhost:12345/graph
   - Grafana Explore : http://localhost:3000
5. Créer un compte dans l’application et cliquer un peu partout pour produire des logs.
6. Dans Grafana : plage de temps *Last 15 minutes*, rafraîchissement automatique activé.
7. Garder les [requêtes LogQL](#requêtes-logql) sous la main pour les copier-coller.

Checklist :

- [ ] Stack lancée
- [ ] Application lancée
- [ ] Onglets ouverts
- [ ] Mode erreur de l’application sur *Off*
- [ ] Option 2 testée (message d’erreur repéré)
- [ ] Captures d’écran de secours prêtes

## Déroulé (15 minutes)

| Étape | Durée | Ce qu’on montre | Message clé |
|-------|-------|-----------------|-------------|
| 1. Le décor | 2 min | `docker-compose.yml` et `docker ps` | D’un côté l’application, de l’autre la stack d’observabilité |
| 2. L’application | 2 min | Application sur le port 5005 : créer un compte, ajouter des plantes | On génère des logs en direct |
| 3. Le trajet des logs | 2 min | Graphe Alloy (port 12345), clic sur `discovery.docker` pour voir les conteneurs trouvés | Découvrir → étiqueter → lire → traiter → envoyer |
| 4. Explorer | 3 min | Grafana Explore, source Loki, *Label browser*, label `container` | Loki indexe les labels, pas le contenu des logs |
| 5. L’incident | 4 min | [Provoquer des erreurs](#provoquer-lincident-étape-5), puis requêtes 1 à 3 | On réduit le bruit jusqu’à la ligne d’erreur précise |
| 6. Des logs aux métriques | 2 min | Requête 4 en mode graphe | Un pic d’erreurs visible, base d’une alerte (Ruler) |

### Le graphe Alloy (étape 3)

1. `discovery.docker` trouve les conteneurs via le socket Docker.
2. `discovery.relabel` transforme les métadonnées Docker en labels (dont `container`).
3. `loki.source.docker` lit les logs de chaque conteneur.
4. `loki.process` traite les lignes (parsing, filtres, labels).
5. `loki.write` envoie le résultat à Loki.

> Les flèches vers `discovery.*` indiquent une dépendance, pas le trajet des logs.

### Provoquer l’incident (étape 5)

**Option 1 : le mode erreur de l’application (recommandé).**
L’application contient un *Bug Service* qui, une fois activé, fait échouer les services au hasard et génère des logs d’erreur.

1. Dans l’application (port 5005), mettre **Toggle Error Mode** sur *On*.
2. Continuer à utiliser l’application pendant une minute.
3. Lancer les requêtes 1 à 3, puis la requête 4 en graphe : le pic d’erreurs apparaît au moment de l’activation.

> Vérifier que le mode erreur est bien sur *Off* avant la démo, pour que la courbe « avant / après » soit nette.

**Option 2 : arrêter un service à la main.**
Plus parlant pour montrer une démarche de diagnostic : « l’application plante → les logs de `main_app` pointent vers `plant_service` → ce conteneur ne répond plus ».

```bash
docker stop greenhouse-plant_service-1
```

Puis le relancer après la démonstration :

```bash
docker start greenhouse-plant_service-1
```

> À tester avant la démo : repérer le message d’erreur exact produit par `main_app` (`ERROR`, `Connection refused`…) pour savoir sur quel mot filtrer.

## Requêtes LogQL

**Requête 1** : tous les logs de l’application.

```logql
{container=~"greenhouse-.*"}
```

**Requête 2** : seulement les erreurs.

```logql
{container=~"greenhouse-.*"} |= "ERROR"
```

**Requête 3** : les erreurs d’un seul service.

```logql
{container="greenhouse-user_service-1"} |= "ERROR"
```

**Requête 4** : nombre d’erreurs par conteneur, à afficher en graphe.

```logql
sum by (container) (count_over_time({container=~"greenhouse-.*"} |= "ERROR" [1m]))
```

> `{container=~".*"}` est refusé par Loki car `.*` accepte une valeur vide : utiliser `.+` pour voir tous les conteneurs.

## Plan B et dépannage

| Problème | Solution |
|----------|----------|
| Aucun log dans Explore | Élargir la plage de temps (*Last 1 hour*), vérifier que les composants Alloy sont *Healthy* |
| Le label `container` n’apparaît pas | Vérifier `discovery.relabel` dans le graphe Alloy |
| Drilldown → Logs vide | Ne pas l’utiliser : Explore et le label browser suffisent |
| Pas d’erreurs dans les logs | Interagir davantage avec l’application, ou filtrer sur un autre mot (`\|= "error"`, `\|= "WARN"`) |
| Docker ou réseau en panne | Basculer sur les captures d’écran de secours |
