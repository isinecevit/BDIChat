# BDIChat: A Conversational Depression Detection Dataset using Beck’s Depression Inventory-II

BDIChat consists of conversations between simulated Early Risk Prediction on the Internet Lab (eRisk) patients and mental health professionals, where the patients are asked about their symptoms by the clinician in order to fill the BDI-II test on their behalf. After every turn, the professional marks which BDI-II symptoms the exchange revealed and scores their severity.

![AnnotationGuidelines1](https://github.com/isinecevit/BDIChat/blob/main/assets/crowdsourcing-chat-setup.png?raw=true)

## At a glance

| | |
|---|---|
| **Conversations** | 100 (5 experts per persona) |
| **Turns** | 2,332 (23.3 ± 11.9 per conversation) |
| **Personas** | 20 simulated eRisk 2026 patients, covering minimal, mild, moderate and severe depression |
| **Experts** | 51 mental health professionals (psychologists, clinical psychologists, psychiatrists) recruited on Prolific |
| **Instrument** | Beck Depression Inventory-II (21 symptoms, each scored 0–3) |
| **Language** | English |

BDIChat is, to our knowledge, the first English dataset that combines multi-turn dialogue, expert annotation with a clinical instrument, turn-level multi-label symptom annotation, and symptom-level severity scores.

## What is annotated

- **Symptom relevance.** For each turn (one expert message plus one persona reply), the expert selects which BDI-II items the turn informs. A turn can have several symptoms or none. About 81% of turns have at least one symptom flagged.
- **Symptom severity.** Each flagged symptom gets a score from 0 (absent) to 3 (most severe). Experts could revise scores as the conversation went on, and did so for 22.5% of items.
- **Consensus scores.** For each persona, the item scores from the five experts are averaged into one consensus score per BDI-II item, giving a total score and severity category (minimal 0–13, mild 14–19, moderate 20–28, severe 29–63).

## Tasks and baselines

We provide baselines for two tasks. Fine-tuned encoders outperform zero-shot GPT-4o-mini on both.

**Symptom relevance detection** (multi-label, micro-averaged)

| Model | P | R | F1 |
|---|---|---|---|
| GPT-4o-mini (item descriptions) | 0.435 | 0.492 | 0.462 |
| SBERT, fine-tuned | 0.519 | 0.493 | 0.506 |
| MentalBERT, fine-tuned | **0.531** | **0.534** | **0.532** |

**Symptom severity detection** (4-class ordinal, 0–3)

| Model | Accuracy | MAE | QWK |
|---|---|---|---|
| GPT-4o-mini (zero-shot) | 0.450 | 0.727 | 0.430 |
| SBERT, fine-tuned | 0.543 | 0.596 | 0.601 |
| MentalBERT, fine-tuned | **0.575** | **0.510** | **0.638** |

Other uses include predicting the overall BDI-II score (from the full dialogue or incrementally), building full relevance-plus-severity pipelines, training interviewer models that ask about uncovered symptoms, and checking how consistent LLM personas are across interviewers.

## Repository contents

| Path | Description |
|---|---|
| `data/all-persona-chats/` | One JSON file per interview (100 files), named `p{persona_id}_c{clinician_id}.json`. Full conversation plus turn annotations. |
| `data/bdichat_turn_annotations.csv` | Turn-level table: 2,332 rows, one per turn, with the expert message, persona reply, symptom flags and scores. |
| `data/bdichat_persona_clinician_symptom.csv` | Final BDI-II item scores, total and severity category given by each expert for each persona (100 rows). |
| `data/bdichat_consensus_scores.csv` | Consensus item scores, total and severity category per persona (20 rows). |
| `assets/` | Annotation guidelines given to the experts (PDF) and the screenshot of the annotation interface. |

## Data format

### Conversation files (`data/all-persona-chats/`)

Each file is one interview:

```json
{
  "session_id": "457eb403-3120-48d8-8d95-e7699d789d04",
  "persona_id": 0,
  "clinician_id": 2,
  "start_time": "2026-06-22T13:19:13.111385",
  "end_time": "2026-06-22T13:53:13.570781",
  "messages": [
    {"role": "system", "content": "You are a simulated patient. ..."},
    {"role": "user", "content": "<expert message>"},
    {"role": "assistant", "content": "<persona reply>"}
  ],
  "turn_annotations": [
    {
      "turn": 1,
      "symptoms": ["20. Tiredness or Fatigue"],
      "scores": {"20. Tiredness or Fatigue": {"label": "2", "score": "2"}},
      "annotated_at": "2026-06-22T13:20:50.620599"
    }
  ]
}
```

- `messages` follows the chat format. The `system` message is the persona instruction, `user` is the expert and `assistant` is the simulated patient. Turn *n* is the *n*-th `user` message and the `assistant` reply after it.
- `turn_annotations` lists the symptoms the expert flagged for a turn and the score they gave each one. Turns with no flagged symptom have an empty `symptoms` list.

### Turn-level table (`bdichat_turn_annotations.csv`)

| Column | Meaning |
|---|---|
| `persona_id`, `clinician_id`, `turn` | Identify the session and turn |
| `Message`, `PersonaReply` | Expert message and persona reply |
| `1. Sadness` … `21. Loss of Interest in Sex` | 1 if the expert flagged this BDI-II item as relevant to the turn, otherwise 0 |
| `1.Score` … `21.Score` | Severity score (0–3) the expert gave the item at this turn, or `-1` if the item was not flagged |

### Session and persona tables

`bdichat_persona_clinician_symptom.csv` has `personaId`, `clinicianId`, the 21 item scores, `BDI Total` and `Severity`. `bdichat_consensus_scores.csv` has the same columns per `personaId`, where item scores are the consensus across the five experts.

### Loading the data

```python
import pandas as pd

turns = pd.read_csv("data/bdichat_turn_annotations.csv")
consensus = pd.read_csv("data/bdichat_consensus_scores.csv")

symptom_cols = [c for c in turns.columns if c[:2].strip(". ").isdigit() and not c.endswith("Score")]
turns["n_symptoms"] = turns[symptom_cols].sum(axis=1)
```

## Limitations

- **Simulated patients.** Personas are LLMs, so their replies may depend on how a question is phrased and may not reflect how real patients express symptoms.
- **Different conversations per expert.** No two experts had the same conversation with a persona. Item-level agreement is low (mean Krippendorff’s α = 0.223; total-score α = 0.468), which reflects both clinical disagreement and different information elicited.
- **Consensus scores are not clinically validated ground truth.**
- **Scope.** 20 personas, English only, BDI-II only. The annotator pool was predominantly female and white.

## Ethical considerations

All patient utterances come from LLM personas that are not based on any real individual. Experts gave informed consent, were told about possibly distressing topics, and could withdraw at any time. No personally identifying or demographic information is included; participant IDs are anonymised.

BDIChat is intended for research on conversational depression screening. **It is not a diagnostic tool and not a substitute for professional mental health care.**

## Citation

Coming soon!

## Authors

Isin Su Ecevit, Ana-Maria Bucur and Fabio Crestani, Università della Svizzera italiana (USI), Lugano, Switzerland.

## License

<!-- Add your license here, e.g. CC BY 4.0 for the data. -->
