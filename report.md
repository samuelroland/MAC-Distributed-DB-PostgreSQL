# Étude des mécanismes de distribution dans PostgreSQL

**Auteurs :** Samuel Roland, Timothée Van Hove

## 1. Introduction
Les bases de données distribuées sont devenues un sujet incontournable aujourd’hui, surtout avec la montée en puissance des applications qui doivent gérer des volumes énormes de données tout en restant rapides et fiables. Pour répondre à ces besoins, ces systèmes répartissent les données sur plusieurs serveurs, garantissant ainsi une meilleure performance, une meilleure tolérance aux pannes, et une disponibilité continue. Dès qu’une application doit évoluer à grande échelle, il lui faut une solution de ce genre.

Au cours des dernières années, les bases de données NoSQL ont explosé en popularité, en grande partie parce qu’elles répondent naturellement à ces problématiques de distribution. Des systèmes comme MongoDB, Cassandra ou DynamoDB ont été conçus pour gérer des environnements massivement distribués, souvent au détriment des fonctionnalités relationnelles traditionnelles. Cette popularité des NoSQL a poussé les systèmes relationnels, comme PostgreSQL ou MySQL, à évoluer pour ne pas être laissés sur la touche. Pour cela, de nouvelles fonctionnalités et extensions ont vu le jour, permettant à ces bases de données relationnelles de rester compétitives tout en intégrant des mécanismes de réplication, de sharding et de cohérence.

PostgreSQL, en particulier, est un excellent exemple de cette évolution. Initialement, c’est un système relationnel classique et conforme aux standards SQL. Mais grâce à un écosystème dynamique d’extensions et de forks comme Citus, Pgpool-II ou BDR, PostgreSQL peut aujourd’hui être utilisé comme une base distribuée capable de rivaliser avec certaines solutions NoSQL, tout en conservant ses points forts comme la gestion fine des transactions.

Dans ce rapport, nous allons explorer comment PostgreSQL a intégré ces mécanismes de distribution. Nous parleront des techniques de réplication et de sharding, mais aussi des outils qui permettent de gérer les pannes ou de rééquilibrer les données. On verra aussi en quoi PostgreSQL se positionne différemment des bases NoSQL et quels compromis il offre dans un environnement distribué.

## 2. Vue d’ensemble des mécanismes de distribution dans PostgreSQL
### 2.1. Réplication
PostgreSQL propose plusieurs mécanismes de réplication permettant de répondre à des besoins variés en matière de scalabilité, de disponibilité et de tolérance aux pannes. Ces mécanismes se distinguent principalement par leurs modèles de synchronisation et leur granularité.

**Réplication synchrone :**

Dans ce mode, les données sont écrites sur un ou plusieurs nœuds répliqués (followers) avant que la transaction ne soit confirmée au client.

- Avantages : Garantit que toutes les répliques impliquées sont cohérentes et à jour à tout moment.
- Inconvénients : La latence des transactions peut augmenter, surtout si les répliques sont géographiquement éloignées.
- Cas d’usage : Applications critiques nécessitant une cohérence forte, telles que les systèmes financiers.

**Réplication asynchrone :**

Les transactions sont confirmées au client dès que les modifications sont écrites sur le leader. Les followers reçoivent ensuite les mises à jour avec un léger délai.

- Avantages : Amélioration de la latence des transactions côté client.
- Inconvénients : Peut entraîner des lectures obsolètes sur les followers en cas de retard de réplication.
- Cas d’usage : Applications tolérant des incohérences temporaires, comme l’analyse de données.

**Réplication logique :**

Permet de répliquer uniquement certaines parties de la base de données ou de transformer les données pendant leur réplication. Les événements sont transmis au niveau des lignes, ce qui est plus flexible que la réplication physique.

- Avantages : Supporte des cas avancés tels que la migration inter-version et l'intégration de données entre systèmes hétérogènes.
- Inconvénients : Configuration plus complexe et performance légèrement moindre comparée à la réplication physique.
- Cas d’usage : Intégration avec des systèmes tiers, migration progressive entre bases de données.

**Cas d’usage typiques de la réplication dans PostgreSQL :**

- **Scalabilité en lecture :** Les répliques asynchrones sont souvent utilisées pour répartir les charges de lecture entre plusieurs nœuds.
- **Haute disponibilité :** La réplication synchrone permet de basculer rapidement sur un follower en cas de panne du leader.
- **Migration et intégration :** La réplication logique facilite le transfert des données vers de nouveaux systèmes sans interruption de service.

### 2.2. Partitionnement et sharding
Avec l'augmentation des volumes de données et des exigences en matière de performance, PostgreSQL fournit des outils pour diviser et distribuer les données, réduisant ainsi les contraintes sur un seul nœud.

**Différence entre partitionnement logique et sharding :**

Le partitionnement logique divise une table unique en sous-tables basées sur des critères définis (par exemple, plages de dates ou plages de valeurs). La gestion est transparente pour l'utilisateur dans PostgreSQL. Les partitions restent sur le même serveur ou cluster.

- Cas d’usage : Optimisation des requêtes sur de très grandes tables grâce à l'exclusion des partitions non pertinentes.

Le Sharding étend le concept de partitionnement en distribuant les partitions sur plusieurs nœuds, chaque nœud gérant une partie du dataset. La gestion nécessite des outils supplémentaires ou des extensions comme Citus.

- Cas d’usage : Scalabilité horizontale pour les bases de données très volumineuses.

**Partitionnement natif dans PostgreSQL :**

PostgreSQL supporte nativement le partitionnement depuis la version 10. Les méthodes de partitionnement : Par plage de valeurs (range partitioning), par liste de valeurs (list partitioning), par hachage (hash partitioning, introduit dans PostgreSQL 11).

- Avantages : les utilisateurs interagissent avec une table unique. Prise en charge d’index locaux pour optimiser les performances des partitions.
- Limites : Conçu pour fonctionner sur un seul nœud (pas de distribution native sur plusieurs nœuds).

**Sharding et extensions**

PostgreSQL ne fournit pas un support de sharding distribué natif, mais des extensions comme comblent cette lacune. Elles permettent de distribuer automatiquement les données en partitions sur plusieurs nœuds, transformant PostgreSQL en un système distribué massivement scalable.

## 3. Extensions et Forks
#### 3.1 Citus

Citus est une extension open-source pour PostgreSQL qui transforme une instance PostgreSQL en un système distribué capable de gérer des charges de données massives en répartissant les données sur plusieurs nœuds (shards).

Les données sont réparties horizontalement en "shards" sur plusieurs nœuds du cluster. Un nœud maître (coordinator) gère les métadonnées et dirige les requêtes vers les nœuds de données (workers). Les requêtes SQL sont automatiquement transformées en requêtes parallèles sur les nœuds de données.

**Sharding basé sur une clé de distribution :**Chaque table est partitionnée en shards selon une clé spécifique (souvent une clé primaire ou une colonne fréquemment utilisée dans les filtres). Les shards sont distribués entre les nœuds du cluster.

**Avantages pour le sharding horizontal :**

- Scalabilité horizontale : Ajout de nouveaux nœuds pour gérer des volumes de données croissants ou des charges plus importantes.
- Performance accrue : Les requêtes sont distribuées et exécutées en parallèle sur les nœuds.
- Simplification des opérations : Les requêtes SQL standards sont prises en charge sans que l'utilisateur ait besoin de connaître les détails de la distribution des données.
- Rééquilibrage dynamique : Redistribution des shards lors de l'ajout ou de la suppression de nœuds pour assurer une charge équilibrée.


#### 3.2 Pgpool-II

Pgpool-II est un middleware open-source conçu pour s'interfacer avec PostgreSQL. Il offre diverses fonctionnalités pour améliorer les performances et la résilience des bases de données.

- Réplication native : Permet de configurer une réplication maître-follower pour assurer la haute disponibilité des données.
- Réplication parallèle : Permet de diviser une requête en sous-requêtes pour les exécuter sur plusieurs nœuds, améliorant ainsi les performances.

Équilibrage de charge : Pgpool-II répartit les requêtes en lecture sur les nœuds followers pour améliorer la scalabilité en lecture. Il maintient un suivi des transactions pour éviter les incohérences dues au décalage de réplication asynchrone.

Autres fonctionnalités : Utilise un pooling de connexions pour réduire la surcharge de gestion des connexions sur PostgreSQL. En cas de panne d’un nœud, Pgpool-II redirige automatiquement les requêtes vers un nœud disponible.

**Limites :**

- La configuration de Pgpool-II peut être difficile, surtout dans des environnements avec de nombreux nœuds.
- Étant une couche intermédiaire, Pgpool-II peut introduire une latence supplémentaire pour certaines requêtes.
- En cas de réplication asynchrone, les requêtes réparties sur plusieurs nœuds peuvent renvoyer des données obsolètes.


#### 3.3 BDR (Bi-Directional Replication)

BDR est une extension avancée pour PostgreSQL qui implémente la réplication multi-leader. Contrairement à la réplication classique à leader unique, elle permet à plusieurs nœuds de jouer le rôle de leader et d’accepter des écritures.

**Réplication multi-leader :**Tous les nœuds dans le cluster peuvent recevoir des écritures. Les modifications effectuées sur un nœud sont propagées aux autres nœuds via la réplication logique.

**Gestion des conflits :** Les conflits d'écriture sont détectés et résolus automatiquement en fonction de politiques définies (ex. dernier écrit gagne, résolution basée sur les colonnes).

**Garanties offertes :**

- Haute disponibilité : Pas de dépendance à un nœud leader unique.
- Cohérence éventuelle : Une fois propagées, les modifications convergent vers un état cohérent.
- Résilience : Si un nœud devient inaccessible, les autres continuent de fonctionner sans interruption.

**Limites :**

- Complexité : La gestion des conflits et le maintien de la cohérence entre les nœuds augmentent la complexité.
- Latence : Les mises à jour doivent être propagées à plusieurs nœuds, ce qui peut augmenter le délai de réplication.

## 4. Gestion des pannes et rééquilibrage

### Détection et récupération des pannes

Dans PostgreSQL, la détection des pannes repose sur des mécanismes simples mais efficaces, comme les *heartbeats* : des messages échangés régulièrement entre les nœuds pour vérifier qu’ils sont en ligne. Par exemple, si une réplique ne répond pas pendant un certain temps défini (un *timeout*), elle est marquée comme "non disponible". Ces heartbeats sont souvent gérés via des extensions ou des outils externes, comme **Pgpool-II** ou **Patroni**, qui surveillent activement les connexions entre les nœuds.

Lorsqu’un follower tombe en panne, PostgreSQL utilise son système de journalisation **Write-Ahead Logging (WAL)** pour le remettre à jour. Chaque transaction effectuée sur le leader est d'abord écrite dans le WAL avant d’être appliquée. Si le follower revient en ligne après une panne, il consulte son journal local pour déterminer où il s’est arrêté (la position dans le WAL, appelée *log sequence number* ou LSN). Ensuite, il demande au leader toutes les transactions manquantes depuis ce point. Ce processus s’appelle le *catch-up recovery* et garantit que la réplique se synchronise sans perdre d’informations, tout en évitant de redémarrer tout le cluster.

La gestion de la panne d’un leader est plus complexe. Ici, un *failover* doit être initié pour promouvoir une réplique en tant que nouveau leader. Patroni, par exemple, utilise un service de coordination comme *Etcd* ou *ZooKeeper* pour surveiller l’état des nœuds et organiser un consensus sur quel nœud doit devenir le nouveau leader. Ce consensus est nécessaire pour éviter qu’un ancien leader, qui reviendrait en ligne, ne cause des conflits en agissant comme s’il était encore en charge. Une fois le failover terminé, Patroni s’assure que l’ancien leader est rétrogradé en follower, ce qui évite les désynchronisations.

### Ajout et retrait de partitions ou répliques

Lorsqu'on ajoute une réplique à PostgreSQL, le processus commence par un **snapshot cohérent** des données du leader. Ce snapshot est une image figée de la base de données à un moment précis, souvent copiée sur le disque. Ensuite, la nouvelle réplique commence à rattraper les transactions survenues après ce snapshot en lisant les journaux WAL du leader. Cela permet d’intégrer la réplique au cluster sans interrompre les opérations en cours. Si on utilise la réplication synchrone, cette réplique peut immédiatement participer au maintien de la cohérence des données.

Retirer une réplique, c’est un peu plus délicat. PostgreSQL ne redirige pas automatiquement les données gérées par la réplique retirée : cette opération est souvent manuelle ou gérée via des extensions comme **Citus**. Avec Citus, les partitions qui étaient sur le nœud retiré sont redistribuées dynamiquement sur les autres nœuds du cluster. C’est possible grâce à une stratégie de *partitions fixes*, où chaque partition est associée à une plage spécifique de données. Ainsi, seule l’affectation des partitions entre les nœuds change, tandis que la structure de la base de données reste intacte. Cela minimise les mouvements de données inutiles.

Pour éviter les pertes de données, PostgreSQL utilise des mécanismes de sauvegarde intégrée et des copies incrémentielles basées sur le WAL. Ainsi, même si une réplique est retirée brusquement, les données peuvent être reconstruites à partir des autres répliques ou directement du leader.

### Mécanismes de rééquilibrage automatique

Le rééquilibrage dans PostgreSQL, en particulier avec **Citus**, est un processus pensé pour minimiser les perturbations. Lorsqu’un nouveau nœud est ajouté, les partitions existantes sont redistribuées pour équilibrer la charge. Citus suit une approche basée sur une fonction de hachage pour assigner chaque clé de données à une partition. Ces partitions sont ensuite réparties sur les nœuds disponibles. Si un nouveau nœud rejoint le cluster, Citus transfère seulement les partitions nécessaires, au lieu de tout réorganiser. Cela évite les mouvements de données inutiles et garantit que le cluster reste opérationnel tout au long du processus.

Une autre approche consiste à éviter la méthode **hash mod N**, qui redistribuerait toutes les données en cas de changement du nombre de nœuds. Citus préfère des partitions fixes, attribuées de manière pseudo-aléatoire, ce qui rend le rééquilibrage plus simple et plus efficace.

Pour les requêtes, des outils comme **Pgpool-II** surveillent en permanence la charge de chaque nœud et redirigent dynamiquement les requêtes vers ceux qui sont moins sollicités. Par exemple, si un nœud reçoit une surcharge de lectures, Pgpool-II peut envoyer les requêtes vers d’autres répliques disponibles. Ce routage intelligent assure que le cluster reste performant, même en cas de variations importantes dans les charges de travail.

Si un nœud tombe en panne, PostgreSQL et ses extensions gèrent automatiquement la redistribution des partitions ou la reconfiguration des connexions. Les partitions du nœud défaillant sont temporairement servies par d’autres nœuds jusqu’à ce que le problème soit résolu. Patroni et Pgpool-II jouent ici un rôle clé, en combinant des mécanismes de surveillance active et de failover automatique pour garantir que le système reste disponible et performant.



## 5. Cohérence et modèles de consistance

Dans PostgreSQL, la cohérence repose sur les mécanismes de réplication et les modèles de consistance adoptés. Ces choix dépendent de la configuration du cluster et des besoins spécifiques des applications.

### Cohérence forte et mise en œuvre

La cohérence forte est principalement atteinte grâce à la **réplication synchrone**, qui garantit que toutes les répliques d’un cluster sont à jour avant que la transaction ne soit validée. Lorsqu’une écriture est effectuée sur le leader, celui-ci transmet les modifications aux répliques synchrones. Chaque réplique doit confirmer qu’elle a appliqué ces modifications avant que le leader n’informe le client que la transaction est terminée.

PostgreSQL gère cela à travers son système de **Write-Ahead Logging (WAL)**. Le WAL journalise toutes les modifications apportées à la base de données. Lors de la réplication synchrone, le leader envoie ces journaux aux répliques, qui les appliquent à leurs propres copies des données. Si une réplique ne répond pas ou est temporairement inaccessible, le leader suspend les validations de nouvelles transactions, maintenant ainsi la cohérence forte mais au prix d’une disponibilité réduite. Cette approche convient aux applications où les erreurs ou les écarts de données sont inacceptables, comme dans les systèmes bancaires ou les registres critiques.

La configuration dans PostgreSQL se fait principalement à travers le fichier `postgresql.conf`, où des paramètres comme `synchronous_commit` sont définis. Les administrateurs peuvent également spécifier quelles répliques doivent être synchrones via `synchronous_standby_names`. Cette granularité offre un contrôle précis sur les niveaux de cohérence et de performance.

### Cohérence éventuelle et tolérance au retard

La cohérence éventuelle est réalisée grâce à la **réplication asynchrone**. Ici, le leader n’attend pas que les répliques confirment qu’elles ont appliqué les modifications avant de valider la transaction. Les journaux WAL sont transmis aux répliques selon leur disponibilité, ce qui introduit un léger retard. Pendant ce délai, une requête de lecture sur une réplique pourrait retourner une version obsolète des données.

PostgreSQL configure cette approche en assignant des répliques comme asynchrones par défaut. Ces répliques utilisent des processus appelés **WAL receivers**, qui se connectent au leader pour récupérer les journaux WAL. Ces journaux sont stockés et appliqués localement par les répliques pour maintenir une version cohérente des données à terme.

L’asynchronisme est souvent combiné avec des mécanismes comme le *Streaming Replication*, où les répliques reçoivent un flux continu des journaux WAL. Bien que cela améliore les délais de propagation, il ne garantit pas une cohérence immédiate. PostgreSQL intègre également des outils comme `pg_stat_replication` pour surveiller le décalage entre le leader et ses répliques, permettant d’identifier et de corriger rapidement les retards excessifs.

### Résolution des incohérences : cohérence lecture-écriture et lectures monotones

Pour répondre aux limitations de la cohérence éventuelle, PostgreSQL offre des mécanismes supplémentaires. Par exemple, pour garantir une cohérence lecture-écriture, les clients peuvent explicitement choisir de lire les données depuis le leader, où toutes les modifications sont immédiatement visibles. Cette approche est utile dans des cas comme les tableaux de bord utilisateur, où un utilisateur doit voir les données qu’il vient de soumettre.

PostgreSQL permet d’implémenter cela via les connexions explicites au leader ou en utilisant des outils de routage comme **Pgpool-II**, qui peut diriger dynamiquement les requêtes de lecture vers le leader ou les répliques en fonction du contexte. Une autre méthode consiste à configurer des *sticky sessions*, où un client est toujours dirigé vers une réplique spécifique, réduisant les risques d’incohérences dues à des retards variables entre les répliques.

Les lectures monotones, quant à elles, garantissent qu’un utilisateur ne verra jamais des données plus anciennes que celles déjà observées, même en cas de retard de réplication. PostgreSQL ne gère pas cela directement, mais des outils comme Citus peuvent aider en assurant que chaque session d’utilisateur est toujours routée vers le même nœud pour toutes ses requêtes.

### Compromis dans PostgreSQL

Les modèles de consistance dans PostgreSQL illustrent les compromis entre cohérence, disponibilité et performance. Si un système exige une cohérence stricte, les performances peuvent en souffrir en raison de la nécessité d’attendre les confirmations des répliques synchrones. À l’inverse, une configuration axée sur la disponibilité peut entraîner des retards dans la propagation des modifications, créant des incohérences temporaires.

PostgreSQL permet de personnaliser ces compromis en jouant sur des paramètres comme le niveau de synchronisation (`synchronous_commit`) et les priorités des répliques. Ces options offrent aux administrateurs la flexibilité nécessaire pour répondre aux exigences spécifiques des applications, qu’il s’agisse de bases de données transactionnelles ou analytiques.

### Extensions pour gérer la cohérence

Les extensions comme **Citus** apportent une couche supplémentaire de contrôle sur les modèles de cohérence dans un environnement massivement distribué. Elles permettent de définir des niveaux de réplication et de cohérence pour chaque shard de données, offrant ainsi une flexibilité granulaire. De plus, des outils comme **Patroni** peuvent automatiser la gestion des rôles de nœuds et garantir une cohérence même en cas de failover, minimisant les risques d’incohérences.



## 7. Conclusion et perspectives
- Résumé des forces et limites de PostgreSQL dans la distribution.
- Futur des bases de données distribuées avec PostgreSQL.

## Références

