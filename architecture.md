# Architecture technique — Yapi Immobilier Automation

## Vue d'ensemble

```
EMAIL ENTRANT
     │
     ▼
Gmail Trigger (polling 1 min)
     │
     ▼
Agent Trieur (llama-4-scout)
     │
     ├── hors_sujet / spam → Mark as Read (fin)
     │
     └── lead
          │
          ▼
     Get Message (complet)
          │
          ▼
     Détecter Type Email (Code JS)
     [extrait: from, subject, body, isReply]
          │
          ▼
     Add Label Gmail "Clients/Immobilier"
          │
          ▼
     Chercher Client Existant (Google Sheets)
          │
          ▼
     Filter (email match ?)
          │
          ▼
     Nouveau ou Réponse ? (If)
          │
    ┌─────┴──────┐
  True         False
(existant)   (nouveau)
    │              │
    ▼              ▼
LLM Chain2    LLM Nouveau
(fusion)      (extraction)
    │              │
    ▼              ▼
Code JS1      Code JS
(parser)      (parser)
    │              │
    ▼              ▼
Update Row   Append Row
(CRM MAJ)    (CRM créa)
    │              │
    └──────┬───────┘
           │
           ▼
      If urgent ?
      (score ≥ 85 + signal)
           │
    ┌──────┴──────┐
  True           False
(urgent)       (normal)
    │               │
    ▼               ▼
LLM Urgent    Email Complet ?
    │               │
    ▼          ┌────┴────┐
Gmail 2h     True      False
    │       (complet) (incomplet)
    ▼           │          │
Slack          ▼          ▼
urgent    Get biens   LLM demande
    │     (Sheets)    infos
    ▼         │          │
Mark lu   Code filtre   Gmail
          (ville/budget) infos
              │          │
              ▼          ▼
          Code fusion  Mark lu
          (client+biens)
              │
              ▼
          LLM Matching
          (3 meilleurs)
              │
              ▼
          Code Parser
          (JSON propre)
              │
              ▼
          If biens > 0 ?
              │
         ┌────┴────┐
       True      False
      (biens)  (aucun)
         │         │
         ▼         ▼
    LLM Email  LLM Email
    Biens      Court
    (fusionné) (rappel 48h)
         │         │
         ▼         ▼
      Gmail      Gmail
      confirmer  confirmer
         │         │
         ▼         ▼
    Update CRM  Mark lu
    (biens_references
     date_envoi
     statut_agent)
         │
         ▼
      Mark lu
```

---

## Workflow 2 — Relances

```
Schedule Trigger (toutes les heures)
     │
     ▼
Get all rows (CRM)
     │
     ▼
Filter relance
(statut=incomplet AND nb_relances < 3)
     │
     ▼
Code filter
(horaires ouvrés 8h30-17h30
 pas weekend, pas férié
 calcul jours écoulés)
     │
     ▼
Switch action
     │
     ├── relance_2 (j+2, nb_relances=0)
     │        │
     │        ▼
     │   LLM relance chaleureux
     │        │
     │        ▼
     │   Gmail "On ne vous a pas oublié"
     │        │
     │        ▼
     │   Update CRM (nb_relances+1)
     │
     ├── relance_3 (j+7, nb_relances=1)
     │        │
     │        ▼
     │   LLM FOMO doux
     │        │
     │        ▼
     │   Gmail "Votre projet — on y tient"
     │        │
     │        ▼
     │   Update CRM (nb_relances+1)
     │
     └── perdu (j+14, nb_relances≥2)
              │
              ▼
         Update CRM (statut=perdu)
              │
              ▼
         Slack #immobilier-perdu ❌
```

---

## Scoring des leads

| Critère | Points |
|---------|--------|
| budget_max renseigné | +25 |
| projet clair | +20 |
| localisation précise | +15 |
| telephone renseigné | +15 |
| signal urgence détecté | +25 BONUS |

**Température** :
- 75-100 → chaud 🔥
- 40-74 → tiède 🌤
- 0-39 → froid ❄️

**Urgence** : score ≥ 85 ET signal non vide → branche urgente (rappel 2h)

---

## Dossier complet

Un dossier est **complet** si et seulement si ces 5 champs sont renseignés :
1. `telephone`
2. `projet` (achat / vente / location / estimation)
3. `type_bien` (appartement / maison / terrain / commerce / autre)
4. `localisation`
5. `budget_max`

---

## Formulaire web → Pipeline

```
Page HTML (yapi-formulaire.html)
     │
     │ POST JSON
     ▼
Webhook n8n (/webhook/formulaire-yapi)
     │
     ▼
[À connecter au pipeline existant]
Code JS de transformation
{
  prenom, nom, email, telephone,
  projet, type_bien, localisation,
  budget_max, message, source: "Formulaire Web"
}
     │
     ▼
Même pipeline que Email (à partir de LLM Nouveau client)
```

---

## Structure JSON lead qualifié

```json
{
  "nom": "",
  "prenom": "",
  "email": "",
  "telephone": "",
  "source": "Email | Formulaire Web",
  "projet": "achat | vente | location | estimation",
  "type_bien": "appartement | maison | terrain | commerce | autre",
  "localisation": "",
  "budget_min": "",
  "budget_max": "",
  "surface": "",
  "nb_pieces": "",
  "delai": "",
  "financement": "",
  "resume": "[projet] [type_bien] à [localisation] budget [budget_max]",
  "motivation": "",
  "contraintes": "",
  "infos_manquantes": [],
  "statut": "complet | incomplet",
  "score": 0,
  "temperature": "chaud | tiède | froid",
  "urgence_signal": null
}
```

---

## Canaux Slack

| Canal | Usage |
|-------|-------|
| #immobilier | Leads normaux (🟢 complet, 🟡 incomplet) |
| #immobilier-urgent | Leads urgents 🚨 (rappel 2h) |
| #immobilier-perdu | Leads archivés ❌ |
