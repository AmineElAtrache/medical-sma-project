# Rapport de Projet — Système Multi-Agents d'Orientation Clinique Préliminaire

**Module :** Systèmes Multi-Agents (SMA)
**Année universitaire :** 2024–2025
**Établissement :** ENIAD

---

## Table des matières

1. [Introduction](#1-introduction)
2. [Objectifs du projet](#2-objectifs-du-projet)
3. [Architecture globale](#3-architecture-globale)
4. [Technologies utilisées](#4-technologies-utilisées)
5. [Le graphe LangGraph — Workflow multi-agents](#5-le-graphe-langgraph--workflow-multi-agents)
6. [Les agents du système](#6-les-agents-du-système)
7. [État partagé (MedicalState)](#7-état-partagé-medicalstate)
8. [Serveur MCP — Lignes directrices de soins](#8-serveur-mcp--lignes-directrices-de-soins)
9. [API REST (FastAPI)](#9-api-rest-fastapi)
10. [Interface utilisateur (Streamlit)](#10-interface-utilisateur-streamlit)
11. [Human-in-the-Loop (HITL)](#11-human-in-the-loop-hitl)
12. [Persistance des données (SQLite)](#12-persistance-des-données-sqlite)
13. [Gestion des erreurs](#13-gestion-des-erreurs)
14. [Logging structuré](#14-logging-structuré)
15. [Containerisation Docker](#15-containerisation-docker)
16. [Intégration continue (GitHub Actions)](#16-intégration-continue-github-actions)
17. [Tests](#17-tests)
18. [Rapport final PDF](#18-rapport-final-pdf)
19. [Structure du projet](#19-structure-du-projet)
20. [Guide d'installation et d'exécution](#20-guide-dinstallation-et-dexécution)
21. [Scénarios de démonstration](#21-scénarios-de-démonstration)
22. [Considérations éthiques](#22-considérations-éthiques)
23. [Conclusion](#23-conclusion)

---

## 1. Introduction

Ce projet réalise un **Système Multi-Agents (SMA)** d'orientation clinique préliminaire, construit dans un cadre strictement académique. Le système simule un workflow médical complet : un patient décrit ses symptômes, un agent pose des questions structurées, un LLM génère une synthèse clinique, un médecin humain valide et donne son avis (Human-in-the-Loop), puis un rapport final structuré est produit.

**⚠️ Ce système ne remplace en aucun cas une consultation médicale professionnelle.**

Le projet couvre l'ensemble du cahier des charges :

- Graphe multi-agents avec LangGraph (StateGraph, routage conditionnel, checkpointing)
- Intégration MCP (Model Context Protocol) pour les lignes directrices de soins
- API REST avec FastAPI (6 endpoints)
- Interface Streamlit (4 écrans)
- Human-in-the-Loop avec interruption du graphe
- Rapport final structuré (7 sections) + export PDF
- Dockerisation complète, CI GitHub Actions, base SQLite

---

## 2. Objectifs du projet

| # | Objectif | Réalisation |
|---|----------|-------------|
| 1 | Construire un graphe multi-agents avec LangGraph | ✅ StateGraph avec 4 nœuds + routage conditionnel |
| 2 | Implémenter un superviseur déterministe | ✅ Routage basé sur l'état, sans LLM |
| 3 | Collecter 5 questions obligatoires au patient | ✅ Agent de diagnostic avec questions structurées |
| 4 | Générer une synthèse clinique via LLM | ✅ Groq (LLaMA 3.3 70B) |
| 5 | Intégrer un serveur MCP | ✅ Dual-mode : HTTP REST + MCP SDK (stdio) |
| 6 | Implémenter le HITL (revue médecin) | ✅ `interrupt_before=["physician_review"]` |
| 7 | Produire un rapport final structuré | ✅ 7 sections + Pydantic + PDF |
| 8 | Fournir une API REST | ✅ FastAPI avec 6 endpoints |
| 9 | Créer une interface utilisateur | ✅ Streamlit, 4 écrans, design moderne |
| 10 | Dockeriser le projet | ✅ Docker Compose avec 3 services |
| 11 | Mettre en place une CI | ✅ GitHub Actions (lint + tests) |
| 12 | Persister les consultations | ✅ SQLite avec tables relationnelles |

---

## 3. Architecture globale

Le système est composé de **3 services** communicant par HTTP :

```
┌─────────────────┐     HTTP      ┌──────────────────┐     HTTP/MCP    ┌─────────────────┐
│   Streamlit UI  │ ◄──────────►  │   FastAPI API    │  ◄──────────►   │  MCP Server     │
│   (port 8501)   │               │   (port 8000)    │                 │  (port 8001)    │
└─────────────────┘               └────────┬─────────┘                 └────────┬────────┘
                                           │                                    │
                                    ┌──────┴──────┐                      ┌──────┴──────┐
                                    │  LangGraph  │                      │ care_guide- │
                                    │  StateGraph │                      │ lines.json  │
                                    │  +MemorySaver│                     │ (20 cond.)  │
                                    └──────┬──────┘                      └─────────────┘
                                           │
                                    ┌──────┴──────┐
                                    │   SQLite    │
                                    │ consulta-   │
                                    │ tions.db    │
                                    └─────────────┘
```

**Flux de données :**

1. L'utilisateur saisit les informations patient dans Streamlit
2. Streamlit appelle l'API FastAPI (`/consultation/start`)
3. FastAPI initialise le graphe LangGraph qui exécute le flux multi-agents
4. L'agent de diagnostic pose 5 questions, puis appelle Groq pour la synthèse
5. L'agent appelle le serveur MCP pour les recommandations intermédiaires
6. Le graphe s'interrompt avant la revue médecin (HITL)
7. Le médecin saisit son avis, le graphe reprend
8. L'agent de rapport génère le rapport final et le sauvegarde en base SQLite
9. Streamlit affiche le rapport et permet le téléchargement PDF

---

## 4. Technologies utilisées

| Technologie | Version | Rôle |
|-------------|---------|------|
| **Python** | 3.11+ | Langage principal |
| **LangGraph** | ≥ 0.2.0 | Orchestration multi-agents (StateGraph, MemorySaver) |
| **LangChain** | ≥ 0.2.0 | Intégration LLM, messages, tools |
| **LangChain-Groq** | ≥ 0.1.0 | Connecteur vers Groq Cloud |
| **Groq** (LLaMA 3.3 70B) | — | LLM pour synthèse clinique et conclusion |
| **FastAPI** | ≥ 0.111.0 | API REST backend |
| **Uvicorn** | ≥ 0.30.0 | Serveur ASGI |
| **Streamlit** | ≥ 1.35.0 | Interface utilisateur frontend |
| **MCP SDK** | ≥ 1.0.0 | Protocole MCP officiel (Mode SDK) |
| **Pydantic** | ≥ 2.0.0 | Validation des données, modèles structurés |
| **SQLite** | 3 (stdlib) | Persistance des consultations |
| **ReportLab** | ≥ 4.2.0 | Génération de PDF |
| **Docker** / **Docker Compose** | — | Containerisation |
| **GitHub Actions** | — | Intégration continue (lint + tests) |
| **Ruff** | — | Linter Python rapide |
| **Pytest** | ≥ 8.0.0 | Framework de tests |

---

## 5. Le graphe LangGraph — Workflow multi-agents

Le cœur du système est un **StateGraph** LangGraph compilé avec un checkpointer `MemorySaver` et une interruption HITL.

### Définition du graphe (`backend/app/graph.py`)

```python
graph = StateGraph(MedicalState)

# 4 nœuds
graph.add_node("supervisor", supervisor_node)
graph.add_node("diagnostic_agent", diagnostic_agent_node)
graph.add_node("physician_review", physician_review_node)
graph.add_node("report_agent", report_agent_node)

# Point d'entrée
graph.add_edge(START, "supervisor")

# Routage conditionnel depuis le superviseur
graph.add_conditional_edges("supervisor", route_supervisor, {
    "diagnostic_agent": "diagnostic_agent",
    "physician_review": "physician_review",
    "report_agent": "report_agent",
    END: END,
})

# Retour vers le superviseur après chaque agent
graph.add_edge("diagnostic_agent", "supervisor")
graph.add_edge("physician_review", "supervisor")
graph.add_edge("report_agent", "supervisor")
```

### Compilation avec HITL

```python
checkpointer = MemorySaver()
medical_graph = build_graph().compile(
    checkpointer=checkpointer,
    interrupt_before=["physician_review"],
)
```

Le `MemorySaver` permet de persister l'état du graphe entre les appels API (chaque `thread_id` a son propre checkpoint). L'option `interrupt_before=["physician_review"]` met le graphe en **pause** automatiquement avant d'exécuter le nœud `physician_review`, implémentant ainsi le pattern Human-in-the-Loop.

### Diagramme du flux

```
START → Supervisor → Diagnostic Agent ⟲ (boucle 5 questions)
                   → Diagnostic Agent (synthèse + MCP)
                   → Physician Review [INTERRUPTION HITL]
                   → Report Agent → Supervisor → END
```

---

## 6. Les agents du système

### 6.1 Superviseur (`backend/app/nodes/supervisor.py`)

Le superviseur est un **routeur déterministe** (pas de LLM). Il examine l'état courant et décide du prochain nœud à exécuter :

| Condition | Prochain nœud | Raison |
|-----------|---------------|--------|
| `final_report` existe | `FINISH` (END) | Consultation terminée |
| `physician_treatment` existe | `report_agent` | Avis médecin reçu → rapport |
| `question_count ≥ 5` et `diagnostic_summary` | `physician_review` | Synthèse prête → revue médecin |
| `question_count > len(answers)` | `FINISH` | Attente réponse patient |
| Sinon | `diagnostic_agent` | Continuer le questionnaire |

Le superviseur retourne un champ `next` dans l'état, lu par la fonction `route_supervisor()` pour le routage conditionnel.

### 6.2 Agent de diagnostic (`backend/app/nodes/diagnostic_agent.py`)

Cet agent a **deux phases** :

**Phase 1 — Collecte (question_count < 5) :**
- Pose la prochaine question parmi les 5 questions obligatoires
- Utilise l'outil `ask_patient` (LangChain `@tool`) pour formater la question
- Incrémente le compteur et met à jour l'état

**Phase 2 — Synthèse (question_count ≥ 5 et 5 réponses collectées) :**
- Enregistre les réponses via l'outil `record_patient_answer`
- Appelle Groq LLM pour générer une **synthèse clinique préliminaire**
- Appelle l'outil `recommend_interim_care` qui consulte le serveur MCP
- Met le statut à `"awaiting_physician"`

**Les 5 questions obligatoires :**
1. Quel est votre symptôme principal ou motif de consultation ?
2. Depuis combien de temps ressentez-vous ces symptômes ?
3. Sur une échelle de 1 à 10, comment évaluez-vous l'intensité de votre douleur ou gêne ?
4. Avez-vous des antécédents médicaux importants ou des allergies connues ?
5. Prenez-vous actuellement des médicaments ? Si oui, lesquels ?

### 6.3 Revue médecin (`backend/app/nodes/physician_review.py`)

Ce nœud est le **point d'interruption HITL**. Le graphe se met en pause *avant* son exécution. L'API injecte l'avis du médecin dans l'état via `medical_graph.update_state()`, puis reprend l'exécution.

Quand le médecin a fourni son avis :
- Enregistre le traitement/conduite à tenir
- Met le statut à `"report_generated"`
- Le superviseur route ensuite vers `report_agent`

### 6.4 Agent de rapport (`backend/app/nodes/report_agent.py`)

- Appelle Groq LLM pour générer une **conclusion générale**
- Formate le rapport en texte structuré (7 sections)
- Crée un modèle Pydantic `FinalReportModel` pour la sortie JSON structurée
- Sauvegarde la consultation en base SQLite via `save_consultation()`
- Met le statut à `"completed"`

**Les 7 sections du rapport :**
1. Informations patient
2. Anamnèse — Questions & Réponses
3. Synthèse clinique préliminaire
4. Recommandation intermédiaire
5. Avis du médecin traitant
6. Conclusion générale
7. Avertissement légal

---

## 7. État partagé (MedicalState)

L'état est défini comme un `TypedDict` LangGraph (`backend/app/state.py`) :

```python
class MedicalState(TypedDict, total=False):
    messages: Annotated[list, add_messages]     # Historique des messages
    next: Literal["diagnostic_agent", ...]       # Prochain nœud (routage)
    patient_info: dict                           # Nom, âge, cas initial
    question_count: int                          # Compteur de questions (0-5)
    questions_and_answers: List[dict]            # Paires Q&R
    current_question: str                        # Question en cours
    diagnostic_summary: str                      # Synthèse clinique (LLM)
    interim_care: str                            # Recommandation intermédiaire (MCP+LLM)
    physician_treatment: str                     # Avis du médecin (HITL)
    final_report: str                            # Rapport final texte
    final_report_json: dict                      # Rapport final structuré (Pydantic)
    consultation_status: Literal[...]            # Statut courant
    thread_id: str                               # ID de session
    error: Optional[str]                         # Erreur éventuelle
```

Le champ `messages` utilise `add_messages` de LangGraph, ce qui **accumule** les messages au lieu de les écraser — permettant un historique complet des interactions.

### Modèles Pydantic

- **`FinalReportModel`** — Validation structurée du rapport final (patient_info, Q&A, synthèse, recommandation, avis médecin, conclusion, disclaimer)
- **`QuestionAnswer`** / **`PatientInfo`** — Modèles de données auxiliaires

---

## 8. Serveur MCP — Lignes directrices de soins

### Architecture (`mcp_server/server.py`)

Le serveur MCP est un service indépendant qui fournit des **lignes directrices de soins cliniques** basées sur les symptômes du patient. Il fonctionne en **double mode** :

| Mode | Commande | Transport | Usage |
|------|----------|-----------|-------|
| HTTP REST | `uvicorn mcp_server.server:app` | HTTP (FastAPI) | Mode par défaut, utilisé par l'API |
| MCP SDK | `python -m mcp_server.server --mcp` | stdio | Protocole MCP officiel |

### Base de données de soins (`mcp_server/data/care_guidelines.json`)

Le fichier JSON contient **20 conditions cliniques**, chacune avec :
- `condition` : nom de la condition
- `keywords` : mots-clés pour le matching
- `guidelines` : recommandations textuelles
- `urgency` : niveau d'urgence (`low`, `medium`, `high`)
- `recommended_actions` : liste d'actions recommandées

**Conditions couvertes :**

| # | Condition | Urgence |
|---|-----------|---------|
| 1 | Syndrome respiratoire supérieur | medium |
| 2 | Troubles digestifs | medium |
| 3 | Syndrome cardiovasculaire d'alerte | high |
| 4 | Douleurs musculo-squelettiques | low |
| 5 | Céphalées / migraines | medium |
| 6 | Troubles dermatologiques | low |
| 7 | Infections ORL | medium |
| 8 | Troubles urologiques | medium |
| 9 | Troubles ophtalmiques | medium |
| 10 | Symptômes bénins / fatigue | low |
| 11 | Diabète et troubles glycémiques | high |
| 12 | Anxiété et troubles dépressifs | medium |
| 13 | Réactions allergiques | high |
| 14 | Troubles du sommeil | low |
| 15 | Hypertension artérielle | high |
| 16 | Traumatismes et blessures | high |
| 17 | Troubles thyroïdiens | medium |
| 18 | Infections cutanées | medium |
| 19 | Urgences respiratoires aiguës | high |
| 20 | Troubles gastro-intestinaux chroniques | medium |

### Algorithme de matching

Le matching utilise une approche par **chevauchement de mots-clés** :

1. **Tokenisation** — Le texte des symptômes et les mots-clés sont normalisés (minuscules, suppression de la ponctuation, filtrage des stop-words français)
2. **Score exact** — Chaque mot-clé trouvé dans les symptômes ajoute +1
3. **Score partiel** — Les correspondances partielles (sous-chaînes) ajoutent +0.5
4. **Tri** — Les résultats sont triés par score décroissant

### Outils MCP SDK

En mode MCP officiel, le serveur expose 3 outils :
- `get_care_guidelines(symptoms)` — Recherche par symptômes
- `get_all_care_guidelines()` — Liste complète
- `get_top_matches(symptoms)` — Top 3 correspondances

### Client MCP (`backend/app/tools/mcp_client.py`)

Le client appelle le serveur MCP par HTTP REST (par défaut) ou via le SDK MCP (si `MCP_USE_SDK=true`). Il est utilisé par l'outil `recommend_interim_care` de l'agent de diagnostic.

---

## 9. API REST (FastAPI)

### Endpoints (`backend/app/api.py`)

| Méthode | Endpoint | Description |
|---------|----------|-------------|
| `GET` | `/health` | Vérification de l'état de l'API |
| `POST` | `/sessions/start` | Crée une session (retourne un `thread_id`) |
| `POST` | `/consultation/start` | Démarre la consultation (initialise le graphe, retourne Q1) |
| `POST` | `/consultation/resume` | Reprend la consultation (réponse patient ou avis médecin) |
| `GET` | `/consultation/{thread_id}` | État complet de la consultation |
| `GET` | `/consultation/{thread_id}/report` | Rapport final |
| `GET` | `/consultations/history` | Historique de toutes les consultations (SQLite) |
| `GET` | `/consultations/history/{thread_id}` | Détails d'une consultation archivée |

### Validation Pydantic

Les requêtes sont validées avec des modèles Pydantic :

```python
class ConsultationStartRequest(BaseModel):
    thread_id: str = Field(..., min_length=1)
    patient_name: str = Field(..., min_length=1)
    patient_age: int = Field(..., ge=1, le=120)
    initial_case: str = Field(..., min_length=10)

class ConsultationResumeRequest(BaseModel):
    thread_id: str = Field(..., min_length=1)
    answer: str = Field(..., min_length=1)
    role: str = Field(..., pattern="^(patient|physician)$")
```

### Gestion du graphe dans l'API

Pour les réponses patient (Q1 à Q4), l'API met à jour l'état du graphe via `medical_graph.update_state()` sans ré-exécuter le graphe complet — ce qui est plus efficace.

Pour la 5ème réponse et l'avis médecin, l'API reprend l'exécution du graphe via `medical_graph.stream(None, config)`, ce qui déclenche :
- La génération de la synthèse via Groq (après Q5)
- La génération du rapport final (après l'avis médecin)

### Middleware CORS

L'API utilise le middleware `CORSMiddleware` de FastAPI pour permettre les appels cross-origin depuis Streamlit.

---

## 10. Interface utilisateur (Streamlit)

### Architecture de l'interface (`frontend/app.py`)

L'interface est structurée en **4 écrans** avec un indicateur de progression :

```
[1. Saisie patient] → [2. Questionnaire] → [3. Revue médecin] → [4. Rapport final]
```

### Écran 1 — Saisie patient
- Formulaire : nom, âge, motif de consultation
- Validation côté client (nom requis, description ≥ 20 caractères)
- Appel `/sessions/start` puis `/consultation/start`

### Écran 2 — Questionnaire clinique
- Affichage de la question courante dans une bulle stylisée
- Métriques : question actuelle, réponses données, questions restantes
- Barre de progression
- Historique des réponses précédentes (expandable)
- Appel `/consultation/resume` avec `role="patient"`

### Écran 3 — Revue médecin (HITL)
- Affichage du dossier patient (informations + Q&R)
- Synthèse clinique préliminaire (formatage markdown → HTML)
- Recommandation intermédiaire
- Zone de texte pour l'avis du médecin
- Appel `/consultation/resume` avec `role="physician"`

### Écran 4 — Rapport final
- Métriques récapitulatives (patient, âge, nombre de questions, statut)
- Sections du rapport dans des cartes avec titres colorés
- Questions & réponses en format structuré
- Bouton "Nouvelle consultation"
- Bouton "Télécharger le rapport PDF"

### Design
- Palette de couleurs : dégradé teal (`#0f766e`, `#115e59`, `#134e4a`)
- Police Inter (Google Fonts)
- Cards avec ombres et effets hover
- Sidebar avec gradient
- Compatible dark mode (couleurs explicites avec `!important`)
- Branding Streamlit masqué (menu, footer, header)

---

## 11. Human-in-the-Loop (HITL)

Le HITL est le mécanisme central qui rend ce système **collaboratif** : l'IA propose, le médecin décide.

### Implémentation

1. **Compilation du graphe** avec `interrupt_before=["physician_review"]`
2. Quand le graphe atteint le nœud `physician_review`, il se met en **pause automatique**
3. L'état est sauvegardé par le `MemorySaver` (checkpoint)
4. L'API retourne `status: "awaiting_physician"` au frontend
5. Le médecin saisit son avis dans l'interface Streamlit
6. L'API appelle `medical_graph.update_state()` pour injecter l'avis du médecin
7. L'API appelle `medical_graph.stream(None)` pour **reprendre** l'exécution
8. Le graphe continue : `physician_review` → `supervisor` → `report_agent` → `END`

### Pourquoi le HITL est essentiel

- **Sécurité** : aucune décision médicale n'est prise automatiquement
- **Validation** : le médecin examine la synthèse IA et peut corriger
- **Responsabilité** : la décision finale est humaine, pas algorithmique
- **Confiance** : le système est un outil d'aide, pas un remplaçant

---

## 12. Persistance des données (SQLite)

### Schéma de la base (`backend/app/database.py`)

```sql
CREATE TABLE consultations (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    thread_id TEXT UNIQUE NOT NULL,
    patient_name TEXT NOT NULL,
    patient_age INTEGER NOT NULL,
    initial_case TEXT NOT NULL,
    diagnostic_summary TEXT,
    interim_care TEXT,
    physician_treatment TEXT,
    conclusion TEXT,
    final_report_json TEXT,
    consultation_status TEXT DEFAULT 'started',
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL
);

CREATE TABLE questions_answers (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    consultation_id INTEGER NOT NULL,
    question_number INTEGER NOT NULL,
    question TEXT NOT NULL,
    answer TEXT NOT NULL,
    FOREIGN KEY (consultation_id) REFERENCES consultations(id)
);
```

**Caractéristiques :**
- Mode WAL (Write-Ahead Logging) pour de meilleures performances concurrentes
- Clés étrangères activées (`PRAGMA foreign_keys=ON`)
- Index sur `thread_id` et `consultation_id`
- Context manager avec `commit`/`rollback` automatique
- Upsert : insertion ou mise à jour si le `thread_id` existe déjà

---

## 13. Gestion des erreurs

### Classes d'exception (`backend/app/exceptions.py`)

Le système utilise une hiérarchie d'exceptions personnalisées :

```
SMABaseError (base)
├── ConsultationNotFoundError    → HTTP 404
├── ConsultationAlreadyExistsError → HTTP 409
├── ReportNotReadyError          → HTTP 404
├── LLMError                     → HTTP 502
├── MCPServerError               → HTTP 503
├── GraphExecutionError          → HTTP 500
├── ValidationError              → HTTP 422
└── DatabaseError                → HTTP 500
```

Chaque exception contient :
- `message` : description lisible par l'utilisateur
- `error_code` : code machine (ex: `CONSULTATION_NOT_FOUND`)
- `detail` : informations techniques

### Handler global

Un `@app.exception_handler(SMABaseError)` capture toutes les exceptions et retourne une réponse JSON structurée avec le bon code HTTP.

### Retry decorator

Un décorateur `retry_on_failure(max_retries=2, delay=1.0)` est disponible pour les appels réseau transitoires (ConnectionError, TimeoutError).

---

## 14. Logging structuré

### Configuration (`backend/app/logging_config.py`)

Le logging utilise le module standard Python avec :

- **Console handler** : sortie stdout pour le monitoring en temps réel
- **File handler** : écriture dans `logs/sma_clinique.log` (encodage UTF-8)
- **Format** : `%(asctime)s | %(levelname)-8s | %(name)s | %(message)s`
- **Niveau configurable** via la variable d'environnement `LOG_LEVEL` (défaut : `INFO`)
- Suppression du bruit des librairies HTTP (`httpx`, `httpcore`, `uvicorn.access`)

Tous les `print()` du backend ont été remplacés par des appels `logger.info()`, `logger.error()`, etc.

---

## 15. Containerisation Docker

### Dockerfile

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY backend/requirements.txt requirements.txt
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8000 8001 8501
```

### Docker Compose (`docker-compose.yml`)

3 services orchestrés :

| Service | Conteneur | Port | Commande |
|---------|-----------|------|----------|
| `mcp-server` | `sma-mcp-server` | 8001 | `uvicorn mcp_server.server:app` |
| `backend` | `sma-backend` | 8000 | `uvicorn backend.app.api:app` |
| `frontend` | `sma-frontend` | 8501 | `streamlit run frontend/app.py` |

**Fonctionnalités :**
- **Healthcheck** sur le serveur MCP (`/health`)
- **Dépendances** : le backend attend que le MCP soit healthy, le frontend attend le backend
- **Volume persistant** (`db-data`) pour la base SQLite
- **Variables d'environnement** injectées via `.env`
- **Réseau interne Docker** : le frontend appelle `http://backend:8000`, le backend appelle `http://mcp-server:8001`

### Lancement

```bash
# Créer le fichier .env
cp .env.example .env
# Éditer .env et ajouter GROQ_API_KEY

# Lancer les 3 services
docker-compose up --build

# Ouvrir http://localhost:8501
```

---

## 16. Intégration continue (GitHub Actions)

### Pipeline (`.github/workflows/ci.yml`)

Le pipeline s'exécute à chaque **push** et **pull request** sur la branche `main` :

| Étape | Description |
|-------|-------------|
| **Checkout** | `actions/checkout@v4` |
| **Setup Python** | `actions/setup-python@v5` (Python 3.11 avec cache pip) |
| **Install** | `pip install -r backend/requirements.txt` + `pip install ruff` |
| **Lint** | `ruff check` avec règles E (erreurs), F (pyflakes), W (warnings), ignore E501 (longueur de ligne) |
| **Tests unitaires** | `pytest tests/test_graph.py tests/test_api.py` (pas besoin de clé API) |
| **Tests d'intégration** | `pytest tests/` (seulement si `GROQ_API_KEY` est configuré en secret) |

---

## 17. Tests

### Structure des tests

| Fichier | Type | Description |
|---------|------|-------------|
| `tests/test_graph.py` | Unitaire | Compilation du graphe, routing superviseur |
| `tests/test_api.py` | Unitaire | Endpoints FastAPI, validation Pydantic |
| `tests/test_scenarios.py` | Intégration | 3 scénarios cliniques complets |
| `tests/conftest.py` | Configuration | Marker `integration` (auto-skip sans API key) |

### Scénarios de test

| # | Scénario | Symptômes | Urgence attendue |
|---|----------|-----------|------------------|
| 1 | Syndrome respiratoire | Toux sèche, 3 jours, douleur 4/10 | medium |
| 2 | Red flags cardiovasculaires | Douleur thoracique, 9/10, hypertendu | high |
| 3 | Cas bénin | Fatigue légère, 2/10, vitamines | low |

### Exécution

```bash
# Tests unitaires (pas de clé API requise)
python -m pytest tests/test_graph.py tests/test_api.py -v

# Tous les tests (clé API requise)
python -m pytest tests/ -v
```

---

## 18. Rapport final PDF

### Génération (`frontend/app.py` — `build_report_pdf()`)

Le rapport PDF est généré avec **ReportLab** et contient :

1. **En-tête** : titre "RAPPORT CLINIQUE FINAL" + sous-titre + référence
2. **Tableau métadonnées** : patient, âge, référence, date, ID session
3. **Section 1** : Motif initial
4. **Section 2** : Anamnèse (tableau Q&R avec lignes alternées colorées)
5. **Section 3** : Synthèse clinique préliminaire (formatage markdown → gras/italique)
6. **Section 4** : Recommandation intermédiaire
7. **Section 5** : Avis du médecin traitant
8. **Section 6** : Conclusion générale
9. **Avertissement légal** (encadré rouge)
10. **Pied de page** : "SMA Clinique — Rapport académique simulé" + numéro de page

**Style :** palette teal, sections avec fond coloré, police Helvetica, mise en page A4 professionnelle.

---

## 19. Structure du projet

```
medical-sma-project/
├── .github/
│   └── workflows/
│       └── ci.yml                    # Pipeline CI GitHub Actions
├── backend/
│   ├── app/
│   │   ├── __init__.py
│   │   ├── api.py                    # API REST FastAPI (6 endpoints)
│   │   ├── config.py                 # Configuration (variables d'environnement)
│   │   ├── database.py               # Module SQLite (persistance)
│   │   ├── exceptions.py             # 8 classes d'exception personnalisées
│   │   ├── graph.py                  # Définition du StateGraph LangGraph
│   │   ├── logging_config.py         # Configuration logging structuré
│   │   ├── state.py                  # MedicalState + modèles Pydantic
│   │   ├── nodes/
│   │   │   ├── __init__.py
│   │   │   ├── supervisor.py         # Superviseur (routage déterministe)
│   │   │   ├── diagnostic_agent.py   # Agent de diagnostic (5 questions + synthèse)
│   │   │   ├── physician_review.py   # Revue médecin (HITL)
│   │   │   └── report_agent.py       # Agent de rapport (rapport final + SQLite)
│   │   └── tools/
│   │       ├── __init__.py
│   │       ├── patient_tools.py      # Outils LangChain (ask_patient, record_answer)
│   │       ├── care_tools.py         # Outil recommend_interim_care (MCP + LLM)
│   │       └── mcp_client.py         # Client MCP (HTTP REST ou SDK)
│   ├── requirements.txt              # Dépendances Python
│   ├── studio_graph.py               # Point d'entrée LangGraph Studio
│   └── studio_demo.py                # Script de démonstration (3 scénarios)
├── frontend/
│   └── app.py                        # Interface Streamlit (4 écrans + PDF)
├── mcp_server/
│   ├── __init__.py
│   ├── server.py                     # Serveur MCP dual-mode (HTTP + SDK)
│   └── data/
│       └── care_guidelines.json      # 20 conditions cliniques
├── tests/
│   ├── conftest.py                   # Configuration pytest + markers
│   ├── test_graph.py                 # Tests unitaires graphe
│   ├── test_api.py                   # Tests unitaires API
│   └── test_scenarios.py             # Tests d'intégration (3 scénarios)
├── .env.example                      # Template des variables d'environnement
├── .gitignore
├── Dockerfile                        # Image Docker Python 3.11
├── docker-compose.yml                # Orchestration 3 services
├── langgraph.json                    # Configuration LangGraph Studio
├── README.md                         # Documentation technique
└── rapport_technique.md              # Rapport technique initial
```

---

## 20. Guide d'installation et d'exécution

### Prérequis

- Python 3.11+
- Clé API Groq (gratuite sur https://console.groq.com)
- Docker et Docker Compose (pour le mode containerisé)

### Installation manuelle

```bash
# Cloner le projet
git clone https://github.com/AmineElAtrache/medical-sma-project.git
cd medical-sma-project

# Créer l'environnement virtuel
python -m venv venv
source venv/bin/activate  # Linux/Mac
# .\venv\Scripts\Activate.ps1  # Windows PowerShell

# Installer les dépendances
pip install -r backend/requirements.txt

# Configurer l'environnement
cp .env.example .env
# Éditer .env et ajouter GROQ_API_KEY=gsk_...
```

### Lancement manuel (3 terminaux)

```bash
# Terminal 1 — Serveur MCP
python -m uvicorn mcp_server.server:app --port 8001

# Terminal 2 — API FastAPI
python -m uvicorn backend.app.api:app --host 0.0.0.0 --port 8000

# Terminal 3 — Interface Streamlit
python -m streamlit run frontend/app.py --server.port 8501
```

### Lancement Docker (recommandé)

```bash
cp .env.example .env
# Éditer .env et ajouter GROQ_API_KEY

docker-compose up --build
# Ouvrir http://localhost:8501
```

---

## 21. Scénarios de démonstration

Le script `backend/studio_demo.py` exécute 3 scénarios complets à travers le graphe :

### Scénario 1 — Syndrome respiratoire simple

| Question | Réponse |
|----------|---------|
| Symptôme principal | Toux sèche persistante avec léger essoufflement |
| Durée | Depuis 3 jours, s'aggrave le soir |
| Intensité douleur | 4 sur 10 |
| Antécédents | Aucun antécédent particulier, pas d'allergie connue |
| Médicaments | Non, aucun médicament actuellement |

**Résultat attendu :** urgence medium, recommandation de consultation si persistance.

### Scénario 2 — Red flags cardiovasculaires

| Question | Réponse |
|----------|---------|
| Symptôme principal | Douleur thoracique intense irradiant vers le bras gauche |
| Durée | Depuis 2 heures, apparue brutalement |
| Intensité douleur | 9 sur 10 |
| Antécédents | Hypertension artérielle, père décédé d'infarctus à 55 ans |
| Médicaments | Amlodipine 5mg quotidien pour l'hypertension |

**Résultat attendu :** urgence high, orientation vers les urgences.

### Scénario 3 — Cas bénin

| Question | Réponse |
|----------|---------|
| Symptôme principal | Fatigue légère et petite baisse de forme |
| Durée | Depuis environ une semaine |
| Intensité douleur | 2 sur 10 |
| Antécédents | Aucun antécédent, bonne santé générale |
| Médicaments | Vitamines en complément alimentaire |

**Résultat attendu :** urgence low, recommandations générales (repos, hydratation).

---

## 22. Considérations éthiques

Le système intègre des garde-fous éthiques à plusieurs niveaux :

1. **Terminologie prudente** : le système n'émet jamais de "diagnostic" — il utilise uniquement les termes "orientation clinique préliminaire" et "recommandation intermédiaire"

2. **Disclaimer omniprésent** : chaque réponse API, chaque écran Streamlit, et le rapport PDF contiennent l'avertissement : *"Ce système ne remplace pas une consultation médicale"*

3. **HITL obligatoire** : aucun rapport ne peut être généré sans la validation d'un médecin humain

4. **Pas d'auto-prescription** : les recommandations sont toujours générales (repos, hydratation, surveillance) et renvoient vers un professionnel de santé

5. **Cadre académique** : le rapport final mentionne explicitement qu'il s'agit d'un exercice académique

---

## 23. Conclusion

Ce projet démontre la mise en œuvre complète d'un **Système Multi-Agents** appliqué au domaine médical, en utilisant des technologies modernes (LangGraph, LangChain, Groq, FastAPI, MCP, Streamlit). Le système couvre l'ensemble du cahier des charges :

- **4 agents** coopérant via un graphe d'état partagé
- **Routage conditionnel** intelligent par le superviseur
- **Intégration MCP** pour les lignes directrices de soins
- **Human-in-the-Loop** pour la validation médicale
- **API REST** complète avec gestion avancée des erreurs
- **Interface utilisateur** moderne et professionnelle
- **Persistance** SQLite, **logging** structuré, **tests** automatisés
- **Dockerisation** et **intégration continue** GitHub Actions

Le caractère académique et les limites éthiques du système sont clairement communiqués à tous les niveaux de l'application.
