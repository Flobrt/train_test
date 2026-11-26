# Guide évaluateur – exercice « ponctualité des trains » (PySpark)

## A. Mise en place de A à Z (≈ 20 min)

1. **Héberger les données** : crée un dépôt GitHub **public** et dépose le dossier `data/` (contenu de `data.zip`). L'URL à utiliser est `https://raw.githubusercontent.com/<ORG>/<REPO>/main/data`. Alternative : bucket GCS public ou dossier Drive partagé (plus fastidieux : il faut monter le Drive).
2. **Configurer le notebook** : ouvre `exercice_data_engineer_trains.ipynb` dans Colab (Fichier > Importer) et remplace `<ORG>/<REPO>` dans `DATA_BASE_URL`.
3. **Tester en conditions réelles** : « Exécuter tout » dans une session neuve. Le setup installe PySpark (mode local, Java est déjà présent dans Colab), télécharge les données et crée la session `spark`. Compte 1 à 2 min la première fois.
4. **IA** : choisis une option.
   - *Gemini intégré* : rien à faire, le candidat clique sur l'icône étincelle.
   - *`%%ai` (Claude)* : le candidat ajoute le secret `ANTHROPIC_API_KEY` (onglet 🔑 de Colab, accès notebook activé). Le plus propre : crée une clé dédiée avec une limite de dépense, donne-la le jour J et révoque-la après. L'assistant reçoit automatiquement les schémas des tables, et les prompts sont journalisés dans `prompts_log.jsonl`.
5. **Partager** : « Partager > Tout utilisateur disposant du lien : lecteur ». Le candidat fait « Fichier > Enregistrer une copie dans Drive ».
6. **Récupérer le rendu** : le candidat te partage sa copie (ou l'exporte en `.ipynb`) et te transmet `prompts_log.jsonl`.

Volume : par défaut ~40k lignes de `stop_times`, ce qui suffit pour l'exercice. Pour un jeu plus volumineux et plus « Spark », lance `TRIPS_PER_DAY=50 python generate_data.py` (~400k lignes).

## B. Pièges introduits (corrigé)

| Source | Piège | Attendu du candidat |
|---|---|---|
| stations | 3 doublons de gares (ST101/104/105), noms en casse/espaces différents | déduplication / table de correspondance |
| stations | 3 gares sans coordonnées, 2 avec lat/lon inversés | détection (lat 41-51, lon -5 à 10) |
| trips | ~120 lignes en doublon | `dropDuplicates(["trip_id"])` |
| trips | `service_date` en formats mixtes (ISO / JJ/MM/AAAA) | `coalesce` de plusieurs `to_date` |
| trips | `status` : `ran`, `cancelled`, `CANCELLED`, `annulé` | normalisation |
| stop_times | ~450 doublons exacts | dédoublonnage sur (trip_id, stop_sequence) |
| stop_times | 200 `station_id` orphelins (ST999) | contrôle d'intégrité (`left_anti`), décision documentée |
| stop_times | heures théoriques GTFS > 24:00:00 | `to_timestamp` renvoie `null` : parser en secondes |
| stop_times | `actual_*` en 2 formats | `coalesce` de plusieurs `to_timestamp` |
| stop_times | ~150 retards aberrants (+24h / -10h) | calcul du retard, règle de rejet justifiée |
| trips / stop_times | 5 trains annulés avec horaires réels | incohérence à signaler |
| stop_times | 1er arrêt sans arrivée, dernier sans départ | normal, ne pas traiter comme une erreur |
| incidents.json | JSON imbriqué, champs optionnels, `trip_id` inexistants (T999999) | `multiLine`, accès `cause.category`, gestion des nulls |

**Points spécifiques Spark à observer** : `multiLine=True` pour le JSON, `inferSchema` et ses effets, jointures qui multiplient les lignes (vérifier les counts), fenêtres (`row_number`) pour trouver le dernier arrêt, `collect()`/`toPandas()` évités sur de gros volumes, `show()` vs transformations paresseuses.

**Pièges métier** : retard à l'arrivée **finale** du train (pas la moyenne de tous les arrêts) ; annulés comptés à part et exclus du taux de ponctualité (à justifier) ; mois = date de service (trains après minuit).

## C. Solution de référence

`solution_reference_pyspark.py` (à ne pas partager) : tourne de bout en bout sur les données. Résultats attendus avec les règles ci-dessus : 159 retards aberrants écartés, 200 arrêts avec gare inconnue, 33 trains circulés sans retard final calculable (arrivée finale aberrante ou orpheline). Pour L01 en janvier 2024 : 155 trains programmés, 5 annulés, ponctualité ≈ 74 %. Les chiffres peuvent légèrement varier selon les règles du candidat, l'important est qu'elles soient justifiées.

## D. Grille (sur 20)

| Critère | Points |
|---|---|
| Diagnostic des données (nombre de pièges trouvés) | 5 |
| Qualité du code PySpark et du modèle de données | 5 |
| Contrôles qualité pertinents | 3 |
| Esprit critique face au code de l'IA (vérifs, corrections) | 4 |
| Communication, hypothèses, synthèse | 3 |

## E. Questions de relance

1. Comment avez-vous vérifié qu'une jointure n'a ni perdu ni multiplié de lignes ?
2. L'IA a-t-elle produit une erreur ? Comment l'avez-vous détectée ?
3. Avec 100 fois plus de données, qu'est-ce qui changerait (partitionnement, format Parquet, broadcast join) ?
4. Comment orchestrer et surveiller ce pipeline en production (Airflow, tests de données, alerting) ?

## F. Variante locale (Jupyter sur un PC, sans Colab)

Le même notebook fonctionne en local : la cellule de setup détecte l'environnement, n'installe PySpark que s'il est absent, vérifie la présence de Java et lit les données dans `./data` (aucun hébergement nécessaire).

- **Kit prêt à donner** : `kit_local.zip` = notebook + `data/` + `requirements.txt` + `LISEZMOI_local.md` (instructions pour un venv Python ou pour Docker).
- **Chemin le plus fiable** : l'image Docker `quay.io/jupyter/pyspark-notebook` (Java et Spark inclus). Sous Windows sans Docker, privilégier WSL2.
- **Clé IA** : voir section G (Gemini gratuit par défaut).
- **Deux scénarios** : exercice sur ta machine en visio avec partage d'écran (tu contrôles tout, aucun souci d'installation), ou sur la machine du candidat (envoie le kit à l'avance et fais-lui tester le lancement la veille).
- **Pas d'IA intégrée en local** : seule la commande `%%ai` est fournie. Si tu veux autoriser un autre assistant (Copilot, Jupyter AI, ChatGPT dans le navigateur), précise-le dans les règles du jeu, sinon la journalisation des prompts sera incomplète.

## G. IA gratuite pour la commande `%%ai`

Le notebook gère 3 fournisseurs via la variable `AI_PROVIDER` (cellule « Assistant IA ») :

| Fournisseur | Coût | Mise en place |
|---|---|---|
| `gemini` (défaut) | offre gratuite avec limites de débit (les quotas dépendent du modèle et sont visibles dans AI Studio) | créer une clé sur aistudio.google.com, puis variable d'environnement `GEMINI_API_KEY` (ou saisie à la première utilisation, ou secret Colab) |
| `ollama` | gratuit, 100 % local, sans clé ni quota | installer Ollama, `ollama pull qwen2.5-coder:7b` (~5 Go, prévoir 8 Go de RAM libres au minimum). Qualité inférieure aux modèles hébergés |
| `anthropic` | payant | variable `ANTHROPIC_API_KEY` |

Conseils :
- Modèle Gemini par défaut : `gemini-3.5-flash` (modifiable dans `MODELS`). Si l'API répond « modèle introuvable » ou erreur 429, vérifier le nom du modèle et les quotas dans AI Studio : ils évoluent.
- **Données** : en offre gratuite, vérifie les conditions d'utilisation de Google concernant l'usage des prompts. Ici les données sont fictives, donc sans enjeu.
- Ne partage pas ta clé personnelle avec le candidat : crée-en une dédiée (ou fais-lui créer la sienne, c'est gratuit) et supprime-la après.
- Pour un exercice en visio sur ta machine, la clé reste chez toi.
