# Mise en place d'un environnement de développement pour un système de trading

Ce guide en français décrit une configuration minimale, reproductible et extensible pour travailler sur un système de trading algorithmique. Il privilégie Python, mais les étapes sont facilement adaptables à d'autres langages.

## 1. Prérequis système
- **OS** : Linux, macOS ou WSL2 sur Windows.
- **Outils** : `git`, `curl`, `make`, `gcc`/`clang` pour compiler les dépendances natives.
- **Python** : version 3.10 ou 3.11 recommandée. Installez `pyenv` pour gérer plusieurs versions si nécessaire.

```bash
# Exemple d'installation des dépendances de compilation (Debian/Ubuntu)
sudo apt-get update && sudo apt-get install -y build-essential python3-dev python3-venv libffi-dev libssl-dev
```

## 2. Création de l'environnement Python isolé
```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
```
> Utilisez `pyenv local 3.11.x` si vous voulez verrouiller une version précise de Python dans votre projet.

## 3. Dépendances Python de base pour un système de trading
Installez un socle commun couvrant la collecte de données, l'analyse, le backtest et l'automatisation :

```bash
pip install "pandas>=2.2" "numpy>=1.26" "scipy>=1.12" "ta>=0.11" "ccxt>=4.3" \
  "backtrader>=1.9" "vectorbt>=0.26" "yfinance>=0.2" "jupyterlab>=4.2" \
  "pydantic>=2.7" "python-dotenv>=1.0" "pre-commit>=3.7"
```

### TA-Lib (indicateurs techniques avancés)
TA-Lib nécessite une bibliothèque native :
```bash
# Linux (Debian/Ubuntu)
sudo apt-get install -y ta-lib
pip install TA-Lib
```
Si `ta-lib` n'est pas disponible dans votre distribution, compilez-la :
```bash
curl -L https://sourceforge.net/projects/ta-lib/files/ta-lib/0.4.0/ta-lib-0.4.0-src.tar.gz | tar xz
cd ta-lib-0.4.0 && ./configure --prefix=/usr && make && sudo make install
pip install TA-Lib
```

## 4. Variables d'environnement sensibles
Stockez vos clés API (exchange, data provider) dans un fichier `.env` non versionné :
```bash
cp .env.example .env
# puis éditez .env
EXCHANGE_KEY="..."
EXCHANGE_SECRET="..."
DATA_PROVIDER_TOKEN="..."
```
Chargez-les automatiquement avec `python-dotenv` ou via votre orchestrateur (Docker, systemd, etc.).

## 5. Organisation du dépôt
Adoptez une structure claire pour séparer stratégie, infrastructure et données :
```
project/
├── data/                # Jeux de données et caches locaux (exclus du VCS)
├── notebooks/           # Explorations et prototypes
├── src/                 # Code applicatif (brokers, stratégies, backtests)
├── config/              # Configs YAML/JSON, schémas Pydantic
├── scripts/             # Jobs ponctuels (ingestion, maintenance)
├── tests/               # Tests unitaires et d'intégration
├── requirements.in      # Vos dépendances déclaratives
├── requirements.txt     # Versionnées via `pip-compile`
└── pyproject.toml       # Métadonnées et outillage
```
Ajoutez `data/`, `.env` et les caches d'exécution à votre `.gitignore`.

## 6. Qualité, tests et vérifications
Activez un pipeline local léger :
```bash
# Hooks Git
pre-commit install

# Lint/format
pip install "ruff>=0.4" "black>=24.4"
ruff check src tests
black --check src tests

# Tests
pytest -q
```

## 7. Option Docker pour la reproductibilité
Un Dockerfile minimal pour encapsuler l'environnement :
```Dockerfile
FROM python:3.11-slim
RUN apt-get update && apt-get install -y build-essential libffi-dev libssl-dev ta-lib && rm -rf /var/lib/apt/lists/*
WORKDIR /app
COPY requirements.txt ./
RUN pip install --upgrade pip && pip install -r requirements.txt
COPY . .
CMD ["bash"]
```
Construisez et utilisez l'image :
```bash
docker build -t trading-dev .
docker run -it --env-file .env -v $(pwd):/app trading-dev
```

## 8. Vérification rapide
- `python -c "import pandas, ccxt, backtrader; print('ok')"` : vérifie les imports principaux.
- `jupyter lab` : lance l'environnement interactif.
- `pytest` : valide vos tests.

## 9. Adapter l'environnement à une approche PO3/OPR multi-sessions
Pour les stratégies inspirées d'ICT (PO3) qui exploitent l'Opening Price Range (OPR) des sessions asiatique, londonienne et new-yorkaise, préparez l'environnement avec ces garde-fous :

- **Fuseaux horaires et sessions** : verrouillez la timezone dans votre code (ex. `Europe/Paris`) pour éviter les décalages d'heure d'été/hiver. Stockez les fenêtres de sessions dans un YAML versionné (voir `config/po3_sessions.example.yaml`).
- **Ingestion des données** : récupérez au minimum les données en M1/M5 pour projeter les OPR sur M15/H1. Exemple rapide avec CCXT :
  ```python
  import ccxt, pandas as pd

  ex = ccxt.binance({"enableRateLimit": True})
  ohlcv = ex.fetch_ohlcv("BTC/USDT", timeframe="1m", limit=2000)
  df = pd.DataFrame(ohlcv, columns=["timestamp","open","high","low","close","volume"])
  df["timestamp"] = pd.to_datetime(df["timestamp"], unit="ms", utc=True).dt.tz_convert("Europe/Paris")
  ```
- **Backtests PO3** : dans vos notebooks ou scripts backtrader/vectorbt, isolez les trois plages horaires (Asie, Londres, New York) pour tracer les OPR et les précédents hauts/bas (PO3). Conservez 2-3 jours de lookback minimum pour repérer la structure.
- **Gestion des niveaux** : ajoutez une marge (`buffer_pips`) pour éviter les entrées exactes sur le niveau OPR/PO3 et réduire le slippage.
- **Paramétrage reproductible** : dupliquez `config/po3_sessions.example.yaml` vers `config/po3_sessions.yaml` et ajustez les horaires de vos brokers/données. Versionnez uniquement le fichier d'exemple.

Avec ces étapes, vous disposez d'un environnement prêt pour développer, backtester et déployer un système de trading algorithmique.
