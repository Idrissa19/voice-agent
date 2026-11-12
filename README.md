# Assurantia — Agent vocal de service client

Agent vocal en français pour le service client d'une compagnie d'assurance fictive (Assurantia). Il répond à l'oral aux questions sur les contrats, les sinistres et les résiliations. Pour ne pas inventer de réponse, il s'appuie sur une base de connaissances interrogée par recherche vectorielle (RAG).

## Fonctionnement

```
voix du client ─▶ STT (Whisper, en local) ─▶ LLM (Groq ou Ollama) ─▶ TTS (ElevenLabs) ─▶ voix de l'agent
                                               │
                                               └─▶ outil search_knowledge_base ─▶ ChromaDB (RAG)
```

- **Temps réel** : [LiveKit Agents](https://docs.livekit.io/agents/) gère la session audio (WebRTC) et la détection de prise de parole (VAD Silero).
- **Transcription** : Whisper large-v3-turbo tourne en local via `mlx-whisper`. Aucun audio n'est envoyé à un service externe.
- **Raisonnement** : un LLM hébergé sur Groq (`openai/gpt-oss-120b`), ou un modèle local via Ollama.
- **Synthèse vocale** : ElevenLabs, en français.
- **RAG** : les documents de `knowledge_base/` sont découpés par paragraphes entiers (avec chevauchement), vectorisés avec un modèle d'embedding multilingue (`paraphrase-multilingual-MiniLM-L12-v2`) et stockés dans ChromaDB. Le LLM appelle l'outil `search_knowledge_base` avant toute réponse sur un contrat, un sinistre ou une résiliation.
- **Garde-fous** : le prompt système interdit d'inventer un délai, un montant ou une condition. Il demande aussi de proposer un conseiller humain quand la base ne contient pas la réponse ou que le client reste insatisfait.

## Structure

| Fichier | Rôle |
|---|---|
| `agent.py` | Point d'entrée : session vocale, prompt système, choix du LLM |
| `stt_local.py` | Adaptateur LiveKit pour Whisper en local (transcription dans un thread séparé) |
| `rag.py` | Découpage, indexation et recherche dans ChromaDB |
| `tools.py` | Outil `search_knowledge_base` exposé au LLM |
| `ingest.py` | Construit l'index vectoriel à partir de `knowledge_base/` |
| `download_model.py` | Télécharge le modèle Whisper à l'avance |
| `knowledge_base/` | Documents de référence (contrats, sinistres, résiliation) |

## Installation

Prérequis : Python 3.10+, un Mac Apple Silicon (pour `mlx-whisper`) et un compte [LiveKit Cloud](https://cloud.livekit.io).

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env
```

Remplis ensuite `.env` avec tes clés LiveKit, ElevenLabs et Groq.

## Lancement

```bash
python download_model.py
python ingest.py
python agent.py dev
```

Ouvre ensuite le [playground LiveKit](https://agents-playground.livekit.io) pour parler à l'agent.

Pour utiliser un modèle local à la place de Groq, lance Ollama (`ollama pull llama3.1`), puis mets `LLM_PROVIDER=ollama` dans `.env`.

## Pistes d'amélioration

- Brancher la téléphonie (SIP via Twilio) pour recevoir de vrais appels.
- Ajouter une interface pour gérer les documents de la base de connaissances.
- Passer à une transcription en streaming pour réduire la latence.
