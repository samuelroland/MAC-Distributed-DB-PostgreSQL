<!--
# Based on DAI theme:
# https://github.com/heig-vd-dai-course/heig-vd-dai-course/blob/7eb91c050f1bad18f766b4a4916d2ebc425be47b/04-java-intellij-idea-and-maven/PRESENTATION.md?plain=1#L8
theme: gaia
size: 16:9
paginate: true
style: |
    :root {
        --color-background: #fff;
        --color-foreground: #333;
        --color-highlight: #f96;
        --color-dimmed: #1566aa;
        --color-headings: #336791;
    }
    blockquote {
        font-style: italic;
    }
    table {
        width: 100%;
    }
    section {
        padding: 30px;
    }
    ul {
        margin: 0.4rem 0
    }
    p, li {
        font-size: 0.6rem
    }
    th:first-child {
        width: 15%;
    }
    h1, h2, h3, h4, h5, h6 {
        color: var(--color-headings);
    }
    h2, h3 {
        font-size: 1.1rem;
    }
    h4, h5, h6 {
        font-size: 1.0rem;
    }
    h1 a:link, h2 a:link, h3 a:link, h4 a:link, h5 a:link, h6 a:link {
        text-decoration: none;
    }
    section:not([class=lead]) > p, blockquote {
        text-align: justify;
    }
    .columns2 {
        display: grid;
        grid-template-columns: repeat(2, minmax(0, 1fr));
        gap: 1rem;
    }
    .columns3 {
        display: grid;
        grid-template-columns: repeat(3, minmax(0, 1fr));
        gap: 1rem;
    }
-->
<center style="margin:auto; padding:20px;">

# PostgreSQL en système distribué

##### Timothée Van Hove et Samuel Roland

<div style="height: 500px; display: flex; justify-content: center; align-items: center">

![width:300](imgs/postgresql.logo.png) ![width:400](imgs/citus.svg#citus-elicorn-green)

</div>
</center>


---

### Comment faire de PostgreSQL un SGBD distribué?

<!--

PostgreSQL, est conçu à l'origine pour des systèmes monolithiques.

Nous allons voir ensemble que PostgreSQL peut aussi être utilisé comme un SGBD distribué avec des fonctionnalités de réplication, mais aussi de partitionnement et de sharding avec des extensions.


Dans une première partie, nous allons explorer les possibilités de réplication

Dans une deuxième partie, nous allons expliquer les capacités natives de partitionnement et les extensions permettant de mettre en place du sharding.

-->

Nous allons traiter 2 axes principaux de la distribution dans PostgreSQL :

- **Réplication** : Garantir la disponibilité des données en les répliquant entre plusieurs serveurs.
- **Partitionnement et Sharding** : Diviser les données pour améliorer la scalabilité et l'efficacité.


---

<center>

### Réplication dans PGSQL

![width:700](imgs/replication-intro.jpg)

</center>

---

### Qu’est-ce que la Streaming Replication ?

<!--

La streaming replication de PostgreSQL, la plus courante, est une réplication physique qui réplique les changements au niveau byte par byte, créant une copie identique de la base de données sur des autres serveurs.

C'est une réplication à **leader unique** qui fonctionne en transmettant les journaux WAL (Write-Ahead Logging) depuis le leader vers les répliques via une connexion réseau.

Les followers reçoivent ces journaux de manière quasi-continue, ce qui permet de maintenir leur état aussi proche que possible de celui du leader.

Un processus appelé WAL receiver, fonctionnant sur le serveur répliqué, se connectera au serveur primaire à l'aide d'une connexion TCP. 

Sur le serveur primaire, il existe un autre processus, appelé WAL sender, qui est chargé d'envoyer les registres WAL au serveur de secours au fur et à mesure.

-->


* Réplication la plus courante (leader unique)
* Utilise les journaux WAL pour synchroniser les followers (standby nodes) avec le leader (primary node).
* Les followers recoivent les journeaux WAL de manière quasi-continue


![width:1000](imgs/streaming-replication.png)

---

### Streaming Replication - Modes

<!--

Il existe deux modes de Streaming replication, synchrone et asynchrone.

* Synchrone: Si une réplique distante se trouve sur un autre continent ou utilise une connexion réseau lente, la latence peut augmenter considérablement le temps nécessaire pour valider les transactions. Cela peut affecter les performances globales du système. **Cas d’usage : Systèmes critiques (ex. : banques).**
* Asynchrone: Les répliques peuvent accumuler un léger retard (replication lag), mais elles restent proches du leader si les ressources réseau et matérielles sont suffisantes. **Cas d’usage : Applications tolérant des incohérences temporaires (ex. : réseaux sociaux).**
* Réplication en cascade: Si on a beaucoup de répliques, envoyer directement les journaux à chacune peut surcharger le Primary Server. En utilisant la cascade, certains folowers agissent comme relais. **Cas d'usage** : Dans des environnements distribués géographiquement, un Standby Server intermédiaire peut être positionné plus près des autres serveurs pour minimiser la latence réseau.

-->

<div class="columns2">

<div>

* **Mode Synchrone**
  * Le Primary Server (Leader) attend la confirmation des répliques avant de valider une transaction.
* **Mode Asynchrone**
  * Le Primary Server (Leader) n'attend pas de confirmation des répliques ; il envoie les WAL dès qu'ils sont générés.
* **Réplication en cascade**
  * Un Standby Server (Follower) peut avoir le charge d'envoyer les WAL à d'autres followers. On parle de "Sending server"
  * ça permet de réduire la charge du leader

</div>

<div>

![width:600](imgs/cascade.png)

</div>
</div>

---

### Qu’est-ce que la Logical Replication ?

<!--

* La réplication logique réplique les données au niveau des lignes et des colonnes. Contrairement à la Streaming replication, elle ne copie pas les blocs de données au niveau byte, mais les modifications au niveau logique.
* Elle fonctionne sur le principe publisher-subscriber:
  * **Publisher** : Définit des publications (ensembles de données et types de changements à répliquer).
  * **Subscriber** : Souscrit à une ou plusieurs publications et applique les changements.

-->


* Réplique les modifications au niveau des transactions (lignes/tables spécifiques).
* Fonctionne via un **publisher** (leader) et des **subscribers** (followers)
* Le Publisher transforme le WAL en opérations transactionnelles (UPDATE, INSERT, DELETE...)
* Les opérations sont envoyées aux subscribers, puis sont appliquées dans le même ordre transactionnel que sur le Publisher.
* Les schémas doivent être identiques ou compatibles entre Publisher et Subscriber.



![width:800](imgs/logical-replication-simple.png)

<!--

---

### Comment fonctionne la Logical Replication ?


- Le **processus wal sender** côté Publisher extrait les modifications à partir du WAL.
- Il utilise un **plugin de décodage logique** (`pgoutput` par défaut) pour traduire ces modifications en un format compréhensible pour la réplication logique.
- Les modifications sont ensuite envoyées aux subscribers

- **Le processus apply worker** sur le Subscriber reçoit les modifications.
- Il les mappe aux tables locales et applique chaque modification dans le même ordre transactionnel que sur le Publisher.


* Le Publisher transforme le WAL en opérations transactionnelles
* Les informations sont envoyées aux subscribers
* Les schémas doivent être identiques ou compatibles entre Publisher et Subscriber.


<center>

![width:700](imgs/logical-replication.png)
</center>

-->

---

### Streaming vs Logical Replication

<div class="columns3">
<div>

**Streaming Replication**

- Objectif : Maintenir une copie exacte de la base pour haute disponibilité et basculement.
- Avantages :
  - Simple à configurer.
  - Faible latence.
- Limites :
  - Réplique toute la base.
  - Pas de personnalisation ou de filtrage des données.


</div>
<div>

**Logical Replication**

- Objectif : Partager des données spécifiques
- Avantages :
  - Flexible, permet de cibler des tables ou types de modifications.
  - Compatible entre versions ou plateformes.
- Limites :
  - Conflits possibles en cas d'écritures locales sur le Subscriber.
  - Schéma non répliqué automatiquement.

</div>
<div>

![width:350](imgs/streaming-logical.jpeg)

</div>
</div>


---

### "Réplication multi-leader ? Bi-Directional Replication"

<!--
BDR est une extension conçue pour offrir la réplication multi-leader. Dans ce modèle, plusieurs nœuds peuvent agir comme leaders, chacun acceptant des écritures. ça permet une répartition des charges d'écriture.

BDR se base sur la réplication logique. Chaque modification effectuée sur un nœud est répliquée aux autres, et les conflits potentiels sont gérés grâce à un système de détection et de résolution des conflits.

Les conflits apparaissent lorsque deux nœuds modifient simultanément une même ligne. Par défaut, BDR applique un résolveur appelé `update_if_newer`, qui conserve la version la plus récente basée sur le timestamp de commit. Si les timestamps sont identiques, l'ID du nœud est utilisé comme critère de départage.

-->

<div style="display: flex">

![width:400](imgs/multi-leader.jpg)

<div style="margin-left: 20px">

PGSQL ne supporte pas la réplication multi-leader nativement.  BDR est une extension pour la réplication multi-leader basée sur le logical replication:

* Chaque nœud agit comme un leader capable d’accepter des écritures
* Les conflits d’écriture sont détectés lorsque plusieurs nœuds modifient les mêmes données.
* Utilise des résolveurs de conflits pour déterminer comment gérer ces situations.
  * Par défaut, BDR applique le résolveur `update_if_newer`, qui conserve la version de la ligne ayant le timestamp de commit le plus récent.

* **Avantages :**
  * Partage de la charge d’écriture entre plusieurs nœuds.

* **Limites :**
  * Complexité : gestion des conflits entre les nœuds.
  * Licence commerciale et non open-source complète.

</div>
</div>

---

## Partionnement et sharding


![width:900](imgs/cake.png)

> https://www.thegeekyminds.com/post/database-sharding-vs-partitioning

```sql
select username from users where company_id = 12; -- Entreprise 12
select price from invoices where date >= '2024-05-01' AND date < '2024-06-01' -- Mai 2024
select name from events where date = '2024-05-01' -- Jour spécifique
```

---

![width:1100](./imgs/partionnement.png)

<!-- --- -->
<!---->
<!-- ## Késako ? -->
<!---->
<!-- <div class="columns2"> -->
<!---->
<!-- <div> -->
<!---->
<!-- **Partionnement** -->
<!---->
<!-- </div> -->
<!---->
<!-- <div> -->
<!---->
<!-- **Sharding** -->
<!---->
<!-- </div> -->
<!-- </div> -->
<!---->

<!-- --- -->
<!-- ## 3 type de partionnements -->
<!-- 1. Partitionnement par plages (range partitioning) -->
<!-- 1. Partitionnement par liste (list partitioning) -->
<!-- 1. Partitionnement par hachage (hash partitioning) -->
<!---->
<!-- <div style="display: flex"> -->
<!---->
<!-- ![width:400](imgs/multi-leader.jpg) -->
<!---->
<!-- <div style="margin-left: 20px"> -->
<!---->
<!---->
<!-- - test -->
<!-- - test -->
<!---->
<!-- </div> -->
<!-- </div> -->

