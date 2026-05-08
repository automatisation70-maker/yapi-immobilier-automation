# 🏠 Yapi Immobilier — Système d'Automatisation

Système complet d'automatisation pour agence immobilière, construit avec **n8n**, **Google Sheets**, et **Groq (LLaMA)**.

---

## 📁 Structure du projet

```
yapi-immobilier-automation/
├── formulaire-web/
│   └── yapi-formulaire.html        # Page formulaire prospect (HTML standalone)
├── workflows-n8n/
│   ├── workflow-1-traitement-emails.json   # Traitement emails entrants
│   ├── workflow-2-relances.json            # Relances automatiques
│   └── workflow-3-matching.json           # Matching biens + email fusionné
├── docs/
│   └── architecture.md             # Documentation technique complète
└── README.md
```

---

## 🏗️ Architecture globale

### Workflow 1 — Traitement emails entrants
- **Déclencheur** : Gmail Trigger (polling toutes les minutes)
- **Tri intelligent** : Double trieur LLM (immobilier vs assurance)
- **Qualification** : Extraction automatique des données prospects
- **CRM** : Sauvegarde/MAJ Google Sheets (36 colonnes)
- **Emails** : Confirmation, demande infos manquantes, urgence
- **Notifications** : Slack (#immobilier, #immobilier-urgent)

### Workflow 2 — Relances automatiques
- **Déclencheur** : Schedule Trigger (toutes les heures)
- **Logique** : Relance J+2, J+7, archivage J+14
- **Horaires ouvrés** : Lun-Ven 8h30-17h30, hors jours fériés
- **Notifications** : Slack (#immobilier-perdu)

### Workflow 3 — Matching automatique
- **Déclencheur** : Dossier complet détecté
- **Matching** : LLM sélectionne les 3 meilleurs biens
- **Email fusionné** : Confirmation dossier + sélection biens
- **CRM** : Mise à jour colonnes Biens proposés / Date envoi

---

## 🤖 LLMs utilisés

| Nœud | Modèle | Rôle |
|------|--------|------|
| Agent Trieur | llama-4-scout-17b | Classification email immobilier |
| Agent Trieur2 | llama-4-scout-17b | Classification email assurance |
| LLM Nouveau client | llama-3.3-70b | Qualification lead entrant |
| LLM Client existant | llama-3.3-70b | Mise à jour dossier existant |
| LLM Matching | llama-3.3-70b | Sélection 3 meilleurs biens |
| LLM Email Biens | llama-3.3-70b | Génération email HTML fusionné |
| LLM Urgent | llama-3.3-70b | Email urgence |
| LLM Relance J+2 | llama-3.3-70b | Email relance chaleureux |
| LLM Relance J+7 | llama-3.3-70b | Email FOMO doux |

---

## 📊 Google Sheets — CRM_Leads_Immobilier

**36 colonnes** organisées en 6 blocs :

| Bloc | Colonnes |
|------|----------|
| 👤 Identité | ID Lead, Nom, Prénom, Email, Téléphone |
| 🏠 Projet | Source, Type projet, Type bien, Localisation, Budgets, Surface, Pièces, Délai, Financement |
| 🧠 Contexte | Résumé besoin, Motivation, Signal urgent, Contraintes |
| 📊 Qualification | Lead statut, Infos manquantes, Score, Température |
| ⚙️ Suivi | Date réception, Dernière MAJ, Relance envoyée, Nb relances, Statut agent, Biens proposés, Date envoi biens |
| 🔧 Métadonnées | Agent assigné, Canal précis, Thread ID, isReply, Doublon, Notes, Historique |

---

## 🌐 Formulaire Web

Page HTML standalone aux couleurs Yapi Immobilier.

**Champs** : Prénom, Nom, Email, Téléphone, Type projet, Type bien, Localisation, Budget max, Message libre.

**Configuration** : Remplacer `VOTRE_WEBHOOK_N8N_ICI` dans `yapi-formulaire.html` par l'URL du webhook n8n.

---

## ⚙️ Configuration requise

### n8n
- Instance n8n (self-hosted ou cloud)
- Credentials Gmail OAuth2
- Credentials Google Sheets OAuth2
- API Key Groq
- Webhook Slack

### Variables à adapter
- URL webhook formulaire web
- ID Google Sheet CRM
- ID Google Sheet biens_immobiliers
- Canaux Slack (#immobilier, #immobilier-urgent, #immobilier-perdu)
- Clés API Groq (MEUB-0, MEUB-1)

---

## 📧 Emails automatiques

| Email | Déclencheur | Délai réponse |
|-------|------------|---------------|
| Confirmation dossier + biens | Dossier complet | Immédiat |
| Demande infos manquantes | Dossier incomplet | Immédiat |
| Email urgent | Score ≥ 85 + signal urgent | Immédiat (rappel 2h) |
| Relance J+2 | Incomplet sans réponse 2j | Automatique |
| Relance J+7 | Incomplet sans réponse 7j | Automatique |
| Archivage | Sans réponse 14j | Automatique |

---

## 🚧 Roadmap (V3)

- [ ] Formulaire web → webhook → pipeline existant
- [ ] Google Calendar — créneaux + réservation automatique
- [ ] Dashboard HTML temps réel (Google Sheets → React)
- [ ] Analyse et stockage documents (Google Drive)
- [ ] Synchronisation HubSpot / Notion
- [ ] Multi-sources (SeLoger, Leboncoin via webhook)

---

## 👨‍💻 Stack technique

- **Automatisation** : n8n (self-hosted)
- **LLM** : Groq API (LLaMA 3.3 70b, LLaMA 4 Scout)
- **CRM** : Google Sheets
- **Email** : Gmail API
- **Notifications** : Slack
- **Formulaire** : HTML/CSS/JS vanilla
- **Biens** : Google Sheets (150 biens fictifs)
