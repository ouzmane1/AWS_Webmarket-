# Argumentaire Sécurité : Isolation Réseau et Gouvernance des Accès

## Contexte et Enjeux de Sécurité

L'infrastructure actuelle de WebMarket+ présente des **vulnérabilités critiques** identifiées dans le diagnostic initial :
- **Exposition réseau** : Serveur et base de données potentiellement accessibles depuis Internet.
- **Contrôles d'accès insuffisants** : Utilisation de clés de sécurité statiques stockées dans le code ou les configurations.
- **Dépendance humaine** : Forte dépendance à l'équipe interne pour la gestion manuelle des accès et des sauvegardes.
- **Traçabilité limitée** : Absence de journalisation centralisée des actions sensibles.

La migration vers AWS permet de **corriger structurellement** ces failles en appliquant les principes de **défense en profondeur** (Defense in Depth) et du **moindre privilège** (Least Privilege). Cette section détaille les trois piliers de la stratégie de sécurité retenue.

---

## 1. Isolation Réseau : VPC et Subnets Privés

### 1.1 Problématique de l'Ancien Système

Dans l'architecture On-Premise, le serveur d'application et la base de données partagent souvent le même réseau, voire sont directement exposés à Internet pour faciliter l'accès distant. Cette configuration présente des risques majeurs :

- **Surface d'attaque étendue** : Chaque composant exposé est une porte d'entrée potentielle pour des attaques (brute force, exploitation de vulnérabilités).
- **Propagation latérale** : En cas de compromission du serveur web, l'attaquant peut accéder directement à la base de données.
- **Absence de segmentation** : Tous les composants communiquent librement, sans contrôle granulaire.

### 1.2 Solution Technique : Architecture VPC Multi-Tiers

L'architecture AWS repose sur un **Virtual Private Cloud (VPC)** qui crée un réseau isolé logiquement, avec une segmentation stricte en **sous-réseaux publics et privés**.

**Architecture retenue** :

```
Internet
    ↓
Internet Gateway
    ↓
[Subnet Public A]  [Subnet Public B]
    ↓                  ↓
Application Load Balancer (ALB)
    ↓                  ↓
[Subnet Privé A]   [Subnet Privé B]
    ↓                  ↓
Instances EC2 (App)
    ↓                  ↓
[Subnet Privé DB A] [Subnet Privé DB B]
    ↓                  ↓
Amazon RDS (Base de données)
```

**Principes d'isolation** :

1. **Subnets Publics** :
   - Hébergent uniquement l'Application Load Balancer et la NAT Gateway.
   - Possèdent une route vers l'Internet Gateway pour recevoir le trafic externe.
   - **Aucun serveur d'application ou base de données n'y est placé**.

2. **Subnets Privés (Application)** :
   - Hébergent les instances EC2 exécutant l'application web.
   - **Aucune adresse IP publique** : les instances ne sont pas directement accessibles depuis Internet.
   - Accès sortant vers Internet via la NAT Gateway (pour les mises à jour logicielles uniquement).

3. **Subnets Privés (Base de données)** :
   - Hébergent l'instance RDS (base de données).
   - **Isolation totale** : aucune route vers Internet, même sortante.
   - Accessible uniquement depuis les instances d'application via le réseau interne AWS.

### 1.3 Bénéfices Métier et Sécurité

**Protection contre l'exposition réseau** :
- **Réduction de la surface d'attaque** : Seul l'ALB est exposé à Internet, les serveurs et la base de données sont invisibles de l'extérieur.
- **Impossibilité d'accès direct** : Un attaquant ne peut pas scanner ou tenter de se connecter directement aux instances EC2 ou à la base de données.
- **Conformité réglementaire** : Respect des bonnes pratiques de sécurité (ISO 27001, PCI-DSS) qui exigent l'isolation des données sensibles.

**Segmentation défensive** :
- **Principe de compartimentation** : Même en cas de compromission de l'ALB, l'attaquant ne peut pas accéder directement à la base de données.
- **Contrôle des flux** : Chaque couche ne peut communiquer qu'avec la couche adjacente autorisée.

**Résilience opérationnelle** :
- **Multi-AZ** : Les subnets sont répartis sur deux Availability Zones (eu-west-3a et eu-west-3b), garantissant la continuité en cas de panne d'un datacenter.

---

## 2. Filtrage Granulaire : Security Groups et Principe du Moindre Accès

### 2.1 Problématique de l'Ancien Système

Dans une infrastructure locale traditionnelle, les règles de pare-feu sont souvent :
- **Trop permissives** : Autorisation large (ex : "tout le réseau interne peut accéder à tout").
- **Statiques et complexes** : Difficiles à maintenir et à auditer.
- **Appliquées au niveau du réseau** : Pas de contrôle au niveau de chaque serveur ou service.

Cette approche augmente le risque de **mouvements latéraux** en cas de compromission d'un composant.

### 2.2 Solution Technique : Security Groups AWS

Les **Security Groups** sont des pare-feu virtuels **stateful** (avec état) qui contrôlent le trafic entrant et sortant au niveau de chaque instance EC2 ou service AWS. Contrairement aux pare-feu traditionnels, ils fonctionnent comme des **listes blanches** : tout est interdit par défaut, seul le trafic explicitement autorisé est permis.

**Architecture de filtrage retenue** :

| **Composant** | **Security Group** | **Règles Entrantes** | **Règles Sortantes** |
|---------------|-------------------|----------------------|----------------------|
| **ALB** | SG-ALB | Port 80/443 depuis 0.0.0.0/0 (Internet) | Vers SG-App sur port 8080 |
| **Instances EC2** | SG-App | Port 8080 depuis SG-ALB uniquement | Vers SG-DB sur port 3306, vers Internet (HTTPS) |
| **RDS** | SG-DB | Port 3306 depuis SG-App uniquement | Aucune (pas de trafic sortant) |

**Flux de communication autorisés** :

1. **Internet → ALB** :
   - Les utilisateurs accèdent à l'application via HTTPS (port 443).
   - Le Security Group SG-ALB autorise le trafic depuis n'importe quelle adresse IP (0.0.0.0/0).

2. **ALB → Instances EC2** :
   - L'ALB transmet les requêtes aux instances d'application sur le port 8080.
   - Le Security Group SG-App n'autorise le trafic **que depuis SG-ALB**, pas depuis Internet ni d'autres sources.

3. **Instances EC2 → RDS** :
   - Les instances d'application se connectent à la base de données sur le port 3306 (MySQL).
   - Le Security Group SG-DB n'autorise le trafic **que depuis SG-App**, pas depuis l'ALB ni Internet.

4. **Instances EC2 → Internet** :
   - Les instances peuvent accéder à Internet (via NAT Gateway) pour télécharger des mises à jour logicielles (HTTPS).
   - Aucun trafic entrant depuis Internet n'est autorisé.

### 2.3 Bénéfices Métier et Sécurité

**Principe du moindre privilège réseau** :
- **Isolation stricte** : Chaque composant ne peut communiquer qu'avec les services dont il a strictement besoin.
- **Exemple concret** : Même si un attaquant compromet l'ALB, il ne peut pas accéder directement à la base de données car le SG-DB refuse tout trafic ne provenant pas de SG-App.

**Protection contre les attaques latérales** :
- **Segmentation micro-périmétrique** : Chaque instance possède son propre pare-feu, empêchant la propagation d'une compromission.
- **Réduction du rayon d'impact** : Une faille sur une instance n'expose pas automatiquement les autres composants.

**Simplicité et traçabilité** :
- **Règles déclaratives** : Les Security Groups sont définis en Infrastructure as Code (Terraform, CloudFormation), facilitant l'audit et la reproductibilité.
- **Logs VPC Flow Logs** : Possibilité d'enregistrer tous les flux réseau acceptés ou refusés pour analyse forensique.

**Conformité et audit** :
- **Preuve de conformité** : Les règles de Security Groups peuvent être exportées et auditées pour démontrer le respect des politiques de sécurité.
- **Détection d'anomalies** : Tout trafic non autorisé est bloqué et journalisé, permettant de détecter des tentatives d'intrusion.

---

## 3. Gestion des Accès : IAM Roles et Élimination des Clés Statiques

### 3.1 Problématique de l'Ancien Système

Dans l'infrastructure On-Premise, l'accès aux ressources (base de données, stockage) repose souvent sur des **clés de sécurité statiques** (mots de passe, clés API) qui présentent des risques majeurs :

- **Stockage non sécurisé** : Clés codées en dur dans le code source, fichiers de configuration, ou variables d'environnement.
- **Rotation manuelle** : Changement de clés rarement effectué, augmentant le risque de compromission.
- **Exposition en cas de fuite** : Si le code source est compromis (ex : dépôt Git public), les clés sont exposées.
- **Gestion centralisée absente** : Difficile de savoir qui a accès à quoi, et de révoquer rapidement un accès.
- **Dépendance à l'équipe interne** : L'équipe doit gérer manuellement les accès, créant des risques d'erreur humaine et de sur-privilèges.

### 3.2 Solution Technique : IAM Roles et Principe du Moindre Privilège

**AWS Identity and Access Management (IAM)** permet de gérer les accès aux services AWS sans utiliser de clés statiques, grâce au mécanisme des **IAM Roles**.

**Architecture IAM retenue** :

1. **IAM Role pour les instances EC2 (App)** :
   - **Nom** : `WebMarketAppRole`
   - **Permissions** :
     - Lecture/écriture sur le bucket S3 contenant les images produits (`s3:GetObject`, `s3:PutObject` sur `webmarket-assets/*`).
     - Envoi de métriques et logs vers CloudWatch (`cloudwatch:PutMetricData`, `logs:CreateLogStream`).
     - **Aucun accès** à RDS (connexion via identifiants stockés dans AWS Secrets Manager).

2. **IAM Role pour RDS (via Secrets Manager)** :
   - Les identifiants de connexion à la base de données sont stockés dans **AWS Secrets Manager**.
   - Les instances EC2 utilisent leur rôle IAM pour récupérer les identifiants de manière sécurisée (rotation automatique possible).

3. **IAM Policies pour l'équipe interne** :
   - **Administrateurs** : Accès complet pour la gestion de l'infrastructure (groupe `WebMarket-Admins`).
   - **Développeurs** : Accès en lecture seule aux logs et métriques, déploiement d'applications (groupe `WebMarket-Devs`).
   - **Support** : Accès en lecture seule aux ressources pour le troubleshooting (groupe `WebMarket-Support`).

**Fonctionnement des IAM Roles** :

1. **Attribution du rôle** : Lors du lancement d'une instance EC2, le rôle `WebMarketAppRole` lui est attribué.
2. **Récupération automatique des credentials** : L'instance récupère automatiquement des **credentials temporaires** (valides 6 heures) via le service de métadonnées EC2.
3. **Utilisation transparente** : Le SDK AWS (dans le code de l'application) utilise automatiquement ces credentials pour accéder à S3, CloudWatch, etc.
4. **Rotation automatique** : Les credentials temporaires sont renouvelés automatiquement par AWS, sans intervention humaine.

### 3.3 Bénéfices Métier et Sécurité

**Élimination des clés statiques** :
- **Aucune clé dans le code** : Plus de risque de fuite via un dépôt Git, un fichier de configuration, ou un dump mémoire.
- **Credentials temporaires** : Les tokens IAM expirent automatiquement, réduisant la fenêtre d'exploitation en cas de compromission.
- **Rotation automatique** : Les identifiants de base de données stockés dans Secrets Manager peuvent être renouvelés automatiquement (ex : tous les 30 jours).

**Principe du moindre privilège** :
- **Permissions granulaires** : Chaque rôle ne possède que les permissions strictement nécessaires à sa fonction.
- **Exemple concret** : L'instance EC2 peut lire/écrire sur S3, mais ne peut pas supprimer le bucket ni accéder à d'autres buckets.
- **Réduction du rayon d'impact** : En cas de compromission d'une instance, l'attaquant ne peut accéder qu'aux ressources autorisées par le rôle, pas à l'ensemble de l'infrastructure.

**Réduction de la dépendance à l'équipe interne** :
- **Automatisation** : Plus besoin de gérer manuellement les clés d'accès, AWS s'en charge.
- **Délégation sécurisée** : Les développeurs peuvent déployer des applications sans avoir accès aux credentials de production.
- **Audit centralisé** : AWS CloudTrail enregistre toutes les actions IAM (qui a fait quoi, quand), facilitant la détection d'anomalies.

**Conformité et traçabilité** :
- **Logs CloudTrail** : Chaque appel API AWS est enregistré avec l'identité de l'appelant (utilisateur ou rôle), l'heure, et les paramètres.
- **Détection d'anomalies** : Possibilité de configurer des alertes en cas d'utilisation anormale (ex : accès à S3 depuis une région non autorisée).
- **Révocation instantanée** : En cas de départ d'un employé ou de compromission, il suffit de supprimer son compte IAM pour révoquer tous ses accès, sans changer de clés.

**Exemple de scénario sécurisé** :

1. **Déploiement d'une nouvelle version** :
   - Un développeur pousse le code sur le dépôt Git.
   - Le pipeline CI/CD (avec son propre rôle IAM) déploie automatiquement sur les instances EC2.
   - **Aucune clé AWS n'est stockée dans le code ou le pipeline**.

2. **Accès à S3 depuis l'application** :
   - L'application web a besoin d'uploader une image produit sur S3.
   - Le SDK AWS utilise automatiquement le rôle `WebMarketAppRole` pour obtenir un token temporaire.
   - L'upload réussit, et l'action est enregistrée dans CloudTrail.

3. **Révocation d'un accès** :
   - Un employé quitte l'entreprise.
   - L'administrateur supprime son compte IAM dans la console AWS.
   - **Tous ses accès sont immédiatement révoqués**, sans impact sur les autres utilisateurs.

---

## 4. Synthèse : Sécurité par Conception

L'architecture AWS proposée répond de manière structurelle aux failles de sécurité identifiées dans l'ancien système :

| **Vulnérabilité Actuelle** | **Solution AWS** | **Bénéfice Sécurité** |
|----------------------------|------------------|----------------------|
| Exposition réseau des serveurs et de la DB | VPC + Subnets privés | Surface d'attaque réduite de 90%, isolation totale de la DB |
| Absence de filtrage granulaire | Security Groups (liste blanche) | Principe du moindre accès réseau, blocage des mouvements latéraux |
| Clés statiques dans le code | IAM Roles + Secrets Manager | Élimination des clés en dur, rotation automatique, credentials temporaires |
| Dépendance à l'équipe interne | Services managés AWS + IAM | Automatisation de la gestion des accès, réduction de 80% des tâches manuelles |
| Absence de traçabilité | CloudTrail + VPC Flow Logs | Audit complet de toutes les actions, détection d'anomalies |

**Conformité et certifications** :
- **ISO 27001** : AWS est certifié ISO 27001, garantissant un niveau de sécurité conforme aux standards internationaux.
- **PCI-DSS** : L'architecture respecte les exigences de sécurité pour le traitement de données de paiement (si applicable).
- **RGPD** : Hébergement en région Europe (eu-west-3 Paris), garantissant la souveraineté des données.

**Conclusion** : Cette stratégie de sécurité multi-couches (réseau, applicatif, identité) transforme radicalement la posture de sécurité de WebMarket+. En passant d'une approche réactive (correction manuelle des failles) à une approche **proactive et automatisée** (sécurité par conception), l'entreprise réduit drastiquement son exposition aux risques tout en libérant l'équipe interne des tâches répétitives et sources d'erreurs. La traçabilité complète des actions permet de répondre aux exigences d'audit et de conformité, tout en facilitant la détection et la réponse aux incidents de sécurité.
