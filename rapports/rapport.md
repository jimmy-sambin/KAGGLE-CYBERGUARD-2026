# Rapport — EGSPEC CYBERGUARD CODEFORGECLUB CSBS 2026 (épreuve rejouée)

> Dossier : `C:\A\kaggle-egspec-cyberguard-2026` · Analyse réalisée le 16/11/2026 · Compte Kaggle : `jsambin`

## 1. Synthèse

Hackathon d'une journée (26 sept. 2026) organisé par l'E.G.S Pillay Engineering College (Inde),
sur la **détection d'intrusions réseau par ML**. Description officielle : classification multiclasse
en 7 types d'attaques, métrique Macro F1. **La réalité des données est différente** (voir §3) :
cible binaire `Benign`/`Malicious`, signal concentré sur 2 variables, et un jeu de test
probablement 100 % bénin. L'épreuve rejouée atteint **Macro F1 OOF = 0,9934**.

## 2. Ce que disait l'annonce (vs ce que contiennent les données)

| Élément | Annonce officielle | Données réelles |
|---|---|---|
| Cible | 7 classes (`Normal`, `DoS`, `Probe`, `Brute_Force`, `Web_Attack`, `Botnet`, `Other`) | **2 classes** : `Benign` (70,1 %) / `Malicious` (29,9 %) |
| Métrique | Macro F1 sur 7 classes | Macro F1 (binaire) vraisemblablement |
| Valeurs manquantes | « some features may contain missing values » | **Aucune** |
| Catégorielles haute cardinalité | « challenge to solve » | **1 seule** : `Protocol` (TCP/UDP/ICMP) |
| `SAMPLE_DATA.csv` | « format de soumission » | Copie de 50 lignes du test (49 colonnes de features) — pas un format de soumission |
| Déséquilibre | « extreme imbalance » | Modéré : 70/30 |

## 3. Les données

- `TRAIN_DATA.csv` : 100 000 lignes × 50 colonnes (49 features + `Attack_Label`)
- `TEST_DATA.csv` : 25 000 lignes × 49 colonnes — `Flow_ID` continu (100 001 → 125 000)
- Features : stats de flux CIC-IDS-like (durée, paquets/octets fwd-bwd, flags TCP, IAT, actif/inactif, fenêtres…)

### Le générateur du train est parfaitement identifiable

| Zone | P(Malicious) |
|---|---|
| `SYN_Flag_Count ≤ 2` | **0 %** (13 906 + 14 032 + 14 091 lignes, toutes bénignes) |
| `SYN_Flag_Count ≥ 5` | **100 %** (~24 947 lignes, toutes malicieuses) |
| `SYN ∈ {3, 4}` & `FPPS > 5000` | **≈ 26 %** (zone ambiguë) |
| Aucun malicieux n'a `FPPS < 5000` | (0 / 29 853) |

Deux features font tout le travail : `SYN_Flag_Count` (corr. 0,80) et `Flow_Packets_Per_Sec` (corr. 0,81).
Toutes les autres ont une corrélation ≈ 0,005 avec la cible — le reste du dataset est du « bruit décoratif ».

### 🔍 Anomalie majeure : le test semble 100 % bénin

- `SYN_Flag_Count` du test ne prend **que les valeurs 0–4** (jamais ≥ 5)
- `Flow_Packets_Per_Sec` du test : max ≈ 9 999 (plage bénigne), 50 % > 5 000
- Le modèle (valide en OOF à 0,9934) donne sur le test : **probabilité max = 0,15**, moyenne ≈ 0,003
  → **0 ligne prédite malicieuse sur 25 000**
- Or sur un holdout du train, le même modèle détecte correctement ~14 % de malicieux dans la zone ambiguë
- Les features de la zone ambiguë sont quasi identiques entre train et test (pas de drift détectable)

**Conclusion (inférence forte)** : le jeu de test a été généré uniquement à partir de la distribution
bénigne. La « bonne » soumission pour le leaderboard était probablement `Benign` partout.

## 4. Méthode mise en œuvre

1. **Feature engineering réseau** : `bytes_per_packet`, ratios fwd/bwd, `syn_ratio`, `rst_ratio`,
   `flags_total`, `active_idle_ratio`, `iat_std_ratio`, port well-known, one-hot `Protocol`.
2. **Validation** : StratifiedKFold(5), seed 42 — scores par fold : 0,9930 / 0,9929 / 0,9933 / 0,9944 / 0,9936.
3. **Modèle** : XGBoost (`n_estimators=400, max_depth=8, lr=0.08`, `scale_pos_weight≈2.35`, hist).
4. **Référence « règle experte »** : `SYN ≥ 5 → Malicious` seul atteint déjà **Macro F1 = 0,9383**.

### Résultats OOF (100 000 lignes)

| | Précision | Rappel | F1 |
|---|---|---|---|
| Benign | 0,9923 | 1,0000 | 0,9961 |
| Malicious | 1,0000 | 0,9817 | 0,9908 |
| **Macro F1** | | | **0,9934** |

Matrice de confusion : 70 147 vrais bénins, 0 faux positifs, **547 malicieux ratés**
(tous dans la zone ambiguë `SYN ∈ {3,4}` — erreur quasi irréductible, le générateur les rendant
indiscernables des bénins sur les features disponibles).

Top features (gain) : `Flow_Packets_Per_Sec` 0,56 · `SYN_Flag_Count` 0,31 · `flags_total` 0,03 · `syn_ratio` 0,02.

## 4 bis. Comparaison de modèles et optimisation du seuil (16-17/11/2026)

Même cadre pour tous : StratifiedKFold(5), seed 42, mêmes features.

| Modèle | Macro F1 OOF (seuil 0,5) | Seuil optimal | Macro F1 optimal |
|---|---|---|---|
| Régression logistique | 0,9854 | 0,617 | 0,9885 |
| RandomForest (100 arbres) | 0,9934 | 0,348 | 0,9934 |
| **XGBoost** | **0,9934** | 0,214 | **0,9934** |
| MLP (64,32) | 0,9839 | 0,585 | 0,9843 |
| Règle experte `SYN ≥ 5` (référence) | 0,9383 | — | — |

**Enseignements du seuillage** : pour XGBoost et RF la courbe seuil→F1 est un *plateau* — les 547 faux négatifs sont statistiquement indiscernables (erreur irréductible), aucun seuil ne les récupère. Le seuillage rapporte +0,3 pt à la régression logistique et +0,04 pt au MLP, qui souffraient d'un déséquilibre mal calibré à 0,5. Le seuil optimal de XGBoost (0,214) est bas car la classe malicieuse est mieux séparée en probabilité.

**Courbe d'apprentissage** : Macro F1 = 0,9939 dès **25 arbres** sur holdout 20 % — plate jusqu'à 800. Le signal est trivial ; toute complexité supplémentaire n'apporte rien (différence RF/XGB ≈ 0).

**Confusion OOF** : XGBoost et RF sont identiques (TN 70 147 · FP 0 · FN 547 · TP 29 306). Le MLP est légèrement en retrait (FN 1 041, FP 298).

### Graphiques produits par MATLAB R2024a (mode batch, `matlab/generer_graphiques.m`)

Dossier `figures/` :
1. `fig1_classes_et_signal.png` — distribution des classes + P(Malicious) par SYN
2. `fig2_features_cles.png` — histogrammes FPPS + nuage bivarié (frontière)
3. `fig3_seuil_optimal.png` — courbe seuil → Macro F1, optimum marqué
4. `fig4_comparaison_modeles.png` — barres + dispersion des folds + ligne règle experte
5. `fig5_roc_pr.png` — courbes ROC et Precision-Rappel (XGBoost OOF)
6. `fig6_matrices_confusion.png` — 4 matrices de confusion côte à côte
7. `fig7_importance_features.png` — top 15 features (gain)
8. `fig8_courbe_apprentissage.png` — Macro F1 vs nombre d'arbres
9. `fig9_dashboard.png` — tableau de bord récapitulatif 2×2

Données intermédiaires pour MATLAB : dossier `matlab_data/` (14 CSV).

## 5. Livrables du dossier

| Fichier | Contenu |
|---|---|
| `sujet-du-concours.md` | Compréhension de l'énoncé officiel |
| `rapport.md` | Ce rapport |
| `data/TRAIN_DATA.csv`, `TEST_DATA.csv`, `SAMPLE_DATA.csv` | Données du concours |
| `eda.py`, `eda2.py`, `eda3.py`, `eda4.py` | Scripts d'exploration (+ sorties `*_output.txt`) |
| `diag_test.py`, `diag_prob.py` | Diagnostics du drift test et des probabilités |
| `solution_baseline.py` | Pipeline complet (features + CV + XGBoost + submission) |
| `compare_models.py` | Comparaison 4 modèles × 5 folds + optimisation du seuil |
| `build_notebook.py` | Générateur du notebook pédagogique |
| `notebook_pedagogique.ipynb` | **Notebook pédagogique complet (10 sections)** |
| `matlab/generer_graphiques.m` | Script MATLAB des 9 figures (mode batch) |
| `figures/` (fig1…fig9 .png) | Graphiques MATLAB |
| `matlab_data/*.csv` | Données intermédiaires pour MATLAB |
| `submission.csv` | Prédictions finales (25 000 lignes) |
| `download_data.py` | Téléchargement API Kaggle (Bearer KGAT) |

## 6. Enseignements

1. **Toujours valider l'annonce contre les données** : 10 minutes d'EDA ont révélé que
   l'énoncé (7 classes, imbalance extrême, valeurs manquantes) ne décrivait pas le dataset réel.
2. **Chercher d'abord la frontière de décision** : ici 2 features suffisaient ; un modèle complexe
   n'améliore que les 547 cas ambigus, pas la stratégie.
3. **Le holdout local est le seul juge fiable** — surtout quand la distribution du test diverge.
4. **Un test sans cas positifs est un piège classique** des datasets synthétiques : vérifier la
   distribution des features prédictives du test AVANT de finaliser des prédictions.
5. Les datasets « CIC-IDS-like » régénérés aléatoirement perdent la structure réelle du trafic
   (les corrélations réseau authentiques ont disparu sauf pour 2 features).

## 7. Sources

- https://www.kaggle.com/competitions/egspec-cyberguard-codeforgeclub-csbs-2026/overview
- https://www.kaggle.com/competitions/egspec-cyberguard-codeforgeclub-csbs-2026/data
- https://www.kaggle.com/competitions/egspec-cyberguard-codeforgeclub-csbs-2026/rules
- Données téléchargées via l'API Kaggle (token Bearer, compte `jsambin`)

*Note : conclusions marquées « inférence » reposent sur l'analyse statistique, pas sur une
vérification officielle des organisateurs (aucune discussion ni writeup public sur le concours).*
