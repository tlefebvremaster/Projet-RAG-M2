# Suivi Airbnb (gratuit) — n8n

Deux workflows n8n qui surveillent Airbnb sans API payante, conformes au [cahier des charges](CAHIER-DES-CHARGES.md) et à l'[architecture technique](ARCHITECTURE.md) :

1. **airbnb-suivi-prix-dispo.json** — suit une liste de biens précis (ceux de tes PDF villas Cannes) et alerte sur Slack quand le prix ou la disponibilité change, pour la semaine du 13→20 mai 2027 (±2 jours de flexibilité), ou sur 3 échecs de vérification consécutifs pour un bien.
2. **airbnb-nouvelles-annonces.json** — cherche chaque jour de nouvelles villas avec piscine à Cannes, pour cette même période, entre 10 000€ et 25 000€ au total, et alerte sur Slack pour toute nouveauté.

Les deux sont **100% gratuits** : n8n auto-hébergé, Google Sheets pour le stockage, Slack pour les notifications. Ils lisent l'API/le HTML publics d'Airbnb — pas de clé payante, pas d'abonnement à un service de scraping.

## ⚠️ À savoir avant de démarrer

Airbnb n'a pas d'API publique officielle et peut faire évoluer ses pages/son endpoint interne sans préavis. Ces workflows ont été testés et fonctionnent aujourd'hui (28/09/2026), mais :
- Si Airbnb change la structure de sa page de résultats de recherche, le nœud **Chercher Nouvelles Annonces** peut cesser de trouver des annonces (le code doit alors être ajusté).
- Si Airbnb change son `sha256Hash` de requête GraphQL (déploiement front-end), le nœud **Récupérer Prix & Dispo** renverra des erreurs jusqu'à ce que le hash soit mis à jour (voir section "Si ça casse" plus bas) — mais grâce à la règle "alerte sur 3 échecs consécutifs seulement", tu ne seras notifié qu'après 3 jours d'échec, pas dès le premier.
- Chaque bien/recherche est interrogé sur 3 fenêtres de dates (±2 jours), donc jusqu'à 3× plus d'appels qu'une vérification à date fixe — reste largement raisonnable à 1 exécution/jour.

## 1. Installer n8n (gratuit)

```bash
docker run -it --rm --name n8n -p 5678:5678 -v n8n_data:/home/node/.n8n docker.n8n.io/n8nio/n8n
```

Puis ouvre `http://localhost:5678`. (Alternative sans Docker : `npx n8n`.)

Pour que les workflows tournent en continu (pas seulement quand ton Mac est allumé), il faudra héberger n8n quelque part accessible 24/7.

## 2. Créer la Google Sheet

Crée une Google Sheet avec 3 onglets :

- **Tracked** — colonnes : `listing_id, name, url, check_in, check_out, guests, flex_days, last_price_eur, last_available, last_checked, consecutive_errors`
  → importe `tracked-listings-template.csv` pour démarrer avec les 18 villas Cannes déjà dans tes dossiers, dates réglées sur le 13→20 mai 2027 (±2j).
- **History** — colonnes : `checked_at, listing_id, name, price_eur, available, matched_check_in, matched_check_out`
- **Seen** — colonnes : `listing_id, name, price_eur, url, first_seen`

Récupère l'ID de la feuille dans son URL : `https://docs.google.com/spreadsheets/d/CET_ID_ICI/edit`.

## 3. Connecter Slack (gratuit)

1. Sur [api.slack.com/apps](https://api.slack.com/apps), crée une nouvelle app pour ton workspace.
2. Ajoute le scope `chat:write`, installe l'app sur le workspace.
3. Dans n8n, crée des identifiants Slack (API Token ou OAuth2 selon la méthode choisie).
4. Invite l'app dans le channel où tu veux recevoir les alertes.

## 4. Importer les workflows dans n8n

Dans n8n : **Workflows → Import from File** → sélectionne `airbnb-suivi-prix-dispo.json`, puis répète pour `airbnb-nouvelles-annonces.json`.

Pour chaque workflow :
- Sur chaque nœud **Google Sheets**, sélectionne/crée tes identifiants Google Sheets (OAuth2), remplace `REMPLACE_PAR_TON_ID_GOOGLE_SHEET` par l'ID de ta feuille.
- Sur le nœud **Alerte Slack**, sélectionne tes identifiants Slack et le channel de destination.
- Dans `airbnb-nouvelles-annonces.json`, le nœud **Critères de Recherche** est pré-rempli (Cannes, 13→20 mai 2027, ±2j, 10-25k€, villa+piscine) — ajuste si besoin.
- Active le workflow (toggle en haut à droite).

## 5. Tester

Clique sur "Execute Workflow" pour lancer une exécution manuelle et vérifier que :
- La lecture Google Sheets fonctionne.
- Le prix/la disponibilité sont bien récupérés.
- Un message Slack arrive bien (force un changement en modifiant à la main `last_price_eur` dans la sheet pour tester l'alerte).

## Si ça casse (maintenance)

Si le workflow **Suivi Prix & Dispo** renvoie des erreurs GraphQL :
1. Ouvre une annonce Airbnb dans Chrome, ouvre les DevTools → onglet Network, filtre sur `StaysPdpBookItQuery`.
2. Sélectionne des dates dans le widget de réservation de la page pour déclencher la requête.
3. Copie le nouveau `sha256Hash` depuis l'URL de la requête et remplace `QUERY_HASH` dans le nœud **Récupérer Prix & Dispo**.

Si le workflow **Nouvelles Annonces** ne trouve plus rien :
1. Ouvre une page de résultats de recherche Airbnb, récupère le HTML brut de la page.
2. Cherche `data-deferred-state` puis `staysSearch.results.searchResults` dans le JSON — si le chemin a changé, ajuste-le dans le nœud **Chercher Nouvelles Annonces**.
