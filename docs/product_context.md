#  Contexte Projet : Migration Cloud WebMarket+

## Présentation de l'entreprise
**WebMarket+** est une PME qui édite une application web permettant aux commerçants de gérer :
* Le catalogue de produits.
* La gestion des stocks.

**État actuel** : Hébergement sur un serveur dédié "On-Premise" (locaux de l'entreprise).

---

##  Limites Identifiées (Pain Points)
1.  **Performance** : Incapacité à absorber les pics de charge (soldes, fêtes).
2.  **Finances** : Manque de visibilité sur les coûts réels d'infrastructure.
3.  **Sécurité** : Risques sur les sauvegardes, contrôles d'accès et exposition réseau.
4.  **RH/Exploitation** : Trop forte dépendance à une petite équipe interne.

---

##  Objectifs de la Migration (AWS)
* **Résilience** : Améliorer la disponibilité de l'application.
* **Scalabilité** : Adapter les ressources à la charge en temps réel.
* **Sécurité** : Renforcer la traçabilité et les contrôles (IAM).
* **Pilotage** : Maîtriser les coûts et obtenir des recommandations d'optimisation.

---

##  Détails de la Mission (Partie 1)

### 1.1 Conception de l'Architecture Cible
L'architecture doit inclure au minimum :
- **Calcul** : Hébergement de l'application web.
- **Données** : Base de données managée et stockage d'objets.
- **Réseau** : VPC, sous-réseaux, accès internet, règles de sécurité.
- **Identités** : Gestion des accès (IAM).

### 1.2 Justification et Arbitrages
Justifier les choix selon trois axes :
1.  **Métier** : Réponse aux besoins de performance et disponibilité.
2.  **FinOps** : Influence des services sur les coûts.
3.  **Stratégie SI** : Évolutivité, industrialisation et résilience.

---

##  Livrables Attendus
* **Schéma d'architecture** annoté.
* **Texte de justification** (1 à 2 pages) détaillant les arbitrages.