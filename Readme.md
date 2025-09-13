## Codebook et documentation d'analyse — Jeu de données commandes retail

### Contexte et objectifs
Ce dépôt contient un jeu de données de commandes (`orders.csv`) et un notebook Python (`PythonSql.ipynb`). L'objectif est de préparer les données, calculer des indicateurs clés (chiffre d'affaires, remises, profit), et analyser l'évolution des ventes, notamment la croissance mensuelle pour 2022 et 2023, ainsi que des découpes par segment, région, catégorie et produits.

### Fichiers
- `orders.csv`: données brutes des commandes.
- `PythonSql.ipynb`: notebook de préparation, calculs et visualisations.
- `retail-orders.zip`: archive d'origine.

---

## Codebook (dictionnaire de données)

Les noms ci-dessous se réfèrent aux colonnes brutes de `orders.csv`. Les types sont ceux attendus après normalisation.

### Colonnes brutes

- **`Order Id`**: entier. Identifiant unique de la ligne de commande. Exemple: `1`.
- **`Order Date`**: date (YYYY-MM-DD). Date de la commande. Exemple: `2023-03-01`.
- **`Ship Mode`**: chaîne. Mode d'expédition. Valeurs possibles : `Standard Class`, `Second Class`, `First Class`, `Not Available`, `unknown`.
- **`Segment`**: chaîne. Segment client. Exemples: `Consumer`, `Corporate`, `Home Office`.
- **`Country`**: chaîne. Pays (dans ce jeu: `United States`).
- **`City`**: chaîne. Ville de livraison.
- **`State`**: chaîne. État de livraison.
- **`Postal Code`**: chaîne (converti depuis entier). Code postal. Exemple: `90032`.
- **`Region`**: chaîne. Région commerciale. Exemples: `West`, `East`, `South`, `Central`.
- **`Category`**: chaîne. Grande catégorie produit. Exemples: `Furniture`, `Office Supplies`, `Technology`.
- **`Sub Category`**: chaîne. Sous-catégorie. Exemples: `Chairs`, `Binders`, `Phones`, `Art`, `Tables`, etc.
- **`Product Id`**: chaîne. Identifiant produit (ex: `FUR-BO-10001798`).
- **`cost price`**: nombre (float). Prix coûtant unitaire.
- **`List Price`**: nombre (float). Prix catalogue unitaire.
- **`Quantity`**: entier. Nombre d'unités commandées.
- **`Discount Percent`**: nombre (float). Pourcentage de remise unitaire, 0–100.

### Colonnes dérivées (créées pour l'analyse)

- **`discount`**: montant total de remise de la ligne.
  - Formule: `discount = list_price * (discount_percent/100) * quantity`
- **`sale_price`**: chiffre d'affaires de la ligne après remise.
  - Formule: `sale_price = (list_price - list_price * (discount_percent/100)) * quantity`
- **`profit`**: marge de la ligne.
  - Formule: `profit = sale_price - (cost_price * quantity)`
- **`year`**: année de `order_date`.
- **`month`**: mois numérique (1–12) ou libellé (Jan, Feb, ...), dérivé de `order_date`.

Note: Après création des colonnes `discount`, `sale_price`, `profit`, les colonnes `cost_price`, `list_price`, `discount_percent` ont été retirées de la version de travail car non nécessaires aux visualisations finales (elles restent documentées ici pour la traçabilité des calculs).

---

## Préparation des données

1. Normaliser les noms de colonnes: minuscules, espaces → `_`.
2. Convertir les types:
   - `order_date` → datetime (format `%Y-%m-%d`).
   - `postal_code` → chaîne (préserver les zéros initiaux).
3. Gérer les valeurs manquantes/inconnues:
   - `ship_mode` peut contenir `Not Available` / `unknown`. Les conserver comme catégorie explicite ou imputer si besoin.
4. Contrôles de qualité:
   - Vérifier duplicats potentiels sur `order_id` (ou clé composite si un `order_id` peut contenir plusieurs lignes). 
   - Repérer `list_price = 0` ou `cost_price = 0` et valider qu'il ne s'agit pas d'anomalies.
   - `discount_percent` hors [0, 100] → invalide.
5. Créer les colonnes dérivées: `discount`, `sale_price`, `profit`, `year`, `month`.
6. Optionnel: supprimer `cost_price`, `list_price`, `discount_percent` une fois les dérivées calculées.

Exemple (pandas):

```python
import pandas as pd

df = pd.read_csv("orders.csv")
df.columns = (
    df.columns.str.strip()
             .str.lower()
             .str.replace(" ", "_", regex=False)
)
df["order_date"] = pd.to_datetime(df["order_date"], format="%Y-%m-%d", errors="coerce")
df["postal_code"] = df["postal_code"].astype(str)

df["discount"] = df["list_price"] * (df["discount_percent"] / 100.0) * df["quantity"]
df["sale_price"] = (df["list_price"] - df["list_price"] * (df["discount_percent"] / 100.0)) * df["quantity"]
df["profit"] = df["sale_price"] - (df["cost_price"] * df["quantity"])

df["year"] = df["order_date"].dt.year
df["month"] = df["order_date"].dt.month
```

---

## Analyses principales

### 1) Croissance mensuelle du CA (2022–2023)
- Regrouper par `year`, `month` et sommer `sale_price`.
- Calculer la croissance MoM: `(CA_t - CA_{t-1}) / CA_{t-1}`.

Exemple (pandas):

```python
sales_m = (
    df.dropna(subset=["year", "month"])
      .groupby(["year", "month"], as_index=False)["sale_price"].sum()
      .sort_values(["year", "month"])
)
sales_m["mom_growth"] = sales_m.groupby("year")["sale_price"].pct_change()
```

Visualisation conseillée: courbes par mois, une série par année.

### 2) Performance par segment / région / catégorie
- Agréger `sale_price` et `profit` par `segment`, `region`, `category`, `sub_category`.
- Indicateurs: CA, marge, taux de remise moyen (`discount / (discount + sale_price)`), contribution (%) au total.

### 3) Produits et paniers
- Top produits par CA et par marge (`product_id`).
- Prix moyens, quantités moyennes par commande.

---

## Indicateurs recommandés

- **Chiffre d'affaires (CA)**: somme de `sale_price`.
- **Marge**: somme de `profit`; **taux de marge** = `profit / sale_price`.
- **Remise totale**: somme de `discount`; **taux de remise** ≈ `discount / (discount + sale_price)`.
- **Panier moyen**: `sale_price` moyen par ligne (ou par commande si clé disponible).

---

## Visualisations

- Courbe CA mensuel par année (2022 vs 2023).
- Barres empilées par `region` / `segment` (CA et marge).
- Pareto des `sub_category` par CA.
- Scatter `discount_percent` vs `profit` (détection d'élasticité / effets de remise).

---

## Reproductibilité

1. Ouvrir `PythonSql.ipynb`.
2. Exécuter les cellules de préparation (normalisation, conversions, dérivées).
3. Exécuter les cellules d'agrégation et de visualisation.

Pré-requis Python (exemple minimal):

```bash
pip install pandas numpy matplotlib seaborn
```

---

## Hypothèses et limites

- Les prix `cost_price` et `list_price` sont unitaires; `quantity` multiplie les montants.
- `Discount Percent` est un pourcentage (0–100) appliqué sur `list_price` unitaire.
- Les valeurs `0` sur les prix doivent être validées métier (promotions, échantillons, données manquantes).
- Le pays est unique (`United States`) dans cet échantillon; l'analyse géographique se fait au niveau `state` / `region`.

---

## Questions auxquelles répondre

- Comment évolue le CA mois par mois en 2022 et 2023 ?
- Quels segments/régions/catégories tirent le plus de CA et de marge ?
- Quelles sous-catégories et produits sont les plus rentables ?
- Quel est l'impact moyen des remises sur la marge ?

