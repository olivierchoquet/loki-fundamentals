# Démo Loki – Carnivorous Greenhouse

Montrer en 15 minutes comment Loki permet de trouver l’origine d’une erreur dans une application en microservices.

**Fil conducteur :** « une application a un problème, comment le trouve-t-on grâce aux logs ? ». Chaque composant de l’architecture apparaît au moment où il sert : l’agent collecte (Alloy), Loki stocke et interroge, Grafana affiche.

## Préparation (30 minutes avant)

Tout doit tourner avant l’arrivée du public, pour que les images soient téléchargées et que des logs existent déjà.

1. Si une démo précédente a tourné, [repartir de zéro](#arrêter-et-réinitialiser).
2. Lancer la stack d’observabilité :
   ```bash
   docker compose up -d
   ```
3. Lancer l’application :
   ```bash
   docker compose -f greenhouse/docker-compose-micro.yml up -d --build
   ```
4. Vérifier que les conteneurs `greenhouse-*`, `loki`, `grafana` et `alloy` sont lancés :
   ```bash
   docker ps --format '{{.Names}}'
   ```
5. Ouvrir les onglets du navigateur dans l’ordre de la démo :
   - Application : http://localhost:5005
   - Alloy : http://localhost:12345/graph
   - Grafana Explore : http://localhost:3000
6. Créer un compte dans l’application et cliquer un peu partout pour produire des logs.
7. Dans Grafana : plage de temps *Last 15 minutes*, rafraîchissement automatique activé.
8. Garder les [requêtes LogQL](#requêtes-logql) sous la main pour les copier-coller.

Checklist :

- [ ] Environnement réinitialisé
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
| 6. Des logs aux métriques | 2 min | [Requête 4 en graphe](#des-logs-aux-métriques-étape-6) | Un pic d’erreurs visible, base d’une alerte (Ruler) |

### Le graphe Alloy (étape 3)

1. `discovery.docker` trouve les conteneurs via le socket Docker.
2. `discovery.relabel` transforme les métadonnées Docker en labels (dont `container`).
3. `loki.source.docker` lit les logs de chaque conteneur.
4. `loki.process` traite les lignes (parsing, filtres, labels).
5. `loki.write` envoie le résultat à Loki.

> Les flèches vers `discovery.*` indiquent une dépendance, pas le trajet des logs.

> Si quelqu’un demande pourquoi chaque conteneur apparaît en double dans `discovery.docker` : Docker publie chaque port en IPv4 (`0.0.0.0`) et en IPv6 (`::`), ce qui crée deux cibles. `loki.source.docker` déduplique par ID de conteneur, donc les logs ne sont lus qu’une fois.

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

### Des logs aux métriques (étape 6)

Message clé : on peut **calculer des métriques à partir des logs**, sans instrumenter l’application.

La requête 4 se lit de l’intérieur vers l’extérieur :

1. `{container=~"greenhouse-.*"}` sélectionne les flux de tous les conteneurs de l’application.
2. `|= "ERROR"` ne garde que les lignes contenant « ERROR ».
3. `count_over_time(... [1m])` compte ces lignes sur une fenêtre glissante d’une minute : le résultat devient un **nombre** à chaque instant.
4. `sum by (container)` regroupe par conteneur : une courbe par service.

Les requêtes 1 à 3 renvoyaient des **lignes de logs** ; la requête 4 renvoie une **série temporelle**, comme une métrique Prometheus.

Dans Grafana :

1. Coller la requête dans Explore et la lancer : Grafana affiche automatiquement un graphe.
2. Choisir une plage de temps qui inclut l’activation du mode erreur (*Last 15 minutes*).
3. Montrer les courbes plates avant l’incident, puis le pic. Le survol indique le conteneur et le nombre d’erreurs.

Variantes si le temps le permet : `rate(...)` au lieu de `count_over_time(...)` pour des erreurs par seconde, ou retirer `sum by (container)` pour le détail brut.

**Le lien avec les alertes.** Si on peut tracer la courbe, on peut alerter quand elle dépasse un seuil :

```logql
sum by (container) (count_over_time({container=~"greenhouse-.*"} |= "ERROR" [1m])) > 10
```

Le **Ruler** de Loki évalue ce type d’expression en continu. Exemple de règle à montrer sur une slide :

```yaml
groups:
  - name: greenhouse
    rules:
      - alert: TropDErreurs
        expr: sum by (container) (count_over_time({container=~"greenhouse-.*"} |= "ERROR" [1m])) > 10
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Plus de 10 erreurs/min sur {{ $labels.container }}"
```

`for: 2m` évite les fausses alertes : la condition doit rester vraie 2 minutes avant le déclenchement.

> Le Ruler n’est pas configuré dans ce dépôt. Pour une alerte réelle, le plus simple est **Grafana Alerting** (*Alerting → Alert rules → New alert rule*) avec la même requête.

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

## Arrêter et réinitialiser

### Après la démo : tout arrêter

Arrêter l’application d’abord, puis la stack d’observabilité (le réseau `loki-fundamentals_loki` est créé par la stack et utilisé par l’application) :

```bash
docker compose -f greenhouse/docker-compose-micro.yml down
docker compose down
```

Les conteneurs sont supprimés, mais les données (logs dans Loki, comptes de l’application) sont conservées si elles sont dans des volumes.

### Avant de refaire la démo : repartir de zéro

Pour un graphe propre, sans les erreurs de la démo précédente, supprimer aussi les volumes avec `-v` :

```bash
docker compose -f greenhouse/docker-compose-micro.yml down -v
docker compose down -v
```

Puis relancer comme dans la [préparation](#préparation-30-minutes-avant).

> `-v` efface définitivement les logs stockés et la base de l’application. Si les données sont dans un dossier monté (bind mount) plutôt qu’un volume, `-v` ne les supprime pas : vérifier la section `volumes:` des fichiers compose.

### Réinitialisation rapide, sans tout arrêter

Si on manque de temps entre deux sessions :

1. Remettre **Toggle Error Mode** sur *Off*.
2. Relancer un service arrêté : `docker start greenhouse-plant_service-1`.
3. Attendre quelques minutes, ou réduire la plage de temps de Grafana (*Last 5 minutes*) pour que le graphe reparte « propre ».

Les logs de la session précédente restent dans Loki, mais ils sortent de la plage affichée.

## Plan B et dépannage

| Problème | Solution |
|----------|----------|
| Aucun log dans Explore | Élargir la plage de temps (*Last 1 hour*), vérifier que les composants Alloy sont *Healthy* |
| Le label `container` n’apparaît pas | Vérifier `discovery.relabel` dans le graphe Alloy |
| Drilldown → Logs vide | Ne pas l’utiliser : Explore et le label browser suffisent |
| Pas d’erreurs dans les logs | Interagir davantage avec l’application, ou filtrer sur un autre mot (`\|= "error"`, `\|= "WARN"`) |
| L’application ne démarre pas (réseau introuvable) | Lancer la stack d’observabilité **avant** l’application |
| Docker ou réseau en panne | Basculer sur les captures d’écran de secours |
