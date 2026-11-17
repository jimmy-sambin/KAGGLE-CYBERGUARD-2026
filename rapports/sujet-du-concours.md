# Sujet du concours — EGSPEC CYBERGUARD CODEFORGECLUB CSBS 2026

> **AI-Powered Network Intrusion Detection Challenge** — *Detect. Classify. Defend.*

## 1. Fiche d'identité

| Élément | Détail |
|---|---|
| **Nom** | EGSPEC _CYBERGUARD_CODEFORGECLUB _CSBS_2026 |
| **Type** | Kaggle Community Hackathon (hackathon d'une journée) |
| **Organisateur** | Mohamed Imran (`@techieimran`), avec Barath Manivannan — E.G.S Pillay Engineering College (Inde) |
| **Public cible** | Étudiants de 3ᵉ année d'ingénierie (AI&DS, CSE, IT, CSBS) |
| **Dates** | 26 septembre 2026, 9h00 → 23h50 (une seule journée) |
| **Prix** | Pas de prix en argent (kudos / reconnaissance) |
| **Participation finale** | 31 inscrits · 18 participants · 12 équipes · 12 soumissions |
| **Taille d'équipe** | Strictement 2 à 3 membres |
| **Limite de soumissions** | 10 par équipe (garder les 2 meilleures pour le scoring privé) |

## 2. Le problème en une phrase

À partir d'enregistrements tabulaires de flux réseau (connexions), prédire la classe
multiclasse **`attack_type`** de chaque connexion du jeu de test.

## 3. Les 7 classes à prédire

| Classe | Signification |
|---|---|
| `Normal` | Trafic légitime |
| `DoS` | Déni de service |
| `Probe` | Surveillance réseau / scans de ports |
| `Brute_Force` | Attaques par devinement de mot de passe |
| `Web_Attack` | XSS, injection SQL, exploits web |
| `Botnet` | Trafic de machines infectées |
| `Other` | Anomalies / attaques diverses |

## 4. Le cœur du défi : déséquilibre extrême des classes

- Le trafic bénin (`Normal`) écrase numériquement les attaques.
- L'**accuracy est un piège** : un modèle qui prédit `Normal` partout fait déjà ~95 %.
- La métrique impose de détecter les classes rares (`Botnet`, `Web_Attack`)
  sans générer un torrent de fausses alertes.

## 5. Métrique d'évaluation : Macro F1-Score

F1 calculé **indépendamment pour chacune des 7 classes**, puis moyenne non pondérée :

```
Macro F1 = (1/C) * Σ F1_c           avec C = 7
F1_c     = 2 * P_c * R_c / (P_c + R_c)
```

Chaque classe rare « pèse » autant que `Normal` → c'est la métrique qui dicte toute la stratégie
(poids de classes, rééchantillonnage, seuillage par classe).

## 6. Données fournies

| Fichier | Rôle |
|---|---|
| `TRAIN_DATA.csv` | Flux réseau étiquetés (avec `attack_type`) |
| `TEST_DATA.csv` | Flux réseau sans label (à prédire) |
| `SAMPLE_DATA.csv` | Exemple de format de soumission (15,77 kB) |

**Familles de variables** (d'après la description officielle) :
- *Network Information* : protocole, port source/destination, type de service
- *Flow Information* : durée du flux, nombre de paquets, volume d'octets
- *Packet Statistics* : compteurs paquets/longueurs forward & backward
- *Traffic Behavior* : débit de flux, taux de paquets, temps actif/inactif
- *Network Flags* : SYN, ACK, RST, PSH, etc.
- *Statistical Features* : moyenne, écart-type, min, max du trafic
- Cible : `attack_type`

Quelques valeurs manquantes, variables très corrélées et déséquilibre sont annoncés.

## 7. Format de soumission

CSV à 2 colonnes exactement, comme `SAMPLE_DATA.csv` :

```csv
ID,attack_type
10001,Normal
10002,DoS
10003,Botnet
10004,Probe
10005,Web_Attack
...
```

## 8. Découpage du leaderboard

- **Public** : 30 % du test, visible en temps réel pendant l'événement.
- **Privé** : 70 % du test, révélé après la clôture (9h30 PM) → classement final.

## 9. Barème de notation final (100 pts)

| Composante | Points | Ce que les juges attendent |
|---|---|---|
| Private Kaggle Leaderboard | **60** | Macro F1 privé final |
| Data Analysis & Preprocessing | **10** | EDA, valeurs manquantes, encodage catégoriel, outliers |
| Feature Engineering & Selection | **10** | Features « domaine réseau », ratios, colinéarité |
| ML Methodology & Validation | **10** | Stratified K-Fold, stratégie d'imbalance, choix de modèle |
| Final Defense & Presentation | **10** | Pitch 5 min + doc technique 2-3 pages |

→ Le score Kaggle ne pèse que 60 % : la **rigueur méthodologique est notée**.

## 10. Livrables obligatoires (avant 21h30)

1. `submission.csv` (prédictions Kaggle).
2. Notebook/code auditable — **chaque ligne doit pouvoir être expliquée oralement**.
3. Document technique (PDF, 2-3 pages) : EDA, préprocessing, features, modèle,
   stratégie d'imbalance, CV local vs leaderboard, enseignements et limites.

## 11. Politique d'outils (règles du concours pendant l'événement)

- **Autorisés** : Python, scikit-learn, XGBoost, LightGBM, CatBoost, pandas, NumPy,
  matplotlib, seaborn, notebooks Kaggle/Jupyter.
- **Interdits pendant l'événement** : assistants IA génératifs (ChatGPT, Claude, Gemini,
  Copilot, Cursor…).
  ⚠️ *Note : le concours étant terminé (26 sept. 2026), refaire l'épreuve hors compétition
  avec de l'assistance IA n'enfreint aucune règle — mais un modèle qui reproduit l'épreuve
  en conditions réelles doit s'appuyer sur les outils autorisés uniquement.*

## 12. Feuille de route technique recommandée (pour refaire l'épreuve)

1. **EDA** : distribution de `attack_type`, taux de valeurs manquantes, cardinalité des catégorielles, corrélations.
2. **Baseline** : `STARTER_NOTEBOOK.ipynb` (onglet *Code* du concours) → Macro F1 ≈ **0,60**.
3. **Encodage** : target encoding ou ordinal contrôlé pour les catégorielles à forte cardinalité (attention fuite).
4. **Imbalance** : `class_weight='balanced'` (XGBoost/LightGBM/CatBoost), SMOTE en variante, seuillage par classe optimisé sur la CV.
5. **Features réseau** : octets/paquet, taux de flux, flags combinés, agrégations par IP/port/service.
6. **Validation** : Stratified K-Fold (K=5) reproductible, cohérence CV local ↔ LB public.
7. **Modèles** : LightGBM/CatBoost/XGBoost en multiclasses → soft-voting / stacking des probabilités, puis calibration.
8. **Optimisation pour Macro F1** : ne pas prédire l'argmax brut ; ajuster les probabilités par classe pour maximiser le F1 macro sur l'out-of-fold.

## 13. Sources

- Overview : https://www.kaggle.com/competitions/egspec-cyberguard-codeforgeclub-csbs-2026/overview
- Description : https://www.kaggle.com/competitions/egspec-cyberguard-codeforgeclub-csbs-2026/overview/description
- Evaluation : https://www.kaggle.com/competitions/egspec-cyberguard-codeforgeclub-csbs-2026/overview/evaluation
- Données : https://www.kaggle.com/competitions/egspec-cyberguard-codeforgeclub-csbs-2026/data
- Règles : https://www.kaggle.com/competitions/egspec-cyberguard-codeforgeclub-csbs-2026/rules
