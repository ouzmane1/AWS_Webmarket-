# Architecture Cible AWS : WebMarket+

Ce document définit les choix technologiques pour la migration de l'application WebMarket+ vers AWS, visant à résoudre les problèmes de scalabilité et de sécurité actuels.

---

## 1. Réseau (Networking & Security)
*L'isolation est la priorité pour répondre aux risques d'exposition réseau identifiés.*

- **VPC (Virtual Private Cloud)** : Création d'un réseau isolé sur la région Paris (eu-west-3).
- **Sous-réseaux (Subnets)** :
    - **Publics** : Hébergent l'Application Load Balancer (ALB) et la NAT Gateway.
    - **Privés** : Hébergent les instances d'application et la base de données (aucune IP publique).
- **Internet Gateway** : Pour permettre l'accès sortant via la NAT Gateway (mises à jour logicielles).
- **Security Groups (Firewall)** : 
    - ALB : Autorise le port 80/443 depuis 0.0.0.0/0.
    - App : Autorise le trafic venant uniquement de l'ALB.
    - DB : Autorise le trafic venant uniquement des instances App.

---

## 2. Services de Calcul (Compute)
*Réponse au besoin de scalabilité lors des pics de charge (soldes).*

- **Amazon EC2 avec Auto Scaling Group (ASG)** : 
    - Type d'instance : Famille **t3.medium** (équilibré) ou **c5** (si calcul intensif).
    - Stratégie : Scaling dynamique basé sur l'utilisation CPU.
- **Application Load Balancer (ALB)** : Répartit intelligemment le trafic entre les instances saines.

---

## 3. Gestion des Données (Storage & Data)
*Réponse au besoin de sauvegarde et de résilience.*

- **Base de données : Amazon RDS (Multi-AZ)** :
    - Moteur : MySQL ou PostgreSQL (selon l'app actuelle).
    - **Multi-AZ** : Réplication synchrone sur une seconde zone de disponibilité pour une haute disponibilité immédiate en cas de panne.
- **Stockage d'objets : Amazon S3** :
    - Pour les images produits et les documents.
    - Durable à 99.999999999% et permet de sortir les fichiers statiques du serveur d'appli.

---

## 4. Identité et Accès (IAM)
*Réponse au manque de traçabilité et de contrôle d'accès.*

- **IAM Roles** : Les instances EC2 utilisent des rôles pour accéder à S3 sans stocker de clés secrètes dans le code.
- **IAM Policies** : Application du principe du "moindre privilège" pour l'équipe interne d'exploitation.

---

## 5. Arbitrages & Optimisation (FinOps)
- **Monitoring** : Utilisation d'**Amazon CloudWatch** pour la visibilité des coûts et des performances.
- **Stratégie d'achat** : 
    - Instances à la demande pour la flexibilité initiale.
    - Instances Réservées (RI) après 3 mois d'observation pour réduire les coûts de 30 à 60%.