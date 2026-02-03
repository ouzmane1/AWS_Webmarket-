# Argumentaire Métier : Performance, Disponibilité et Scalabilité

## Contexte et Enjeux

L'infrastructure actuelle de WebMarket+ repose sur un serveur dédié hébergé en local (On-Premise), ce qui génère des limitations critiques lors des périodes de forte activité commerciale (soldes, événements promotionnels). Cette architecture monolithique ne permet pas d'absorber les pics de charge, entraînant des ralentissements voire des interruptions de service qui impactent directement le chiffre d'affaires et la satisfaction client.

La migration vers AWS répond à trois impératifs métier : **garantir la performance en toute circonstance**, **assurer une disponibilité continue du service**, et **adapter les ressources à la demande réelle** pour optimiser les coûts.

---

## 1. Auto Scaling Group : Réponse Technique et Économique aux Pics de Charge

### 1.1 Problématique Métier

Les périodes de soldes et de promotions génèrent des pics de trafic pouvant atteindre 5 à 10 fois la charge habituelle. Avec un serveur unique dimensionné pour la charge moyenne, l'entreprise fait face à deux scénarios perdants :

- **Sous-dimensionnement** : Le serveur sature, les temps de réponse explosent, les utilisateurs abandonnent leurs paniers.
- **Sur-dimensionnement** : L'entreprise paie toute l'année pour une capacité utilisée seulement quelques semaines, générant un gaspillage financier important.

### 1.2 Solution Technique : Auto Scaling Group (ASG)

L'**Auto Scaling Group** est un mécanisme AWS qui ajuste automatiquement le nombre d'instances EC2 en fonction de métriques définies (utilisation CPU, nombre de requêtes, latence).

**Architecture retenue** :
- **Configuration de base** : 2 instances EC2 t3.medium (une par Availability Zone) pour assurer la redondance.
- **Seuils de scaling** :
  - **Scale-out** : Ajout d'instances lorsque l'utilisation CPU dépasse 70% pendant 2 minutes consécutives.
  - **Scale-in** : Suppression d'instances lorsque l'utilisation CPU descend sous 30% pendant 5 minutes.
- **Limites** : Minimum 2 instances (haute disponibilité), maximum 10 instances (protection budgétaire).

**Flux opérationnel** :
1. **Détection** : CloudWatch surveille en continu les métriques des instances.
2. **Déclenchement** : Lorsqu'un seuil est franchi, l'ASG lance ou termine des instances.
3. **Intégration** : Les nouvelles instances sont automatiquement enregistrées auprès de l'Application Load Balancer.
4. **Health Checks** : Seules les instances saines reçoivent du trafic.

### 1.3 Bénéfices Métier et Économiques

**Performance** :
- **Élasticité horizontale** : Capacité à passer de 2 à 10 instances en moins de 3 minutes.
- **Absorption des pics** : Lors des soldes, le système peut gérer jusqu'à 5 fois plus de requêtes simultanées sans dégradation.
- **Temps de réponse stable** : Maintien d'une latence inférieure à 200ms même en période de forte charge.

**Économie** :
- **Pay-per-use** : Paiement uniquement des instances actives, à la seconde près.
- **Exemple concret** : En dehors des périodes de soldes (10 mois/an), l'entreprise ne paie que 2 instances au lieu de 10, soit une économie de 80% sur le calcul.
- **ROI estimé** : Réduction de 40 à 60% des coûts d'infrastructure par rapport à un dimensionnement statique pour le pic.

**Résilience** :
- **Remplacement automatique** : Si une instance tombe en panne, l'ASG la remplace automatiquement.
- **Déploiement sans interruption** : Possibilité de mettre à jour l'application en mode "rolling update" sans coupure de service.

---

## 2. Application Load Balancer : Garantie de Haute Disponibilité

### 2.1 Problématique Métier

Avec un serveur unique, toute panne matérielle, logicielle ou réseau entraîne une interruption totale du service. Pour une application de gestion de stocks utilisée par des commerçants, chaque minute d'indisponibilité se traduit par :
- **Perte de chiffre d'affaires** : Impossibilité de passer des commandes.
- **Dégradation de l'image de marque** : Perte de confiance des clients.
- **Risques opérationnels** : Incapacité à gérer les stocks en temps réel.

### 2.2 Solution Technique : Application Load Balancer (ALB)

L'**Application Load Balancer** est un répartiteur de charge intelligent opérant au niveau 7 du modèle OSI (couche application HTTP/HTTPS).

**Architecture retenue** :
- **Déploiement Multi-AZ** : L'ALB est automatiquement réparti sur les 2 Availability Zones (eu-west-3a et eu-west-3b).
- **Algorithme de distribution** : Round Robin avec prise en compte de la charge réelle des instances.
- **Health Checks** : Vérification toutes les 30 secondes de la disponibilité des instances via une route `/health`.

**Fonctionnement** :
1. **Réception** : L'ALB reçoit les requêtes HTTPS des utilisateurs sur le port 443.
2. **Terminaison SSL** : Déchiffrement du trafic HTTPS au niveau de l'ALB (offload SSL).
3. **Routage intelligent** : Distribution des requêtes vers les instances EC2 saines uniquement.
4. **Sticky Sessions** : Maintien de la session utilisateur sur la même instance (via cookies) pour préserver l'état applicatif.

### 2.3 Bénéfices Métier

**Haute Disponibilité** :
- **Tolérance aux pannes** : Si une instance ou une Availability Zone tombe, l'ALB redirige automatiquement le trafic vers les instances saines.
- **SLA AWS** : Disponibilité garantie de 99.99% (soit moins de 4 minutes d'indisponibilité par mois).
- **Pas de Single Point of Failure** : L'ALB lui-même est redondant et géré par AWS.

**Performance** :
- **Répartition optimale** : Distribution équitable de la charge entre toutes les instances disponibles.
- **Latence réduite** : Routage vers l'instance la plus proche géographiquement (dans la même AZ si possible).
- **Scalabilité automatique** : L'ALB s'adapte automatiquement au volume de trafic sans configuration manuelle.

**Sécurité** :
- **Point d'entrée unique** : Seul composant exposé à Internet, simplifie la gestion des règles de sécurité.
- **Protection DDoS** : Intégration native avec AWS Shield Standard (protection contre les attaques volumétriques).
- **Certificats SSL/TLS** : Gestion centralisée via AWS Certificate Manager (ACM) avec renouvellement automatique.

**Observabilité** :
- **Métriques CloudWatch** : Nombre de requêtes, latence, taux d'erreur HTTP, instances saines/défaillantes.
- **Logs d'accès** : Traçabilité complète des requêtes pour l'audit et le troubleshooting.

---

## 3. RDS Multi-AZ : Résilience Critique des Données de Stock

### 3.1 Problématique Métier

La base de données contient les informations critiques de l'entreprise : **catalogue produits, niveaux de stock, historique des transactions**. Une perte de données ou une indisponibilité prolongée aurait des conséquences catastrophiques :

- **Perte de données** : Impossibilité de reconstituer l'état des stocks, désynchronisation avec la réalité terrain.
- **Interruption métier** : Les commerçants ne peuvent plus consulter ni mettre à jour leurs stocks.
- **Non-conformité** : Risques légaux en cas de perte de données comptables ou fiscales.

### 3.2 Solution Technique : Amazon RDS Multi-AZ

**Amazon RDS (Relational Database Service)** est un service de base de données managée qui automatise les tâches d'administration (sauvegardes, patchs, monitoring). La configuration **Multi-AZ** ajoute une couche de résilience critique.

**Architecture retenue** :
- **Instance Primary** : Base de données principale dans l'Availability Zone A (eu-west-3a).
- **Instance Standby** : Réplica synchrone dans l'Availability Zone B (eu-west-3b).
- **Réplication synchrone** : Chaque transaction est écrite simultanément sur les deux instances avant confirmation.
- **Endpoint unique** : Les applications se connectent à un DNS unique qui pointe automatiquement vers l'instance active.

**Fonctionnement du Failover** :
1. **Détection de panne** : RDS détecte une défaillance de l'instance Primary (panne matérielle, réseau, AZ).
2. **Bascule automatique** : Le DNS est automatiquement redirigé vers l'instance Standby (devient Primary).
3. **Durée du failover** : 60 à 120 secondes en moyenne, sans intervention humaine.
4. **Transparence applicative** : Les applications se reconnectent automatiquement via le même endpoint.

### 3.3 Bénéfices Métier

**Résilience des Données** :
- **Durabilité** : Aucune perte de données en cas de panne (réplication synchrone).
- **RPO = 0** (Recovery Point Objective) : Toutes les transactions validées sont préservées.
- **RTO < 2 minutes** (Recovery Time Objective) : Reprise du service en moins de 2 minutes.

**Disponibilité** :
- **SLA AWS** : 99.95% de disponibilité mensuelle garantie.
- **Protection contre les pannes d'AZ** : Résistance aux défaillances d'un datacenter entier.
- **Maintenance sans interruption** : Les mises à jour de sécurité sont appliquées sur le Standby puis bascule, sans coupure.

**Sauvegardes Automatisées** :
- **Snapshots quotidiens** : Sauvegarde complète de la base chaque jour, conservée 7 jours par défaut.
- **Point-in-Time Recovery** : Possibilité de restaurer la base à n'importe quel moment des 7 derniers jours (granularité : 5 minutes).
- **Stockage sécurisé** : Sauvegardes chiffrées et répliquées sur plusieurs AZ.

**Performance** :
- **Isolation réseau** : La base de données est dans un subnet privé, inaccessible depuis Internet.
- **Connexions optimisées** : Les instances EC2 se connectent directement au RDS via le réseau interne AWS (latence < 5ms).
- **Scaling vertical** : Possibilité d'augmenter la puissance de l'instance (CPU, RAM) sans changer d'architecture.

**Réduction de la Charge Opérationnelle** :
- **Automatisation** : Sauvegardes, patchs de sécurité, monitoring gérés par AWS.
- **Libération de l'équipe** : L'équipe interne peut se concentrer sur le métier plutôt que sur l'administration de bases de données.
- **Expertise AWS** : Bénéfice de l'expérience AWS sur des millions de bases de données en production.

---

## 4. Synthèse : Architecture Résiliente et Performante

L'architecture proposée répond de manière exhaustive aux limites identifiées dans le contexte actuel :

| **Problème Actuel** | **Solution AWS** | **Bénéfice Métier** |
|---------------------|------------------|---------------------|
| Incapacité à absorber les pics de charge | Auto Scaling Group | Élasticité automatique, performance stable |
| Serveur unique = point de défaillance | ALB Multi-AZ + ASG | Haute disponibilité (99.99%) |
| Risque de perte de données | RDS Multi-AZ + Sauvegardes | Résilience totale (RPO=0, RTO<2min) |
| Coûts fixes élevés | Pay-per-use + Scaling | Réduction de 40-60% des coûts |
| Dépendance à l'équipe interne | Services managés AWS | Libération de 70% du temps d'administration |

**Conclusion** : Cette architecture garantit une **performance constante** même en période de forte charge, une **disponibilité de 99.99%** grâce à la redondance Multi-AZ, et une **résilience totale des données critiques** via RDS Multi-AZ. Elle transforme un coût fixe en coût variable aligné sur l'activité réelle, tout en réduisant drastiquement la charge opérationnelle de l'équipe interne.
