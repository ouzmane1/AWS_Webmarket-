# Guide de Construction du Schéma d'Architecture AWS WebMarket+

Ce guide vous accompagne étape par étape pour créer un schéma d'architecture AWS professionnel et conforme aux exigences de l'examen.

---

## 1. Liste des Composants AWS à Placer

### Composants Réseau
- **Internet Gateway (IGW)** : Passerelle vers Internet
- **NAT Gateway** : Permet l'accès Internet sortant depuis les subnets privés
- **Application Load Balancer (ALB)** : Répartiteur de charge applicatif

### Composants de Calcul
- **EC2 Instance A** : Serveur d'application dans AZ-A
- **EC2 Instance B** : Serveur d'application dans AZ-B
- **Auto Scaling Group** : Groupe de mise à l'échelle automatique (englobe les EC2)

### Composants de Données
- **RDS MySQL Primary** : Base de données principale (AZ-A)
- **RDS MySQL Standby** : Réplica en attente (AZ-B)
- **Amazon S3** : Stockage d'objets pour images et assets

### Composants de Sécurité & Monitoring
- **Security Groups** : SG-ALB, SG-App, SG-DB
- **IAM Role** : EC2-S3-Access-Role
- **CloudWatch** : Monitoring (optionnel sur le schéma)

### Acteurs Externes
- **End Users** : Utilisateurs finaux de l'application

---

## 2. Hiérarchie Visuelle (Structure en Poupées Russes)

### Niveau 1 : Région AWS
```
┌─────────────────────────────────────────────────────────────┐
│ AWS Region: eu-west-3 (Paris)                               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```
- **Couleur suggérée** : Bleu clair
- **Contient** : Tout le reste de l'infrastructure

### Niveau 2 : VPC (Virtual Private Cloud)
```
┌─────────────────────────────────────────────────────────────┐
│ VPC WebMarket+ (10.0.0.0/16)                                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```
- **Couleur suggérée** : Vert clair
- **Contient** : Availability Zones et tous les subnets

### Niveau 3 : Availability Zones (AZ)
```
┌──────────────────────────┐  ┌──────────────────────────┐
│ Availability Zone A      │  │ Availability Zone B      │
│ (eu-west-3a)             │  │ (eu-west-3b)             │
│                          │  │                          │
└──────────────────────────┘  └──────────────────────────┘
```
- **Couleur suggérée** : Gris clair avec bordure pointillée
- **Disposition** : Côte à côte (gauche et droite)
- **Contient** : Subnets publics et privés

### Niveau 4 : Subnets (dans chaque AZ)

#### Dans Availability Zone A :
1. **Public Subnet A** (10.0.1.0/24)
   - Annotation : Public
   - Contient : ALB (partie A), NAT Gateway
   
2. **Private Subnet App A** (10.0.10.0/24)
   - Annotation : Private
   - Contient : EC2 Instance A
   
3. **Private Subnet DB A** (10.0.20.0/24)
   - Annotation : Private - Isolated
   - Contient : RDS Primary

#### Dans Availability Zone B :
1. **Public Subnet B** (10.0.2.0/24)
   - Annotation : Public
   - Contient : ALB (partie B)
   
2. **Private Subnet App B** (10.0.11.0/24)
   - Annotation : Private
   - Contient : EC2 Instance B
   
3. **Private Subnet DB B** (10.0.21.0/24)
   - Annotation : Private - Isolated
   - Contient : RDS Standby

---

## 3. Trajet Précis d'une Requête Utilisateur (Flux Numérotés)

### Flux Entrant : De l'Utilisateur à la Base de Données

#### Flux 1 : Utilisateur → Internet Gateway
- **Type** : HTTPS Request
- **Flèche** : Bleue, épaisse
- **Port** : 443
- **Label** : "1. HTTPS Request from End User"

#### Flux 2 : Internet Gateway → Application Load Balancer
- **Type** : Routage vers le VPC
- **Flèche** : Bleue
- **Port** : 443 → 80/443
- **Label** : "2. Route to ALB in Public Subnets"

#### Flux 3 : ALB → EC2 Instances (A et B)
- **Type** : Load Balancing
- **Flèche** : Verte (deux flèches, une vers chaque EC2)
- **Port** : 80
- **Label** : "3. Load Balancing to Healthy Instances"
- **Annotation** : Health Checks actifs

#### Flux 4 : EC2 Instances → RDS Primary
- **Type** : Database Query
- **Flèche** : Orange (depuis chaque EC2 vers RDS Primary)
- **Port** : 3306 (MySQL)
- **Label** : "4. SQL Query to Database"

### Flux Sortant : Mises à Jour et Accès Internet

#### Flux 5 : EC2 → NAT Gateway
- **Type** : Outbound Traffic
- **Flèche** : Grise, pointillée
- **Label** : "5. Outbound for Updates (via NAT)"

#### Flux 6 : NAT Gateway → Internet Gateway
- **Type** : Internet Access
- **Flèche** : Grise, pointillée
- **Label** : "6. Internet Access (Outbound Only)"

### Flux vers Stockage S3

#### Flux 7 : EC2 → Amazon S3
- **Type** : API Call (via IAM Role)
- **Flèche** : Violette
- **Label** : "7. S3 API - Get/Put Objects"
- **Annotation** : "Authentification via IAM Role (pas de clés)"

### Flux de Réplication

#### Flux 8 : RDS Primary → RDS Standby
- **Type** : Synchronous Replication
- **Flèche** : Rouge, double trait
- **Label** : "8. Multi-AZ Synchronous Replication"
- **Annotation** : "Failover automatique en cas de panne"

---

## 4. Annotations de Sécurité à Ajouter

### Security Groups (Firewalls Virtuels)

#### SG-ALB (Application Load Balancer)
**Position** : Près de l'ALB, icône bouclier vert

**Règles Inbound** :
- Source : `0.0.0.0/0` (Internet)
- Ports : `80, 443`
- Protocole : TCP

**Règles Outbound** :
- Destination : `SG-App`
- Port : `80`
- Protocole : TCP

**Annotation sur le schéma** :
```
┌─────────────────────────┐
│ SG-ALB                  │
│ IN: 0.0.0.0/0 → 80,443  │
│ OUT: SG-App → 80        │
└─────────────────────────┘
```

#### SG-App (EC2 Application Servers)
**Position** : Près des instances EC2, icône bouclier orange

**Règles Inbound** :
- Source : `SG-ALB` uniquement
- Port : `80`
- Protocole : TCP

**Règles Outbound** :
- Destination 1 : `SG-DB` → Port `3306` (MySQL)
- Destination 2 : `0.0.0.0/0` → All (pour NAT Gateway)

**Annotation sur le schéma** :
```
┌─────────────────────────┐
│ SG-App                  │
│ IN: SG-ALB → 80         │
│ OUT: SG-DB → 3306       │
│      Internet → All     │
└─────────────────────────┘
```

#### SG-DB (RDS Database)
**Position** : Près du RDS, icône bouclier rouge

**Règles Inbound** :
- Source : `SG-App` uniquement
- Port : `3306`
- Protocole : TCP

**Règles Outbound** :
- Aucune (base de données isolée)

**Annotation sur le schéma** :
```
┌─────────────────────────┐
│ SG-DB                   │
│ IN: SG-App → 3306       │
│ OUT: Aucun              │
└─────────────────────────┘
```

### IAM (Identity and Access Management)

#### IAM Role : EC2-S3-Access-Role
**Position** : Près des instances EC2 ou dans une zone dédiée

**Annotation** :
```
┌──────────────────────────────────┐
│ IAM Role: EC2-S3-Access-Role     │
│ Attaché à: EC2 Instances         │
│ Permissions:                     │
│  - s3:GetObject                  │
│  - s3:PutObject                  │
│  - s3:ListBucket                 │
│ Principe: Pas de clés en dur     │
└──────────────────────────────────┘
```

### Isolation Réseau

#### Annotation Globale sur les Subnets Privés
Ajouter un cadre ou une étiquette :
```
AUCUNE IP PUBLIQUE
Les instances App et DB ne sont PAS accessibles depuis Internet
```

#### Annotation sur la NAT Gateway
```
NAT Gateway
└─ Permet uniquement le trafic SORTANT
└─ Bloque tout trafic ENTRANT non sollicité
```

---

## 5. Légende du Schéma

Placer en bas ou sur le côté du schéma :

```
┌─────────────────────────────────────────────────────────────┐
│ LÉGENDE                                                     │
├─────────────────────────────────────────────────────────────┤
│ Public Subnet      : Accessible depuis Internet           │
│ Private Subnet     : Aucune IP publique                   │
│ ──►                 : Flux entrant (utilisateur)           │
│ ··►                 : Flux sortant (mises à jour)          │
│ ═══►                : Réplication synchrone (Multi-AZ)     │
│ SG                  : Security Group (Firewall)            │
│ 👤                  : IAM Role                             │
├─────────────────────────────────────────────────────────────┤
│ CODES COULEUR                                               │
│ Bleu   : Trafic Internet                                    │
│ Vert   : Trafic Load Balancer                              │
│ Orange : Trafic Application                                 │
│ Rouge  : Trafic Base de données                            │
│ Violet : Trafic S3                                         │
│ Gris   : Trafic sortant (updates)                          │
└─────────────────────────────────────────────────────────────┘
```

---

## 6. Checklist de Validation Finale

Avant de considérer votre schéma comme terminé, vérifiez :

### Structure
- [ ] Région AWS clairement identifiée (eu-west-3)
- [ ] VPC avec CIDR block (10.0.0.0/16)
- [ ] 2 Availability Zones distinctes (A et B)
- [ ] 6 Subnets au total (3 par AZ : Public, Private App, Private DB)

### Composants
- [ ] Internet Gateway positionné à la frontière du VPC
- [ ] NAT Gateway dans un subnet public
- [ ] ALB réparti sur les 2 subnets publics
- [ ] 2 instances EC2 (une par AZ) dans les subnets privés App
- [ ] Auto Scaling Group englobant les EC2
- [ ] RDS Primary et Standby dans les subnets privés DB
- [ ] Amazon S3 représenté (hors VPC)

### Flux
- [ ] 8 flux numérotés et documentés
- [ ] Flux entrant : User → IGW → ALB → EC2 → RDS
- [ ] Flux sortant : EC2 → NAT → IGW → Internet
- [ ] Flux S3 : EC2 → S3
- [ ] Flux réplication : RDS Primary → RDS Standby

### Sécurité
- [ ] 3 Security Groups documentés (SG-ALB, SG-App, SG-DB)
- [ ] Règles Inbound/Outbound visibles pour chaque SG
- [ ] IAM Role mentionné pour l'accès EC2 → S3
- [ ] Annotation "Aucune IP publique" sur les subnets privés
- [ ] Ports autorisés clairement indiqués (80, 443, 3306)

### Présentation
- [ ] Légende complète et lisible
- [ ] Codes couleur cohérents
- [ ] Annotations claires et professionnelles
- [ ] Schéma équilibré et aéré (pas surchargé)

---

## 7. Conseils pour un Schéma Professionnel

### Outils Recommandés
- **Diagrams.net (draw.io)** : Gratuit, bibliothèque AWS intégrée
- **Lucidchart** : Version gratuite suffisante, templates AWS
- **CloudCraft** : Spécialisé AWS, génère des estimations de coûts

### Bonnes Pratiques
1. **Utilisez les icônes officielles AWS** : Téléchargeables sur aws.amazon.com/architecture/icons
2. **Respectez la hiérarchie visuelle** : Région > VPC > AZ > Subnets
3. **Alignez les éléments** : Utilisez les grilles et guides d'alignement
4. **Numérotez les flux** : Facilite la lecture et l'explication
5. **Ajoutez des annotations** : Ne laissez rien à l'interprétation
6. **Utilisez des couleurs cohérentes** : Public = vert, Private = orange/rouge
7. **Gardez de l'espace** : Un schéma aéré est plus lisible

### Erreurs à Éviter
- ❌ Oublier de montrer la séparation Public/Private
- ❌ Ne pas documenter les Security Groups
- ❌ Placer le RDS dans un subnet public
- ❌ Oublier la NAT Gateway pour les mises à jour
- ❌ Ne pas montrer le Multi-AZ pour le RDS
- ❌ Mélanger les flux entrants et sortants sans distinction
- ❌ Surcharger le schéma avec trop de détails techniques

---

## 8. Exemple de Description à Joindre au Schéma

Accompagnez votre schéma d'une description courte :

> **Architecture AWS WebMarket+ - Haute Disponibilité et Sécurité**
>
> Cette architecture déploie l'application WebMarket+ sur AWS dans la région Paris (eu-west-3) avec une haute disponibilité Multi-AZ. Le trafic utilisateur passe par un Application Load Balancer réparti sur 2 zones de disponibilité, qui distribue les requêtes vers des instances EC2 dans des subnets privés. Les serveurs d'application accèdent à une base de données RDS MySQL en configuration Multi-AZ pour une résilience maximale. L'isolation réseau est assurée par des Security Groups restrictifs : aucune instance applicative ou base de données ne possède d'IP publique. L'accès Internet sortant (mises à jour) est géré via une NAT Gateway. Les assets statiques sont stockés sur S3 avec accès sécurisé via IAM Roles.

---

**Avec ce guide, vous disposez de tous les éléments pour créer un schéma d'architecture AWS professionnel et conforme aux attentes de l'examen !**
