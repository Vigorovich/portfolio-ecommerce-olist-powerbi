# Analyse e-commerce Olist — Power BI

Rapport Power BI de portfolio construit sur le dataset public **Olist** (marketplace e-commerce brésilienne) : ~99 400 commandes passées entre septembre 2016 et octobre 2018, avec clients, produits, vendeurs, paiements et avis.

## Question traitée

« Comment se portent nos ventes et notre logistique, et quel est leur impact sur la satisfaction client ? »

Storytelling du rapport (4 pages) : combien on vend → comment se passe la livraison → le client est-il content. Angle volontairement différent du projet Gironde, pour démontrer d'autres compétences (modélisation multi-tables, DAX, analyse logistique).

**Hors périmètre, volontairement :** ce rapport ne répond pas à "quels clients ou produits rapportent le plus" — aucun visuel de CA par client, et le CA par produit ne descend pas sous le niveau catégorie (le dataset ne contient pas de nom de produit, voir Limites connues). La mesure `Nb clients uniques` existe dans le modèle mais reste orpheline (non utilisée dans un visuel). C'est une limite assumée plutôt qu'un oubli : à ouvrir en piste d'évolution (ex. table "Top clients par CA net", directement exploitable puisque `Fact_Orders ↔ Dim_Customers` est déjà bidirectionnelle).

**Rapport interactif en ligne :** https://app.powerbi.com/view?r=eyJrIjoiMTMwMDkyZjQtZjUwYi00YjJmLWIyZDItNTcyMjA4MWY1MGQ2IiwidCI6ImU1ZDE1MDY5LTQxYTItNDhiZS1hM2YzLWQ3ZjUyZGIxNjQyNSIsImMiOjh9&pageName=3fb0a0b572a2e8a79337

Le fichier `.pbix` source est disponible dans l'onglet [**Releases**](https://github.com/Vigorovich/portfolio-ecommerce-olist-powerbi/releases/tag/v1.0.0) de ce repo (trop volumineux pour l'arborescence de fichiers standard de GitHub). Ce README documente la construction technique du modèle (relations, mesures DAX, logique des visuels), pour quiconque veut explorer la fabrication du rapport plutôt que seulement son rendu.

## Le jeu de données

8 fichiers CSV (`olist_ecommerce_dataset.zip`, source Kaggle) :

| Table | Grain | Lignes | Rôle |
|---|---|---|---|
| `olist_customers_dataset` | 1 client | 99 441 | Dimension client |
| `olist_orders_dataset` | 1 commande | 99 441 | Fait commande (statut, dates) |
| `olist_order_items_dataset` | 1 article commandé | 112 650 | Fait le plus fin (prix, port) |
| `olist_order_payments_dataset` | 1 paiement | 103 886 | Fait paiement |
| `olist_order_reviews_dataset` | 1 avis | 99 224 | Fait avis client |
| `olist_products_dataset` | 1 produit | 32 951 | Dimension produit |
| `olist_sellers_dataset` | 1 vendeur | 3 095 | Dimension vendeur |
| `product_category_name_translation` | 1 catégorie | 71 | Traduction PT → EN des catégories |

En complément, une table **`Dim_EtatsBresil`** (27 lignes, construite manuellement — ne fait pas partie des CSV Kaggle) : `state_code` (2 lettres, ex. `SP`), `state_name` (nom complet, ex. `São Paulo`), `state_name_ascii` (sans accents), `latitude`, `longitude`. Elle traduit les codes d'état bruts (`customer_state`) en noms lisibles pour les tableaux, et fournit les coordonnées utilisées par les visuels carte.

## Modèle de données

Modèle en étoile, `Fact_Orders` au centre :

```
Dim_Customers (1) ──────────────► (*) Fact_Orders[customer_id]

Fact_Orders (1) ─────────────────► (*) Fact_OrdersItems[order_id]
Fact_Orders (1) ─────────────────► (*) Fact_OrderPayments[order_id]
Fact_Orders (1) ─────────────────► (*) Fact_OrderReviews[order_id]

Dim_Products (1) ────────────────► (*) Fact_OrdersItems[product_id]
Dim_Products[product_category_name] (1) ──► (*) Dim_CategoryTranslation[product_category_name]

Dim_Sellers (1) ──────────────────► (*) Fact_OrdersItems[seller_id]

Dim_Dates (1) ────────────────────► (*) Fact_Orders[order_purchase_date]

Dim_EtatsBresil[state_code] (1) ──► (*) Dim_Customers[customer_state]
```

**Sens de filtrage (point important du modèle) :** seules deux relations sont en filtrage croisé bidirectionnel ("Les deux") :
- `Fact_Orders` ↔ `Fact_OrdersItems`
- `Fact_Orders` ↔ `Dim_Customers`

Toutes les autres relations (`Fact_Orders → Fact_OrderPayments`, `Fact_Orders → Fact_OrderReviews`, `Dim_Sellers → Fact_OrdersItems`, `Dim_Products → Fact_OrdersItems`) restent à sens unique par défaut, même si elles sont actives — **"active" ne veut pas dire "bidirectionnelle"**, ce sont deux réglages distincts dans Power BI.

Conséquence concrète : une mesure calculée sur `Fact_Orders` (ex. les délais de livraison) ou sur `Fact_OrderReviews` (ex. les notes) ne se filtre pas automatiquement quand on la croise avec un champ qui ne vit que côté `Fact_OrdersItems` / `Dim_Products` / `Dim_Sellers` (catégorie de produit, vendeur) — la mesure renvoie alors la même valeur globale répétée pour chaque ligne. Ce dataset a été l'occasion de documenter et corriger ce piège avec des mesures-pont `TREATAS` (voir plus bas).

**Point d'attention (grain) :** une commande peut contenir plusieurs articles (`Fact_OrdersItems`) et parfois plusieurs paiements fractionnés (`Fact_OrderPayments`). Le CA est donc calculé à partir de `Fact_OrdersItems`, jamais en sommant `payment_value` directement, pour éviter les doublons.

## Mesures DAX

### Ventes
```DAX
CA net = SUMX(Fact_OrdersItems, Fact_OrdersItems[price])

Frais de port = SUM(Fact_OrdersItems[freight_value])

CA total (produits + port) = [CA net] + [Frais de port]

Nb commandes = DISTINCTCOUNT(Fact_Orders[order_id])

Nb clients uniques = DISTINCTCOUNT(Dim_Customers[customer_unique_id])

Panier moyen = DIVIDE([CA net], [Nb commandes])
```

### Statut des commandes
```DAX
Nb commandes livrées =
CALCULATE([Nb commandes], Fact_Orders[order_status] = "delivered")

Taux de livraison =
DIVIDE([Nb commandes livrées], [Nb commandes])

Nb commandes annulées =
CALCULATE([Nb commandes], Fact_Orders[order_status] = "canceled")

Taux d'annulation =
DIVIDE([Nb commandes annulées], [Nb commandes])
```

### Logistique
```DAX
Délai de livraison moyen (jours) =
AVERAGEX(
    FILTER(Fact_Orders, NOT ISBLANK(Fact_Orders[order_delivered_customer_date])),
    DATEDIFF(Fact_Orders[order_purchase_timestamp], Fact_Orders[order_delivered_customer_date], DAY)
)

Écart vs délai estimé (jours) =
AVERAGEX(
    FILTER(Fact_Orders, NOT ISBLANK(Fact_Orders[order_delivered_customer_date])),
    DATEDIFF(Fact_Orders[order_estimated_delivery_date], Fact_Orders[order_delivered_customer_date], DAY)
)
-- positif = livré en retard, négatif = livré en avance

Nb livraisons en retard =
CALCULATE(
    [Nb commandes],
    Fact_Orders[order_delivered_customer_date] > Fact_Orders[order_estimated_delivery_date]
)

Taux de retard =
DIVIDE([Nb livraisons en retard], [Nb commandes livrées])

Nb commandes en retard critique =
CALCULATE(
    [Nb commandes],
    NOT ISBLANK(Fact_Orders[Délai de livraison (jours)]),
    Fact_Orders[Délai de livraison (jours)] > 46
)
-- seuil = 99e centile du délai de livraison, cohérent avec la table "Commandes en alerte"

Taux de retard critique =
DIVIDE([Nb commandes en retard critique], [Nb commandes livrées])
```

**Mesure-pont pour ventiler le délai par catégorie ou par vendeur** (contourne le sens unique des relations, voir section Modèle) :
```DAX
Délai de livraison moyen (jours) =
VAR CommandesLivrees =
    FILTER(Fact_Orders, NOT ISBLANK(Fact_Orders[order_delivered_customer_date]))
VAR CommandesFiltrees =
    CALCULATETABLE(
        CommandesLivrees,
        TREATAS(VALUES(Fact_OrdersItems[order_id]), Fact_Orders[order_id])
    )
RETURN
AVERAGEX(
    CommandesFiltrees,
    DATEDIFF(Fact_Orders[order_purchase_timestamp], Fact_Orders[order_delivered_customer_date], DAY)
)
```

### Satisfaction
```DAX
Note moyenne = AVERAGE(Fact_OrderReviews[review_score])

Nb avis = DISTINCTCOUNT(Fact_OrderReviews[review_id])

Nb avis 1 étoile =
CALCULATE(DISTINCTCOUNT(Fact_OrderReviews[review_id]), Fact_OrderReviews[review_score] = 1)

% avis négatifs (≤2) =
DIVIDE(
    CALCULATE(DISTINCTCOUNT(Fact_OrderReviews[review_id]), Fact_OrderReviews[review_score] <= 2),
    DISTINCTCOUNT(Fact_OrderReviews[review_id])
)
```

**Mesures-pont à deux étages pour ventiler par vendeur** (`Dim_Sellers → Fact_OrdersItems → Fact_Orders → Fact_OrderReviews`) :
```DAX
Note moyenne par vendeur =
VAR CommandesDuVendeur = VALUES(Fact_OrdersItems[order_id])
RETURN
AVERAGEX(
    CALCULATETABLE(
        Fact_OrderReviews,
        TREATAS(CommandesDuVendeur, Fact_OrderReviews[order_id])
    ),
    Fact_OrderReviews[review_score]
)

Nb avis par vendeur =
VAR CommandesDuVendeur = VALUES(Fact_OrdersItems[order_id])
RETURN
CALCULATE(
    DISTINCTCOUNT(Fact_OrderReviews[review_id]),
    TREATAS(CommandesDuVendeur, Fact_OrderReviews[order_id])
)
```

**Top 10 vendeurs fiable** — ni le filtre natif "10 premiers" (instable combiné à une mesure-pont `TREATAS`), ni un tri manuel par en-tête (réordonne sans limiter le nombre de lignes) ne suffisent. Solution : un rang `RANKX` posé en filtre visuel (`<= 10`), avec un score composite pour départager les ex-æquo (fréquents sur de petits échantillons d'avis) en favorisant le vendeur avec le plus d'avis à note égale :
```DAX
Rang note moyenne (vendeur) =
IF(
    [Nb avis par vendeur] >= 10,
    RANKX(
        FILTER(ALLSELECTED(Dim_Sellers[seller_id]), [Nb avis par vendeur] >= 10),
        [Note moyenne par vendeur] * 10000000 + [Nb avis par vendeur],
        ,
        DESC,
        DENSE
    )
)
```

### Paiement *(mesures de réserve, non utilisées dans le rapport final — voir Bilan)*
```DAX
Montant moyen payé =
AVERAGEX(
    SUMMARIZE(Fact_OrderPayments, Fact_OrderPayments[order_id], "total", SUM(Fact_OrderPayments[payment_value])),
    [total]
)

Nb paiements en plusieurs fois =
CALCULATE(DISTINCTCOUNT(Fact_OrderPayments[order_id]), Fact_OrderPayments[payment_installments] > 1)

Versements moyens = AVERAGE(Fact_OrderPayments[payment_installments])
```

### Évolution temporelle
```DAX
CA N-1 (période comparable) = CALCULATE([CA net], SAMEPERIODLASTYEAR(Dim_Dates[Date]))

Évolution CA vs N-1 % = DIVIDE([CA net] - [CA N-1 (période comparable)], [CA N-1 (période comparable)])
```

### Colonnes calculées (sur `Fact_Orders`)

Power BI n'a pas de type "Histogramme" natif — distribution construite via des colonnes calculées :
```DAX
Délai de livraison (jours) =
IF(
    ISBLANK(Fact_Orders[order_delivered_customer_date]),
    BLANK(),
    DATEDIFF(Fact_Orders[order_purchase_timestamp], Fact_Orders[order_delivered_customer_date], DAY)
)

Tranche de délai =
SWITCH(
    TRUE(),
    ISBLANK(Fact_Orders[Délai de livraison (jours)]), BLANK(),
    Fact_Orders[Délai de livraison (jours)] <= 3, "0-3 jours",
    Fact_Orders[Délai de livraison (jours)] <= 7, "4-7 jours",
    Fact_Orders[Délai de livraison (jours)] <= 14, "8-14 jours",
    Fact_Orders[Délai de livraison (jours)] <= 30, "15-30 jours",
    "30+ jours"
)

Ordre tranche =
SWITCH(
    TRUE(),
    ISBLANK(Fact_Orders[Délai de livraison (jours)]), BLANK(),
    Fact_Orders[Délai de livraison (jours)] <= 3, 1,
    Fact_Orders[Délai de livraison (jours)] <= 7, 2,
    Fact_Orders[Délai de livraison (jours)] <= 14, 3,
    Fact_Orders[Délai de livraison (jours)] <= 30, 4,
    5
)
```
(`Tranche de délai` est triée par `Ordre tranche` via **Trier par colonne**, pour un affichage chronologique au lieu de l'ordre alphabétique.)

**Colonne distincte pour la table d'alerte :**
```DAX
Délai de livraison par commande (jours) =
DATEDIFF(Fact_Orders[order_purchase_date], Fact_Orders[order_delivered_customer_date], DAY)
```
Utilisée spécifiquement dans la table "Top 20 des commandes en alerte" (diffère de `Délai de livraison (jours)` par le champ de départ : `order_purchase_date` au lieu de `order_purchase_timestamp`, et pas de gestion explicite du `BLANK()`).

`DiagDelai` existe dans le modèle mais n'est plus utilisée dans le rapport final (itération de diagnostic conservée).

## Les 4 pages du rapport

1. **Vue d'ensemble** — 5 KPI : CA net, CA N-1, nb commandes, panier moyen, note moyenne. Courbe CA net vs CA N-1 par année et par trimestre ; carte du Brésil par état (CA net).
2. **Logistique & Livraison** — 6 KPI : délai de livraison moyen, écart vs délai estimé, nb commandes en retard critique, taux de retard, taux de retard critique, taux d'annulation. Courbe délai de livraison moyen par année et par trimestre ; graphe barre délai moyen par catégorie de produit (mesure-pont `TREATAS`) ; graphe barre nb commandes livrées par tranche de délai ; table taux de retard critique par état ; table top 20 des commandes en alerte.
3. **Produits & Catégories** — 3 KPI : CA net, nb commandes, panier moyen. Table top 10 vendeurs par CA net ; table top vendeurs par note moyenne avec nombre d'avis (mesures-pont à deux étages `Dim_Sellers → Fact_OrdersItems → Fact_Orders → Fact_OrderReviews`) ; graphe barre top 15 CA net par catégorie de produit.
4. **Satisfaction client** — 4 KPI : nb avis, note moyenne, % avis négatifs (≤2), nb avis 1 étoile. Graphe barre nb avis par note ; graphe barre % avis négatifs par tranche de délai de livraison (angle livraison ↔ satisfaction) ; table des commentaires clients.

## Bilan des mesures

30 mesures créées au total : 19 utilisées directement dans un visuel, 2 utilisées en interne (base d'une autre mesure ou filtre visuel), 9 orphelines (dont tout le dossier "Paiement", gardé en réserve pour une extension future du rapport). Détail dans le guide de construction complet.

## Limites connues

- Le dataset ne contient pas de nom de produit (seulement `product_id` + catégorie + dimensions physiques) : pas de "top produits nommés" possible, le niveau le plus fin exploitable reste la catégorie.
- Certains noms de ville de vendeur sont homonymes entre états brésiliens (ex. deux "Rio de Janeiro" dans des états différents) — géré en combinant ville + état dans une colonne dédiée pour éviter toute ambiguïté dans les tableaux.
- Le dataset couvre septembre 2016 à octobre 2018, avec un volume 2016 trop faible pour être comparé significativement à 2017/2018.

## Outils

Power BI Desktop (modélisation, DAX, visuels), Power Query (nettoyage).
