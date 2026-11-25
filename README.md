# TP Observabilité — Prometheus et Grafana

Karen MOUANOU BANZOUZI

## Fichiers

- `docker-compose.yml` : déploiement de Prometheus, Grafana et Node Exporter
- `prometheus/prometheus.yml` : configuration de la collecte des métriques

## Dashboard réalisé

Dans Grafana, j'ai créé le dashboard « Supervision Linux » avec 5 panneaux :

- CPU Usage (Time series, en %)
- RAM Usage (Time series, en %)
- Disk Usage (Gauge, en %)
- Network Traffic (Time series, en bytes/sec)
- Services Availability (Stat, requête `up`)

## Exercice 5 — Diagnostic

J'ai arrêté Node Exporter avec `docker stop node-exporter`.

1. Dans Prometheus, la target `node-exporter` est passée à l'état DOWN avec l'erreur « lookup node-exporter ... no such host » : le conteneur étant arrêté, son nom n'existe plus sur le réseau Docker.
2. La métrique `up` valait 0 pour `node-exporter` et restait à 1 pour `prometheus`.
3. Dans Grafana, le panneau Services Availability affichait 0 pour `node-exporter`, et les courbes CPU, RAM, disque et réseau se sont arrêtées.

Après `docker start node-exporter`, la target est revenue à UP et `up` est revenu à 1.

## Questions de compréhension

**1. Quelle est la différence entre Prometheus et Grafana ?**

Prometheus collecte et stocke les métriques. Grafana ne stocke rien : il interroge Prometheus et affiche les données sous forme de tableaux de bord.

**2. Quel est le rôle de Node Exporter ?**

Il expose les métriques système de la machine Linux (CPU, mémoire, disque, réseau) sur le port 9100, pour que Prometheus puisse les récupérer.

**3. Que signifie le terme scraping ?**

C'est le fait que Prometheus va lui-même chercher les métriques des services, en interrogeant leur adresse `/metrics` en HTTP à intervalle régulier (ici toutes les 15 secondes).

**4. À quoi sert le fichier `prometheus.yml` ?**

Il configure Prometheus : la fréquence de collecte (`scrape_interval`) et la liste des services à superviser (`job_name` et `targets`).

**5. Pourquoi Grafana utilise-t-il l'adresse `prometheus:9090` ?**

Grafana tourne dans un conteneur. Pour lui, `localhost` désigne son propre conteneur, pas Prometheus. Sur le réseau Docker `monitoring`, chaque conteneur est joignable par son nom, donc Grafana utilise `prometheus:9090`.

**6. À quoi sert PromQL ?**

C'est le langage de requête de Prometheus. Il permet de lire les métriques et de faire des calculs dessus, par exemple le pourcentage d'utilisation du CPU ou de la RAM.

**7. Pourquoi utilise-t-on des volumes Docker ?**

Pour garder les données quand les conteneurs sont supprimés ou recréés : les métriques stockées par Prometheus et la configuration de Grafana (source de données, dashboards).

**8. Que se passe-t-il lorsqu'une target passe à l'état DOWN ?**

Prometheus n'arrive plus à récupérer ses métriques. La métrique `up` passe à 0 et il n'y a plus de nouvelles données pour ce service. Je l'ai vu à l'exercice 5 : les courbes de Grafana se sont arrêtées.

**9. Pourquoi est-il important de surveiller les ressources CPU, RAM et disque ?**

Pour voir un problème avant qu'il ne bloque le serveur : un CPU saturé ralentit les applications, une RAM pleine peut faire tuer des processus et un disque plein empêche d'écrire. Sur ma machine, le disque est déjà à 92 %.

**10. Expliquer la différence entre les états UP et DOWN.**

UP (`up` = 1) : Prometheus arrive à récupérer les métriques du service. DOWN (`up` = 0) : il n'y arrive pas, à cause d'un service arrêté ou d'un problème de réseau ou de configuration.

**11. Pourquoi utilise-t-on la fonction `rate()` pour calculer l'utilisation CPU ?**

`node_cpu_seconds_total` est un compteur : il ne fait qu'augmenter depuis le démarrage de la machine. Sa valeur brute ne dit rien sur l'activité actuelle. `rate()` calcule la vitesse d'augmentation sur une période (ici 5 minutes), ce qui donne la part de temps passée dans chaque mode, et donc le pourcentage d'utilisation.

**12. Pourquoi exclut-on généralement l'interface `lo` ?**

`lo` est l'interface de boucle locale : le trafic de la machine vers elle-même. Ce n'est pas du vrai trafic réseau, et il fausserait les mesures d'entrée et de sortie.