# Lancer l'exercice en local

Prérequis : Python 3.9 à 3.12 et Java 11 ou 17 (`java -version` doit répondre).

## Option 1 – Python local
```bash
python -m venv .venv
source .venv/bin/activate          # Windows : .venv\Scripts\activate
pip install -r requirements.txt
export GEMINI_API_KEY="..."   # Windows PowerShell : $env:GEMINI_API_KEY="..."
jupyter lab exercice_data_engineer_trains.ipynb
```
Clé gratuite : https://aistudio.google.com (« Create API key »). Si la variable n'est pas définie, la commande `%%ai` demande la clé à la première utilisation.
Sans clé du tout : installez Ollama (https://ollama.com), lancez `ollama pull qwen2.5-coder:7b` puis mettez `AI_PROVIDER = "ollama"` dans le notebook.

## Option 2 – Docker (Java et Spark inclus, recommandé sous Windows)
```bash
docker run --rm -p 8888:8888 -e GEMINI_API_KEY="..." \
  -v "$PWD":/home/jovyan/work quay.io/jupyter/pyspark-notebook
```
Ouvrez l'URL avec le token affichée dans le terminal, puis le dossier `work/`.

## Notes
- Windows sans Docker : PySpark fonctionne mais lire/écrire des fichiers peut demander `winutils`. WSL2 évite le problème.
- Les données sont dans `data/`. Le notebook n'a rien à télécharger.
