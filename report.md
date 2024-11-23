# Étude des mécanismes de distribution dans PostgreSQL

**Auteurs :** Samuel Roland, Timothée Van Hove

## 1. Introduction
Les bases de données relationnelles comme PostgreSQL ont longtemps dominé le paysage des SGBD, en fournissant des transactions ACID et des requêtes SQL complexes. Pourtant, à mesure que les applications modernes sont devenues plus globales et intensives en données, les limites d’un système sur une seule machine sont devenues évidentes. Que se passe-t-il lorsque cette machine atteint sa capacité maximale ? Que faire si elle tombe en panne ? Et comment gérer des millions d’utilisateurs répartis à travers le monde ?

Ces défis ont ouvert la voie à une explosion des BD NoSQL, conçues dès le départ pour être distribués, sacrifiant souvent la complexité relationnelle pour fournir une disponibilité et scalabilité accrue. Cependant, tout n’est pas une question de « NoSQL contre SQL ». Les bases de données relationnelles, comme PostgreSQL, ont relevé le défi en évoluant pour intégrer des mécanismes distribués tout en préservant leurs forces historiques.

PostgreSQL s’est adapté à ces nouvelles exigences grâce à un ensemble d’extensions et de fonctionnalités. Avec des outils comme **Citus**, qui offre un sharding transparent, ou **BDR**, qui permet une réplication multi-leader, PostgreSQL peut rivaliser avec certaines des bases de données distribuées les plus performantes. Ces solutions ne sont pas magiques et impliquent des compromis, mais elles montrent à quel point PostgreSQL reste pertinent dans ce nouvel écosystème.

## 2. Vue d’ensemble des mécanismes de distribution dans PostgreSQL

PostgreSQL est souvent reconnu pour sa stabilité et sa richesse fonctionnelle en tant que système relationnel. Cependant, à l’ère des applications massivement distribuées, il est devenu essentiel de dépasser les limites d’un seul serveur. PostgreSQL s’adapte à cette réalité grâce à une combinaison de mécanismes intégrés et d’extensions pour la réplication, le sharding, et le partitionnement. Ces outils répondent aux besoins de haute disponibilité, de tolérance aux pannes, et de scalabilité tout en maintenant les garanties transactionnelles typiques des bases relationnelles.

Ce chapitre explore comment PostgreSQL met en œuvre la distribution des données, en commençant par la réplication. Nous examinerons les défis liés à la latence, les compromis entre cohérence forte et éventuelle, ainsi que les mécanismes qui permettent de tirer parti des répliques pour améliorer les performances.

### 2.1. Réplication

La réplication est au cœur de toute architecture distribuée. Dans PostgreSQL, elle repose sur la propagation des modifications enregistrées dans les Write-Ahead Logs ([WAL](https://www.postgresql.org/docs/current/wal-intro.html)) qui permet à PostgreSQL de synchroniser les répliques avec le leader de manière fiable, bien que les modes de réplication choisis influent sur les performances et la cohérence.

PostgreSQL offre trois approches principales : la réplication synchrone, asynchrone et logique. Ces mécanismes ont chacun leurs avantages et limites, notamment en termes de latence, de tolérance aux pannes et de cohérence des données.

#### La réplication synchrone

Dans la [réplication synchrone](https://www.postgresql.org/docs/17/warm-standby.html#SYNCHRONOUS-REPLICATION), PostgreSQL garantit que toutes les répliques désignées reçoivent et confirment les modifications avant que la transaction ne soit validée. Ce processus repose sur un échange constant de messages entre le leader et les répliques, où chaque réplique doit envoyer un accusé de réception pour signaler qu’elle a bien appliqué les modifications. Ce niveau de coordination renforce la cohérence : une fois qu’une transaction est validée, toutes les répliques synchrones reflètent immédiatement son état.

Cependant, cette approche a un coût. Si une réplique est défaillante ou géographiquement distante, la latence introduite par le réseau peut ralentir considérablement les transactions. Imaginez un leader situé en Europe qui envoie des journaux à une réplique synchronisée en Asie. Même avec des connexions rapides, la latence réseau peut ajouter des dizaines de millisecondes à chaque transaction. Cette attente, bien que supportable pour des systèmes critiques comme ceux des banques, devient problématique dans des applications à haut débit ayant besoin d'une faible latence.

#### La réplication asynchrone

La réplication asynchrone privilégie la performance. Contrairement au mode synchrone, le leader n’attend pas que les répliques confirment qu’elles ont reçu les modifications avant de valider une transaction. ça réduit considérablement la latence, surtout dans des environnements où les répliques sont réparties sur de grandes distances. Par exemple, un utilisateur en Europe peut valider une transaction presque instantanément, même si les répliques en Asie ou en Amérique n’ont pas encore été mises à jour.

Par contre, cette vitesse s’accompagne d’un compromis : le replication lag. Ce décalage peut provoquer des incohérences temporaires, où une réplique asynchrone reflète un état obsolète de la BD. ça pose des problème dans des scénarios où un utilisateur souhaite lire immédiatement une donnée qu’il vient de mettre à jour, mais où la lecture est dirigée vers une réplique encore en retard (reading your own writes).

#### La réplication logique

Pour des cas d’usage plus spécifiques, PostgreSQL propose la réplication logique. Contrairement à la réplication physique, qui copie les blocs de données, la réplication logique se concentre sur les modifications au niveau des lignes et des transactions. ça permet de répliquer uniquement certaines tables ou même un sous-ensemble des données, ce qui est utile pour des migrations de bases de données ou l’intégration avec des systèmes tiers.

Par exemple, une entreprise qui migre progressivement ses données vers une nouvelle version de PostgreSQL peut utiliser la réplication logique pour synchroniser les tables critiques tout en testant la nouvelle configuration. Dans des environnements multi-tenants, où chaque client dispose de sa propre base de données, la réplication logique permet de synchroniser uniquement les données d’un client spécifique, ce qui réduit la surcharge inutile.

#### Les défis de la réplication dans un système distribué

La latence réseau est l’un des principaux facteurs limitants. PostgreSQL utilise un protocole interactif, où chaque commande doit attendre une réponse avant de passer à la suivante. ça signifie que les transactions avec des répliques distantes, surtout en mode synchrone, peuvent être ralenties par les allers-retours réseau. Même en mode asynchrone, un décalage entre le leader et ses répliques peut entraîner des incohérences qui affectent l’expérience utilisateur.

Un autre défi est l’utilisation efficace des répliques pour la scalabilité en lecture. Les répliques peuvent gérer une partie des charges de lecture pour alléger le leader, mais leur décalage peut poser problème pour certaines applications. Par exemple, un shop en ligne pourrait afficher un panier vide à un utilisateur si la réplique interrogée n’a pas encore reçu la mise à jour récente. Ce problème peut être atténué en dirigeant certaines lectures critiques vers le leader ou en utilisant des sticky sessions.

#### Configuration et mise en œuvre de la réplication

Configurer la réplication dans PostgreSQL nécessite de définir les rôles des serveurs (leader ou réplique) et d’ajuster certains paramètres clés.

**1. Préparation du leader :**

[Le serveur principal](https://www.postgresql.org/docs/current/runtime-config-replication.html#RUNTIME-CONFIG-REPLICATION-PRIMARY) (leader) doit être configuré pour activer la réplication. Cela inclut les étapes suivantes :

- **Activer le niveau de WAL approprié :** Modifier le paramètre `wal_level` dans `postgresql.conf` et le définir sur `replica` ou `logical` selon le type de réplication souhaité.
- **Configurer le nombre de connexions de répliques :** Le paramètre `max_wal_senders` détermine le nombre maximum de connexions simultanées pour les processus envoyant les journaux WAL. Par défaut, il est fixé à 10.
- **Activer les slots de réplication :** Pour éviter que des journaux nécessaires aux répliques ne soient supprimés, configurer `max_replication_slots` avec un nombre suffisant pour les répliques prévues.

**2. Configuration des répliques :**

Les [serveurs standby,](https://www.postgresql.org/docs/current/runtime-config-replication.html#RUNTIME-CONFIG-REPLICATION-STANDBY) qu’ils soient pour une réplication synchrone ou asynchrone, nécessitent :

- **Définir `primary_conninfo` :** Fournir les informations de connexion au leader, y compris l’adresse IP ou le nom d’hôte, le port, et les informations d’authentification.
- **Utiliser les slots de réplication :** Associer une réplique à un slot pour garantir la continuité de la réplication, même en cas de déconnexion temporaire.

**3. Paramètres pour la réplication synchrone :**

Pour activer la réplication synchrone, configurez :

- `synchronous_standby_names` sur le leader pour spécifier les répliques synchrones. Vous pouvez utiliser des mots-clés comme `FIRST` ou `ANY` pour prioriser ou définir un quorum de répliques.

**4. Détection des pannes et délai de reprise :**

Les paramètres tels que `wal_sender_timeout` et `wal_receiver_timeout` permettent de gérer les déconnexions inattendues entre le leader et les répliques. Par exemple, en cas de panne réseau, ces paramètres déterminent combien de temps attendre avant de marquer une connexion comme perdue.

**5. Surveillance de la réplication :**

Des vues comme `pg_stat_replication` permettent de surveiller en temps réel l’état des connexions de réplication, y compris la position du WAL appliquée sur chaque réplique. Ces informations sont cruciales pour diagnostiquer des problèmes de décalage ou de performances.

### 2.2. Partitionnement et sharding

Ces deux techniques sont utilisées pour diviser les données en ensembles plus petits et mieux gérables. Elles permettent d'améliorer les performances et permettre une meilleure scalabilité. Elles s'appliquent à des contextes différents mais ne s'excluent pas mutuellement. PGSQL offre un support natif pour le partitionnement, alors que le sharding distribué repose sur des extensions comme **Citus**.

#### Partitionnement natif dans PostgreSQL 

Depuis la version 10, PGSQL propose un [partitionnement](https://www.postgresql.org/docs/17/ddl-partitioning.html) natif qui divise une table en sous-tables ou "partitions" selon des critères déclarés au moment de la création de la table. Cela permet d’organiser les données et d'améliorer les performances des requêtes en travaillant sur des ensembles de données plus petits.

Par exemple, pour une table contenant des données de séries temporelles comme des logs , le partitionnement par plages (range partitioning) permet de diviser les données par mois ou année. Quand une requête cible une période donnée, PGSQL ignore automatiquement les partitions non pertinentes grâce à son mécanisme d’exclusion de partitions. ça réduit le temps de traitement et améliore la performance.

PostgreSQL supporte trois types principaux de partitionnement:

- **Partitionnement par plages (range partitioning)** : Idéal pour les données temporelles ou séquentielles. Exemple : partitionner une table de transactions par mois.
- **Partitionnement par liste (list partitioning)** : Utile pour organiser les données en catégories distinctes comme des régions ou des types d’événements.
- **Partitionnement par hachage (hash partitioning)** : Répartit les données uniformément pour éviter les déséquilibres dans les volumes des partitions.

Les avantages du partitionnement sont multiples :

1. **Amélioration des performances des requêtes** : Les partitions pertinentes sont ciblées, limitant les recherches inutiles.
2. **Suppression efficace des anciennes données** : Les partitions obsolètes peuvent être supprimées rapidement avec `DROP TABLE`, évitant les problèmes de fragmentation.
3. **Gestion optimisée de l’autovacuum** : Chaque partition peut être analysée indépendamment, ce qui réduit les blocages sur des tables volumineuses.

Cependant, le partitionnement natif reste limité à un nœud unique. Si les données ou la charge augmentent au-delà des capacités d’un serveur, cette approche ne suffit plus.

#### Sharding distribué avec [Citus](https://docs.citusdata.com/en/stable/admin_guide/cluster_management.html#postgresql-extensions) : Scalabilité horizontale pour PostgreSQL

Lorsque les limites d’un seul serveur sont atteintes, le [sharding](https://docs.citusdata.com/en/stable/get_started/concepts.html#sharding-models) devient une solution incontournable. Contrairement au partitionnement, qui divise les données sur un nœud unique, le sharding distribue les données entre plusieurs nœuds d’un cluster. Cela permet une scalabilité horizontale où chaque nœud gère un sous-ensemble des données, appelé "[shard](https://docs.citusdata.com/en/stable/get_started/concepts.html#shards)". PGSQL n’implémente pas nativement le sharding, mais des extensions comme Citus transforment PGSQL en une base de données massivement distribuée.

**Comment fonctionne Citus ?**
Avec Citus, les données sont shardées selon une clé de distribution, souvent une colonne comme `user_id` dans une application [multi-tenant](https://docs.citusdata.com/en/stable/sharding/data_modeling.html#multi-tenant-apps). Chaque shard est stocké sur un nœud différent, et un nœud coordinateur gère les métadonnées et distribue les requêtes. Par exemple :

- Si une requête concerne un seul utilisateur (par exemple `user_id=123`), elle est routée vers un seul nœud, minimisant les échanges réseau.
- Pour des requêtes plus complexes nécessitant plusieurs shards, Citus parallélise les opérations entre les nœuds, permettant un traitement rapide même sur de gros volumes de données.

Un avantage clé de Citus est que chaque shard est une table PGSQL normale. Cela veut dire qu'on peut utiliser des index locaux, des contraintes, et d’autres optimisations traditionnelles. De plus, les groupes de shards (shard groups) permettent de regrouper les données fréquemment utilisées ensemble (comme des tables avec des relations fortes), ce qui réduit les opérations entre les nœuds.

#### Partitionnement et sharding : Une combinaison puissante

Partitionnement et sharding ne s’excluent pas, mais peuvent être combinés. Par exemple, dans une application de séries temporelles :

- Les données peuvent être partitionnées par plages temporelles (par mois ou trimestre) pour simplifier la gestion des anciennes données et améliorer les performances des requêtes.
- Chaque partition peut ensuite être shardée et distribuée sur plusieurs nœuds grâce à Citus, permettant une scalabilité horizontale pour des charges de travail massives.

#### Considérations

Ces techniques nécessitent des choix, notamment pour la clé de distribution dans le sharding. Une mauvaise clé peut entraîner des bottlenecks, car les requêtes devront souvent interroger plusieurs shards. 

En plus de ça, le modèle de données doit être fait en prenant en compte des limitations :

- Les relations entre les données doivent idéalement inclure la clé de distribution pour éviter les jointures entre les nœuds.
- Les requêtes lourdes peuvent nécessiter des optimisations pour être parallélisées efficacement.

## 3. Gestion des pannes et rééquilibrage

Les systèmes distribués sont puissants, mais comme tout ce qui est complexe, ils ne sont pas sans leurs défis. Pannes, déséquilibres, nœuds surchargés : PGSQL, avec ses extensions comme Citus ou [Patroni](https://patroni.readthedocs.io/en/latest/), a des solutions solides pour garder tout ça sous contrôle.

### 3.1. Détection et récupération des pannes

Quand un nœud tombe, la première étape est de s’en rendre compte. PostgreSQL ne propose pas directement de mécanisme pour détecter une panne, mais les outils comme Patroni ou [Pgpool-II](https://www.pgpool.net/docs/latest/en/html/intro-whatis.html) utilisent des heartbeats pour vérifier que tout le monde est en ligne. Si un nœud reste silencieux trop longtemps, il est marqué comme défaillant, et le processus de récupération commence.

#### Répliques en panne

Quand une [réplique plante](https://docs.citusdata.com/en/stable/admin_guide/cluster_management.html#worker-node-failures), PGSQL peut la remettre sur pied grâce à ses journaux [WAL](https://www.postgresql.org/docs/current/wal-intro.html) (*Write-Ahead Logs*). Ces journaux enregistrent toutes les modifications effectuées sur le leader. Lorsqu’une réplique revient à la vie, elle consulte son journal local pour voir où elle s’est arrêtée et demande au leader de lui envoyer les transactions manquantes (catch-up recovery). Ce processus est rapide et garantit que tout reste en ordre.

#### Le leader en panne

[La panne d’un leader](https://docs.citusdata.com/en/stable/admin_guide/cluster_management.html#coordinator-node-failures) est une toute autre histoire. Ici, on doit déclencher un failover : promouvoir une réplique en tant que nouveau leader. Patroni surveille les nœuds via un système comme [Etcd](https://github.com/etcd-io/etcd) ou [ZooKeeper](https://zookeeper.apache.org/) et décide, en cas de besoin, qui prendra le relais. Une fois le failover terminé, il faut s’assurer que l’ancien leader ne revienne pas comme si de rien n’était : un mécanisme appelé [STONITH](https://en.wikipedia.org/wiki/STONITH) (*Shoot The Other Node In The Head*) veille à ce qu’un nœud défaillant ne cause pas de confusion.

Si notre ancien leader revient en ligne après un failover, il faut utiliser `pg_rewind` pour le resynchroniser rapidement avec le nouveau leader. C’est plus rapide que de reconstruire une réplique à partir de zéro.

### 3.2. Ajouter ou retirer une réplique

Ajouter une nouvelle réplique dans PostgreSQL, c’est un peu comme mettre quelqu’un à jour avec ce qui s’est passé pendant qu’il était absent. Tout commence par une image cohérente de la base de données, prise sur le leader. Cette image est transférée à la nouvelle réplique, qui rattrape ensuite son retard en lisant les journaux WAL. L’opération se déroule sans interruption pour les utilisateurs.

Le retrait d’une réplique est un peu plus compliqué. PostgreSQL ne redistribue pas automatiquement les données d’un nœud supprimé ; il faut des outils comme Citus pour gérer ça efficacement. Avec Citus, les partitions (ou shards) gérées par la réplique retirée sont transférées vers d’autres nœuds. Citus s’appuie sur une stratégie de shards fixes, ce qui signifie que seules les partitions nécessaires sont déplacées, limitant les perturbations.

Pour éviter les conflits pendant le retrait d’un shard, Citus introduit les shards orphelins. En gros, le shard n’est pas supprimé immédiatement après son déplacement : il est marqué pour suppression différée, le temps que toutes les requêtes en cours soient terminées. Pas de précipitation, pas de problèmes.

### 3.3. Rééquilibrage des données

Ajouter ou retirer un nœud change l’équilibre du cluster. Le [rééquilibrage](https://docs.citusdata.com/en/stable/admin_guide/cluster_management.html#rebalance-shards-without-downtime) consiste à redistribuer les données pour que tout reste fluide. Avec Citus, c’est presque magique.

Quand un nouveau nœud rejoint le cluster, Citus détermine quels shards doivent être déplacés. Le transfert est orchestré de manière transparente, et on peut suivre la progression avec la fonction `get_rebalance_progress`. Elle affiche la taille des shards sur les nœuds source et cible, et estime le pourcentage de transfert terminé. Une requête rapide nous donne une vue d’ensemble :

```sql
SELECT
    table_name,
    shardid,
    pg_size_pretty(source_shard_size) AS source_size,
    pg_size_pretty(target_shard_size) AS target_size,
    CASE WHEN shard_size = 0
        THEN 100
        ELSE LEAST(round(target_shard_size::numeric / shard_size * 100, 2), 100)
    END AS percent_completed_estimate
FROM get_rebalance_progress()
WHERE progress = 1;
```

### 3.4. Tests réguliers pour une résilience accrue

Un cluster n’est résilient que si ses mécanismes de récupération fonctionnent. Pour éviter les mauvaises surprises, il est crucial de tester régulièrement :

- Les basculements (switchover) pour s’assurer qu’un autre nœud peut facilement prendre le rôle de leader.
- Les simulations de pannes pour vérifier la réactivité des outils comme Patroni.
- Le rééquilibrage automatique pour voir si les données sont bien redistribuées en cas de changement.

Avec PGSQL et ses extensions, gérer les pannes et équilibrer les données est loin d’être insurmontable. Le système est conçu pour absorber les chocs tout en restant opérationnel, et les outils disponibles rendent le tout presque agréable à gérer.



## 4. Cohérence et modèles de consistance

Quand on parle de cohérence dans PGSQL, on entre dans un monde de choix subtils et d’équilibres délicats. La manière dont on configure notre système de réplication et nos modèles de consistance dépend fortement de nos besoins : cohérence absolue, performances maximales, ou quelque chose entre les deux.

### Cohérence forte et mise en œuvre

La cohérence forte est le saint Graal des systèmes transactionnels. Dans PostgreSQL, ça s’obtient principalement avec la réplication synchrone. A chaque fois qu’une écriture est faite sur le leader, ce dernier envoie les modifications aux répliques synchrones. Ces répliques doivent confirmer qu’elles ont bien appliqué les changements avant que le leader ne valide la transaction. ça signifie que, lorsque le client reçoit une confirmation, toutes les copies des données sont alignées.

ça fonctionne grâce au **Write-Ahead Logging (WAL)**, qui journalise chaque modification. Cependant, si une réplique tombe en panne ou devient inaccessible, le leader peut bloquer les nouvelles transactions pour maintenir cette cohérence stricte. ça peut être un frein pour la disponibilité, mais pour des cas critiques comme les banques ou les applications médicales, c’est un prix à payer acceptable.

On configure ça via des paramètres comme `synchronous_commit` et `synchronous_standby_names` dans le fichier `postgresql.conf`. ça nous permet même de choisir les répliques qui doivent être synchrones, offrant une certaine flexibilité.

### Cohérence éventuelle et tolérance au retard

La cohérence éventuelle est réalisée grâce à la réplication asynchrone. Ici, le leader n’attend pas que les répliques confirment qu’elles ont appliqué les modifications avant de valider la transaction. Les journaux WAL sont transmis aux répliques selon leur disponibilité, ce qui introduit un léger retard. Pendant ce délai, une requête de lecture sur une réplique pourrait retourner une version obsolète des données.

PGSQL configure cette approche en assignant des répliques comme asynchrones par défaut. Ces répliques utilisent des processus appelés **WAL receivers**, qui se connectent au leader pour récupérer les journaux WAL. Ces journaux sont stockés et appliqués localement par les répliques pour maintenir une version cohérente des données à terme.

L’asynchronisme est souvent combiné avec des mécanismes comme le Streaming Replication, où les répliques reçoivent un flux continu des journaux WAL. Bien que ça améliore les délais de propagation, il ne garantit pas une cohérence immédiate. PGSQL intègre également des outils comme `pg_stat_replication` pour surveiller le décalage entre le leader et ses répliques, permettant d’identifier et de corriger rapidement les trop grands retards.

### Résolution des incohérences : cohérence lecture-écriture et lectures monotones

Pour répondre aux limitations de la cohérence éventuelle, PostgreSQL offre des mécanismes supplémentaires. Par exemple, pour garantir une cohérence lecture-écriture, les clients peuvent explicitement choisir de lire les données depuis le leader, où toutes les modifications sont immédiatement visibles. Cette approche est utile dans des cas comme les tableaux de bord utilisateur, où un utilisateur doit voir les données qu’il vient de soumettre.

PostgreSQL permet d’implémenter cela via les connexions explicites au leader ou en utilisant des outils de routage comme **Pgpool-II**, qui peut diriger dynamiquement les requêtes de lecture vers le leader ou les répliques en fonction du contexte. Une autre méthode consiste à configurer des *sticky sessions*, où un client est toujours dirigé vers une réplique spécifique, réduisant les risques d’incohérences dues à des retards variables entre les répliques.

Les lectures monotones, quant à elles, garantissent qu’un utilisateur ne verra jamais des données plus anciennes que celles déjà observées, même en cas de retard de réplication. PostgreSQL ne gère pas cela directement, mais des outils comme Citus peuvent aider en assurant que chaque session d’utilisateur est toujours routée vers le même nœud pour toutes ses requêtes.

### Compromis dans PostgreSQL

PGSQL permet de jouer sur plusieurs paramètres pour ajuster la cohérence à nos besoins. On peut opter pour une **consistance** lecture-écriture stricte en utilisant des transactions serialisables ou en verrouillant explicitement certaines données via `SELECT FOR UPDATE`. Mais attention : ces mécanismes augmentent la charge et réduisent la concurrence.

Pour les applications analytiques ou massivement parallèles, une approche moins stricte, combinée avec des vérifications explicites au niveau applicatif, peut suffire. Le tout est de trouver le bon équilibre entre cohérence, performance, et disponibilité.

### Citus et les modèles de réplication

Citus  permet de choisir entre deux modèles de réplication :

1. **Rélication par déclarations (statement-based)** : Cette méthode est simple et efficace pour des cas où les données sont append-only, comme des journaux ou des séries temporelles.
2. **Rélication en streaming** : Elle utilise les fonctionnalités natives de PostgreSQL pour garantir que toutes les répliques d’un shard sont parfaitement alignées. C’est particulièrement utile pour des applications [multi-tenants](https://docs.citusdata.com/en/stable/sharding/data_modeling.html#multi-tenant-apps) où les contraintes transactionnelles sont importantes.

Citus aide aussi à gérer des cas complexes, comme la colocation des données pour améliorer les performances des jointures dans un système distribué.

### Quand la cohérence rencontre la concurrence : MVCC et isolation

Le modèle [MVCC](https://wiki.postgresql.org/wiki/MVCC) de PGSQL garantit que les lectures et les écritures ne se bloquent pas mutuellement, même sous des niveaux d’isolation élevés comme `Serializable`. Ce modèle offre une cohérence instantanée à chaque transaction, tout en minimisant les verrous. Cependant, pour des besoins très stricts, on peut utiliser des transactions [serialisables](https://www.postgresql.org/docs/current/transaction-iso.html#XACT-SERIALIZABLE) ou des [verrous explicites](https://www.postgresql.org/docs/current/explicit-locking.html) pour éviter les anomalies.

## 7. Conclusion et perspectives
- Résumé des forces et limites de PostgreSQL dans la distribution.
- Futur des bases de données distribuées avec PostgreSQL.



## Références

1. Shard Rebalancing in Citus 10.1. [Citus Data Blog](https://www.citusdata.com/blog/2021/09/03/shard-rebalancing-in-citus-10-1)
2. Citus' Replication Model: Today and Tomorrow [Citus Data Blog](https://www.citusdata.com/blog/2016/12/15/citus-replication-model-today-and-tomorrow/)
3. Cluster Management [Citus documentation](https://docs.citusdata.com/en/stable/admin_guide/cluster_management.html)
4. Failover. [PG documentation](https://www.postgresql.org/docs/17/warm-standby-failover.html)
5. Replication. [PG documentation](https://www.postgresql.org/docs/current/runtime-config-replication.html)
6. Concurrency Control [PG documentation](https://www.postgresql.org/docs/current/mvcc.html)
7. Understanding partitioning and sharding in Postgres and Citus. [Azure Database for PostgreSQL Blog](https://techcommunity.microsoft.com/blog/adforpostgresql/understanding-partitioning-and-sharding-in-postgres-and-citus/3891629)
8. An Overview of Distributed  PostgreSQL Architectures. [Crunchydata Blog](https://www.crunchydata.com/blog/an-overview-of-distributed-postgresql-architectures)
9. How PostreSQL replication works. [Medium Blog](https://medium.com/moveax/how-postgresql-replication-works-6288b3e6000e)
