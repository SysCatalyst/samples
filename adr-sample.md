# ADR-0015 : Utiliser l'Event Sourcing pour le domaine Commandes

## Statut

Accepted

Date: 2024-02-20
Auteurs: Marie Dupont, Jean Martin
Reviewers: Équipe Architecture

## Contexte et problème

Le système de gestion des commandes actuel utilise un modèle CRUD classique.
Nous rencontrons plusieurs problèmes :

1. **Audit insuffisant** : impossible de retracer l'historique complet d'une commande
2. **Conflits de mise à jour** : les modifications concurrentes créent des incohérences
3. **Analyse métier limitée** : les données historiques sont perdues après mise à jour
4. **Conformité RGPD** : difficulté à prouver l'historique des consentements

Le volume actuel est de 50 000 commandes/jour, projection à 200 000 dans 18 mois.

## Decision Drivers

* Traçabilité complète de chaque modification
* Performance en lecture (dashboards temps réel)
* Capacité de « replay » pour correction d'erreurs
* Conformité audit SOC 2

## Options considérées

### Option 1 : Ajouter une table d'audit

Conserver le modèle CRUD et ajouter une table `order_audit` avec triggers.

**Avantages** :
- Changement minimal du code existant
- Équipe familière avec l'approche

**Inconvénients** :
- Ne résout pas les conflits de concurrence
- Performance des triggers à grande échelle
- Audit partiel (pas toutes les modifications)

### Option 2 : Event Sourcing (choisi)

Stocker chaque modification comme un événement immuable.
Reconstituer l'état courant par projection des événements.

**Avantages** :
- Traçabilité totale par design
- Résout la concurrence (append-only)
- Permet le replay et les projections multiples
- Adapté aux patterns CQRS

**Inconvénients** :
- Courbe d'apprentissage (pattern peu connu de l'équipe)
- Complexité des migrations de schéma d'événements
- Besoin d'un event store (infrastructure nouvelle)

### Option 3 : Temporal tables (SQL Server)

Utiliser les temporal tables natives de SQL Server.

**Avantages** :
- Intégré au SGBD
- Pas de changement d'architecture

**Inconvénients** :
- Vendor lock-in
- Pas de support pour les projections personnalisées
- Limité aux requêtes SQL

## Décision

**Nous adoptons l'Event Sourcing pour le domaine Commandes.**

L'implémentation utilisera :
- EventStoreDB comme event store (open-source, optimisé pour ce pattern)
- Projections vers PostgreSQL pour les lectures
- Format CloudEvents pour la structure des événements

## Conséquences

### Positives

* Audit natif : chaque changement est un événement horodaté
* Résolution de la concurrence par versioning des agrégats
* Nouvelles possibilités : replay, what-if analysis, debug temporel
* Base solide pour futures fonctionnalités analytiques

### Négatives

* 4 semaines de formation équipe (workshop Event Sourcing)
* 2 sprints pour migration du domaine Commandes existant
* Nouvelle infrastructure : cluster EventStoreDB à provisionner
* Complexité accrue pour les développeurs juniors

### Actions requises

1. [ ] Formation équipe (S1-S2)
2. [ ] Provisioning EventStoreDB (S2)
3. [ ] Migration progressive domaine Commandes (S3-S6)
4. [ ] Documentation des patterns pour autres domaines

## Liens

* RFC: /docs/rfc/0008-event-sourcing-strategy.md
* Benchmark: /docs/benchmarks/eventstore-performance.md
* Formation: https://internal-wiki/event-sourcing-workshop
