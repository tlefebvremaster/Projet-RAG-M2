# Spécifications — Suivi Airbnb (n8n)

## 1. Objectif

Surveiller Airbnb pour un usage de location de villas (Cannes, saison haute), sans service payant, via deux automatisations n8n indépendantes :

- **A. Suivi Prix & Disponibilité** — pour une liste de biens déjà identifiés.
- **B. Détection de Nouvelles Annonces** — pour des biens pas encore identifiés, correspondant à des critères.

Contraintes de départ : solution 100% gratuite (pas d'API payante type Apify/RapidAPI), stack = n8n auto-hébergé + Google Sheets + Telegram.

## 2. Sources de données (validées empiriquement le 28/09/2026)

Airbnb n'a pas d'API publique officielle. Deux mécanismes internes ont été identifiés et testés avec succès :

### 2.1 Prix + disponibilité d'une annonce précise

- **Endpoint** : `GET https://www.airbnb.com/api/v3/StaysPdpBookItQuery/<hash>`
- **Type** : requête GraphQL à "persisted query" (le corps de la requête est remplacé par un hash SHA-256 côté serveur).
- **Hash validé** : `45cc0ada54da798665ac40417178a67be1378ca94275d37a2eca61ac1fb2cf7d` — **sujet à changement** lors des déploiements Airbnb (voir §7).
- **Authentification** : header `X-Airbnb-Api-Key: d306zoyjsyarp7ifhu67rjxn52tv0t20` — clé publique utilisée par le site lui-même pour tout utilisateur non connecté, stable, non secrète. Pas de session/cookie requis.
- **Paramètres clés** :
  - `id` : base64 de `DemandStayListing:<id_numérique_annonce>`
  - `dateRange` : `{startDate, endDate}` (format `YYYY-MM-DD`)
  - `guestCounts` : `{numberOfAdults}`
  - flags `include*Fragment` à `true` pour obtenir le détail de prix
- **Réponse exploitable** (`data.node.pdpPresentation.bookIt`) :
  - `availability.isAvailable` (booléen)
  - `availability.unavailabilityMessage` (ex. "Minimum stay is 5 nights", "Those dates are not available")
  - `structuredDisplayPrice.primaryLine.price` (ex. `"€18,130"`)
  - `structuredDisplayPrice.explanationData` (détail nuit/taxes)

### 2.2 Recherche / découverte de nouvelles annonces

- **Endpoint** : `GET https://www.airbnb.com/s/<lieu>/homes?checkin=...&checkout=...&adults=...&price_min=...&price_max=...&room_types[]=Entire home%2Fapt&items_offset=...`
- **Type** : page HTML publique, **pas d'API key requise**. Les résultats sont rendus côté serveur et embarqués dans un `<script data-deferred-state ...>` en JSON.
- **Chemin JSON** : `niobeClientData[0][1].data.presentation.staysSearch.results.searchResults` (tableau)
- **Par résultat** :
  - `demandStayListing.id` → base64 de `DemandStayListing:<id_numérique>` (même format que §2.1)
  - `nameLocalized.localizedStringWithTranslationPreference` ou `title` → nom de l'annonce
  - `structuredDisplayPrice.primaryLine.price` → prix total affiché pour les dates demandées
  - `price_min`/`price_max` de l'URL filtrent le prix **par nuit**
- Plus robuste dans le temps que §2.1 car pas de hash de requête à maintenir — seule la structure du JSON embarqué peut évoluer.

## 3. Modèle de données (Google Sheets)

### Onglet `Tracked` (entrée du workflow A)
| Colonne | Type | Description |
|---|---|---|
| listing_id | texte | ID numérique Airbnb (ex. `51090087`) |
| name | texte | Nom lisible du bien |
| url | texte | `https://www.airbnb.com/rooms/<listing_id>` |
| check_in / check_out | date `YYYY-MM-DD` | Dates de séjour à surveiller pour ce bien |
| guests | nombre | Nombre d'adultes |
| last_price_eur | nombre | Dernier prix total connu (€) — mis à jour par le workflow |
| last_available | booléen | Dernière disponibilité connue — mis à jour par le workflow |
| last_checked | datetime ISO | Horodatage du dernier contrôle |

Pré-rempli avec les 18 villas déjà présentes dans les dossiers `villas 10-15K` et `Villa 15-25K` (voir `tracked-listings-template.csv`).

### Onglet `History` (sortie du workflow A, append-only)
`checked_at, listing_id, name, price_eur, available, unavailability_message`
→ conserve un relevé à chaque exécution, pour reconstituer l'évolution des prix dans le temps.

### Onglet `Seen` (sortie du workflow B, append-only)
`listing_id, name, price_eur, url, first_seen`
→ sert de mémoire de déduplication : une annonce n'est notifiée qu'une seule fois.

## 4. Workflow A — Suivi Prix & Disponibilité

**Déclenchement** : planifié, toutes les 6h (ajustable).

**Logique, pour chaque ligne de `Tracked`** :
1. Appeler l'endpoint §2.1 avec `listing_id`, `check_in`, `check_out`, `guests` de la ligne.
2. Extraire `current_price_eur`, `current_available`, `unavailability_message`.
3. Comparer à `last_price_eur` / `last_available` :
   - `price_changed` = vrai si prix actuel ≠ dernier prix connu (et les deux sont non nuls)
   - `availability_changed` = vrai si disponibilité actuelle ≠ dernière connue
   - Aucune alerte au tout premier contrôle (pas de valeur de référence)
4. Si changement détecté → notification Telegram avec : nom du bien, ancien prix → nouveau prix (si applicable), nouveau statut de disponibilité (si applicable), dates, lien.
5. Dans tous les cas : append d'une ligne dans `History`, puis mise à jour de `last_price_eur` / `last_available` / `last_checked` dans `Tracked` (upsert par `listing_id`).

**Gestion des erreurs** : si l'appel échoue (annonce supprimée, hash de requête expiré, erreur réseau), consigner l'erreur sans faire planter le workflow pour les autres lignes ; ne pas déclencher d'alerte sur une erreur.

**Politesse envers Airbnb** : traitement séquentiel des biens avec une pause d'environ 1,5s entre chaque appel (pas de parallélisation), pour limiter le risque de blocage.

## 5. Workflow B — Détection de Nouvelles Annonces

**Déclenchement** : planifié, toutes les 6h (ajustable, décalé de A pour lisser la charge).

**Critères configurables** (un seul jeu de critères actif à la fois dans cette version) :
- `location_slug` (ex. `Cannes--France`)
- `check_in` / `check_out`
- `guests`
- `price_min` / `price_max` (par nuit, en €)
- Type de bien : logement entier (`Entire home/apt`)
- `pages` : nombre de pages de résultats à parcourir (~18 annonces/page)

**Logique** :
1. Construire l'URL de recherche §2.2 avec les critères, pour chaque page jusqu'à `pages`.
2. Parser chaque page, extraire la liste des annonces trouvées (id, nom, prix, url).
3. Charger la liste des `listing_id` déjà présents dans l'onglet `Seen`.
4. Ne garder que les annonces dont l'id n'est pas dans `Seen` → "nouvelles annonces".
5. Pour chaque nouvelle annonce : notification Telegram (nom, prix, dates, lien) puis append dans `Seen`.

**Hors périmètre de cette version** :
- Un seul jeu de critères par workflow (pour plusieurs lieux/budgets, dupliquer le workflow ou paramétrer plusieurs lignes de critères + boucle).
- Pas de re-vérification de disponibilité pour les nouvelles annonces trouvées (le prix affiché en recherche fait foi au moment du scan).
- Pas de gestion multi-devise (tout en EUR).

## 6. Notifications

Canal : Telegram (bot dédié, gratuit).

- **Workflow A** : un message par changement détecté (pas de digest groupé dans cette version).
- **Workflow B** : un message par nouvelle annonce détectée.

## 7. Risques & maintenance

| Risque | Impact | Mitigation |
|---|---|---|
| Airbnb change le hash de la requête GraphQL (§2.1) | Workflow A renvoie des erreurs | Récupérer le nouveau hash via DevTools (Network → filtrer `StaysPdpBookItQuery`) et le remplacer |
| Airbnb change la structure du JSON de résultats de recherche (§2.2) | Workflow B ne trouve plus rien | Réinspecter le HTML, ajuster le chemin `niobeClientData...` |
| Blocage temporaire d'IP si appels trop fréquents/parallèles | Échecs en rafale | Respecter l'intervalle de 6h, traitement séquentiel avec pause, éviter le multi-workflow simultané sur la même IP |
| Annonce supprimée/désactivée | Erreur ponctuelle sur une ligne `Tracked` | Ne pas bloquer les autres lignes ; à retirer manuellement de `Tracked` |

## 8. Hors périmètre général (non spécifié ici)

- Authentification Airbnb (compte connecté) — tout reste en mode visiteur non connecté.
- Interface de gestion des critères/biens autre que l'édition directe des Google Sheets.
- Historisation au-delà de l'append brut dans `History` (pas de dashboard/graphique inclus).
