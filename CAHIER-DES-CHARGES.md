# Specs | Suivi & Veille Airbnb — Villas Cannes

## 1. Déclencheur & Livrable attendu

### 1.1 Déclencheur
Le processus doit s'exécuter **automatiquement, une fois par jour**, sans aucune action manuelle de l'utilisateur.

### 1.2 Ce que le processus doit surveiller
Deux besoins distincts, à couvrir tous les deux :
- **Suivi d'une liste de biens déjà identifiés** : détecter tout changement de prix ou de disponibilité pour des dates données.
- **Détection de nouveaux biens** : repérer les annonces qui n'existaient pas encore et qui correspondent à des critères de prix, de dates et de lieu définis par l'utilisateur.

### 1.3 Format du livrable
Une **notification instantanée et courte**, envoyée dès qu'un événement pertinent est détecté (pas de récapitulatif groupé, pas de tableau de bord séparé à consulter).

### 1.4 Contenu obligatoire de chaque notification
- Nom/titre du bien et lien direct vers l'annonce
- Ancien prix et nouveau prix (pour les changements de prix)
- Dates concernées (arrivée / départ)
- Statut de disponibilité (redevenu disponible / devenu indisponible)

### 1.5 Critères de recherche pour les nouveaux biens

| Critère | Valeur |
|---|---|
| Lieu | Cannes uniquement |
| Dates de séjour | 13 → 20 mai 2027 (semaine du Festival de Cannes, 80ᵉ édition), avec ±2 jours de flexibilité sur l'arrivée et le départ |
| Budget | 10 000 – 25 000 € au total pour le séjour |
| Type de bien | Villa avec piscine uniquement |

## 2. Outils & Écosystème

### 2.1 Outils imposés
- **Automatisation** : n8n (déjà choisi par l'utilisateur).
- **Canal de notification** : Slack.
- **Gestion de la liste des biens suivis et des critères de recherche** : Google Sheets, modifiable directement par l'utilisateur.

### 2.2 État des accès
- Compte Google : déjà disponible.
- Espace Slack : déjà disponible.
- Instance n8n : **pas encore installée/configurée** — à mettre en place.

### 2.3 Limites connues à prendre en compte
- Slack en **plan gratuit** : l'historique des messages consultable dans l'application est limité dans le temps. Les alertes anciennes pourraient donc ne plus être consultables directement dans Slack après un certain délai.

## 3. Contraintes opérationnelles & Gestion des échecs

### 3.1 Volume attendu
- Environ **15 à 20 biens** suivis en parallèle pour le suivi de prix/disponibilité.
- **1 seule recherche** de nouveaux biens active à la fois (un jeu de critères prix/dates/lieu).

### 3.2 Budget d'exécution
**0 € — strictement gratuit.** Aucune dépense n'est acceptée, y compris pour l'hébergement ou les outils tiers.

### 3.3 Comportement attendu en cas d'échec
Si l'accès à Airbnb (ou à un autre outil du processus) échoue en cours d'exécution :
- Une erreur isolée sur un bien ne doit pas bloquer le traitement des autres biens.
- L'utilisateur doit être **alerté uniquement en cas d'échecs répétés** sur un même bien (signe qu'une vérification manuelle est nécessaire) — pas d'alerte sur un échec ponctuel.

### 3.4 Confidentialité & sécurité
Aucune contrainte particulière : les données manipulées (prix et disponibilités d'annonces publiques) ne sont pas considérées comme sensibles, et aucune restriction d'accès spécifique n'est requise à ce stade.

---

## Synthèse

| Dimension | Réponse |
|---|---|
| Déclencheur | Automatique, 1×/jour |
| Sortie | Notification Slack instantanée et courte |
| Contenu notification | Nom + lien, ancien/nouveau prix, dates, disponibilité |
| Critères nouveaux biens | Cannes, 13→20 mai 2027 (±2j), 10-25k€, villa avec piscine |
| Outil d'automatisation | n8n (à installer) |
| Canal de notification | Slack (déjà disponible) |
| Gestion des biens/critères | Google Sheets (déjà disponible) |
| Volume | ~15-20 biens suivis + 1 recherche de nouveautés |
| Budget | 0 € |
| Gestion des erreurs | Ignorer les échecs isolés, alerter sur échecs répétés |
| Confidentialité | Aucune contrainte |
