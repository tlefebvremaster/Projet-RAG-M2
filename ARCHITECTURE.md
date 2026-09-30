# Architecture technique | Suivi & Veille Airbnb — Villas Cannes

Traduction technique du [cahier des charges](CAHIER-DES-CHARGES.md). Deux workflows n8n indépendants, mêmes briques.

## 1. Vue d'ensemble

```
                    ┌─────────────────────┐
                    │  Déclencheur n8n     │   1×/jour (les deux workflows)
                    │  (Schedule Trigger)  │
                    └──────────┬───────────┘
                               │
        ┌──────────────────────┴──────────────────────┐
        │                                              │
┌───────▼────────┐                          ┌──────────▼─────────┐
│ WORKFLOW A      │                          │ WORKFLOW B          │
│ Suivi des biens │                          │ Veille nouveautés   │
│ déjà identifiés │                          │                     │
└───────┬────────┘                          └──────────┬──────────┘
        │ lit                                            │ lit critères
┌───────▼────────┐                          ┌──────────▼──────────┐
│ Google Sheets   │                          │ Google Sheets        │
│ onglet Tracked  │                          │ onglet Seen (déjà vu)│
└───────┬────────┘                          └──────────┬──────────┘
        │                                                │
┌───────▼──────────────────────┐          ┌─────────────▼────────────────┐
│ Airbnb — endpoint prix/dispo  │          │ Airbnb — page de recherche    │
│ (par annonce, par fenêtre     │          │ publique (filtrée lieu/dates/ │
│  de dates)                    │          │  budget/type de bien)         │
└───────┬──────────────────────┘          └─────────────┬────────────────┘
        │ compare à la dernière valeur connue             │ compare à Seen
┌───────▼────────┐                          ┌─────────────▼────────────┐
│ Changement /    │                          │ Nouvelle annonce trouvée  │
│ échec détecté ? │                          └─────────────┬────────────┘
└───────┬────────┘                                          │
        │ oui                                                │ oui
┌───────▼────────┐                          ┌─────────────▼────────────┐
│ Slack (alerte)  │                          │ Slack (alerte)            │
└───────┬────────┘                          └─────────────┬────────────┘
        │                                                  │
┌───────▼────────┐                          ┌─────────────▼────────────┐
│ MAJ Tracked +   │                          │ Ajout à Seen              │
│ History         │                          └───────────────────────────┘
└─────────────────┘
```

## 2. Traçabilité cahier des charges → choix techniques

| Exigence (cahier des charges) | Choix technique |
|---|---|
| Vérification automatique 1×/jour | `Schedule Trigger` n8n, intervalle 24h, sur les deux workflows |
| Notification Slack courte | Nœud Slack natif n8n, message texte formaté (pas de bloc/fichier) |
| Contenu : nom+lien, prix avant/après, dates, disponibilité | Champs assemblés dans un nœud de code avant l'envoi Slack |
| Gestion des biens/critères dans Google Sheets | 3 onglets : `Tracked`, `History`, `Seen` (détail §4) |
| Lieu = Cannes | Paramètre `location_slug = Cannes--France` |
| Dates = 13→20 mai 2027, ±2 jours | 3 fenêtres testées par bien/recherche : 11→18, 13→20, 15→22 mai (durée de séjour fixe à 7 nuits, fenêtre décalée) |
| Budget = 10 000–25 000 € **au total** | Conversion en prix/nuit pour le filtre Airbnb (10000÷7 ≈ 1400€/nuit à 25000÷7 ≈ 3600€/nuit), **puis** re-vérification stricte sur le prix total réellement affiché par Airbnb pour la fenêtre retenue |
| Type de bien = villa avec piscine | Filtres natifs Airbnb validés : `l2_property_type_ids[]=1` (Maison) + `amenities[]=7` (Piscine) + `room_types[]=Entire home/apt` |
| ~15-20 biens suivis + 1 recherche | Une ligne `Tracked` par bien ; un seul jeu de critères dans Workflow B |
| Budget d'exécution 0€ | n8n auto-hébergé (Docker gratuit), Google Sheets gratuit, Slack plan gratuit, endpoints Airbnb publics sans clé payante |
| Alerte uniquement sur échecs répétés | Compteur `consecutive_errors` par bien dans `Tracked`, alerte Slack déclenchée au 3ᵉ échec consécutif seulement |
| Aucune contrainte de confidentialité | Pas de chiffrement/restriction d'accès spécifique au-delà du partage normal Google/Slack |

## 3. Logique détaillée

### Workflow A — Suivi des biens identifiés

Pour chaque ligne de `Tracked` (une par bien) :

1. Pour chacune des 3 fenêtres de dates (11→18, 13→20, 15→22 mai) :
   - interroger l'endpoint Airbnb de prix/disponibilité pour ce bien et cette fenêtre ;
   - attendre ~1,5s avant la fenêtre suivante (limiter la fréquence des appels).
2. Retenir la **meilleure option disponible** (prix le plus bas parmi les fenêtres où le bien est disponible). Si aucune fenêtre n'est disponible, retenir le message d'indisponibilité.
3. Comparer ce résultat au dernier relevé connu (`last_price_eur`, `last_available`) :
   - changement de prix et/ou de disponibilité détecté → alerte Slack.
4. Si l'appel a échoué (erreur réseau, bien supprimé, etc.) : incrémenter `consecutive_errors` ; à la 3ᵉ fois consécutive → alerte Slack dédiée ("à vérifier manuellement"). Une erreur isolée reste silencieuse. Un succès remet le compteur à 0.
5. Dans tous les cas : ajouter une ligne dans `History` ; mettre à jour `Tracked` (prix/dispo seulement si pas d'erreur, pour ne jamais écraser la dernière valeur connue par une erreur).

### Workflow B — Veille de nouvelles annonces

1. Charger les critères (lieu, dates de base, budget total, type de bien).
2. Pour chacune des 3 fenêtres de dates, et pour chaque page de résultats configurée :
   - interroger la page de recherche Airbnb avec les filtres lieu/dates/prix-par-nuit/type-de-bien/piscine ;
   - extraire les annonces trouvées (id, nom, prix total affiché, lien).
3. Filtrer strictement sur le budget total (10 000–25 000 €) — le filtre Airbnb étant par nuit, cette étape élimine les faux positifs dus à l'approximation.
4. Dédupliquer : une même annonce peut apparaître sur plusieurs fenêtres/pages → ne garder que son occurrence la moins chère.
5. Comparer à la liste `Seen` : ne garder que les annonces jamais notifiées.
6. Pour chaque nouvelle annonce : alerte Slack, puis ajout à `Seen`.

## 4. Modèle de données Google Sheets

### `Tracked`
`listing_id, name, url, check_in, check_out, guests, flex_days, last_price_eur, last_available, last_checked, consecutive_errors`

### `History` (append-only, un relevé par bien et par jour)
`checked_at, listing_id, name, price_eur, available, matched_check_in, matched_check_out`

### `Seen` (append-only, mémoire de déduplication)
`listing_id, name, price_eur, url, first_seen`

## 5. Format des notifications Slack

**Changement détecté (Workflow A)**
```
🔔 Villa Rivieras
💰 Prix : 12 400€ → 13 100€
📅 Disponibilité : devenu indisponible ❌
📆 2027-05-11 → 2027-05-18
🔗 https://www.airbnb.com/rooms/13110275
```

**Échec répété (Workflow A)**
```
⚠️ Villa Rivieras
Échec de vérification 3 fois de suite — à vérifier manuellement.
🔗 https://www.airbnb.com/rooms/13110275
```

**Nouvelle annonce (Workflow B)**
```
🆕 Villa avec piscine vue mer, Super Cannes
💰 €14,750 total
📆 2027-05-15 → 2027-05-22
🔗 https://www.airbnb.com/rooms/1234567890123456789
```

## 6. Risques & limites techniques (inchangés depuis les specs précédentes)

- Airbnb peut faire évoluer son endpoint interne ou la structure de sa page de résultats sans préavis → maintenance ponctuelle possible (voir README).
- Tripler les fenêtres de dates (±2j) triple le nombre d'appels par bien/page — reste largement dans les limites d'un usage gratuit et raisonnable à 1 exécution/jour.
- Le filtre `l2_property_type_ids[]=1` correspond à la catégorie Airbnb "Maison", qui englobe les villas mais aussi d'autres types de maisons individuelles ; il n'existe pas de filtre "Villa" strictement dédié côté Airbnb.
