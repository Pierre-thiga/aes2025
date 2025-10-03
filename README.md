# Exercice 1 - Flow de création d'US

> **Objectif** : à partir d’une *initiative* saisie dans un chat n8n, générer automatiquement des **User Stories structurées** (via un LLM + parser), puis **créer chaque Story dans Jira**.
>
> **Durée** : 30–45 min 

---

## Sommaire

1. Pré-requis & vue d’ensemble du flow
2. Node 🧩 **Chat Trigger** – point d’entrée utilisateur
3. Node 🧩 **Set – Product Description** – contexte produit
4. Node 🤖 **OpenAI Chat Model** – modèle de langage
5. Node 🧾 **Structured Output Parser** – schéma JSON de sortie
6. Node 🧠 **LLM Chain – Write User Stories** – prompt & génération
7. Node 🔀 **Split Out** – itération sur chaque Story
8. Node 📥 **Jira – Create Issue** – création des User Stories
9. Connexions entre les nœuds (wiring)
10. Tests, validation & dépannage
11. Annexe : Bonnes pratiques de prompt pour nœuds LLM

---

## 1) Pré-requis & vue d’ensemble

### Pré-requis

- **n8n Cloud** ou n8n local (Docker) accessible.
- **Credentials OpenAI** (ou autre LLM compatible) configurés dans n8n.
- **Credentials Jira Software Cloud** (email Atlassian + API token) configurés dans n8n.
- Accès à un **projet Jira** (clé, issue type *Story*, éventuels custom fields).

### Architecture du flow

```
Chat Trigger → Set(ProductDescription) → LLM Chain(Write User Stories)
   ↘                    ↗            ↘
   (input)        (context)      Output Parser + LLM model
                                        ↓
                                 Split Out (user_stories[])
                                        ↓
                              Jira – Create Issue (Story)
```

---

## 2) Node — **Chat Trigger**&#x20;

### 🎯 Rôle

Point d’entrée : l’utilisateur colle l’initiative dans un chat embarqué.

### 🔎 Où le trouver / ajouter

- Dans l’éditeur n8n : **Add node → AI → Triggers → Chat Trigger**.
- Dépose-le sur le canvas.

### ⚙️ Configuration (minimum)

- Laisse les **Options** par défaut (un *webhookId* est généré).
- Renomme le nœud en \`chat\`;

### 🧪 Test rapide

- Clique **Execute workflow** → ouvre le panneau **Chat** si disponible → envoie un texte (initiative).
- La donnée sera disponible via `$('chat').item.json.chatInput`.

### 🧱 Code du nœud

```json
{
  "parameters": {
    "options": {}
  },
  "type": "@n8n/n8n-nodes-langchain.chatTrigger",
  "typeVersion": 1.1,
  "position": [16, 2512],
  "id": "373832cc-845c-4bf0-bba7-3e6fd6e9725f",
  "name": "chat",
  "webhookId": "f42bc8cc-4a76-4fcd-80d9-5f05603f78d1"
}
```

---

## 3) Node — **Set (Product Description)**

## 🎯 Rôle

Fournir au LLM un **contexte produit** clair et stable, distinct de l’initiative.

### 🔎 Où le trouver / ajouter

- **Add node → Data transformation → Set**.

### ⚙️ Configuration

- Onglet **Assignments** → ajoute une variable :
  - **Name** : `ProductDescription`
  - **Type** : `string`
  - **Value** : votre description synthétique (ex. ci-dessous).
- Renomme le nœud : \`**Product Description**\`

### ✅ Bonnes pratiques

- Rédigez une description **de l'objectif de ton produit, des utilisateurs et des fonctionnalités principales**.;

### 🧱 Code du nœud

```json
{
  "parameters": {
    "assignments": {
      "assignments": [
        {
          "id": "c4562e6e-5eb7-4826-9604-7a919fd2cbba",
          "name": "ProductDescription",
          "value": "La description de ton produit",
          "type": "string"
        }
      ]
    },
    "options": {}
  },
  "type": "n8n-nodes-base.set",
  "typeVersion": 3.4,
  "position": [400, 2512],
  "id": "d23836e1-e364-4c97-ba8a-91c9aaa4d0a3",
  "name": "Product Desciption"
}
```

---

## 4) Node — **OpenAI Chat Model**

### 🎯 Rôle

Fournir le **modèle de langage** (LLM) utilisé par la *LLM Chain*.

### 🔎 Où le trouver / ajouter

- **Add node → AI → Language Models → OpenAI Chat Model** (ou "OpenAI").

### ⚙️ Configuration

- **Model** : `gpt-4o` *(ou celui disponible sur votre compte)*
- **Options → Temperature** : `0.2` (plus déterministe)
- **Credentials** : choisissez vos **OpenAI API** configurés.
- Renommez : \`\`.

> ℹ️ Cette brique **doit se connecter à un LLM pour fonctionner.**

### 🧱 Code du nœud

```json
{
  "parameters": {
    "model": {
      "__rl": true,
      "value": "gpt-4o",
      "mode": "list",
      "cachedResultName": "gpt-4o"
    },
    "options": {
      "temperature": 0.2
    }
  },
  "type": "@n8n/n8n-nodes-langchain.lmChatOpenAi",
  "typeVersion": 1.2,
  "position": [784, 2720],
  "id": "cf2b83b4-b765-4e90-a80d-93a2eb51f85f",
  "name": "gpt-4o-for-Automated-Workflow",
  "credentials": {
    "openAiApi": {
      "id": "t9veyCUirYiWelXS",
      "name": "OpenAi account 4"
    }
  }
}
```

---

## 5) Node — **Structured Output Parser**&#x20;

### 🎯 Rôle

Imposer un **schéma JSON** que le LLM doit respecter (liste de User Stories avec `title` et `description`).

### 🔎 Où le trouver / ajouter

- **Add node → AI → Output Parsers → Structured Output**.

### ⚙️ Configuration

- **Schema type** : `Manual`
- **Input schema** : collez le JSON Schema fourni (ci-dessous).

### 🧱 Code du nœud

```json
{
  "parameters": {
    "schemaType": "manual",
    "inputSchema": "{\n  \"$schema\": \"https://json-schema.org/draft/2020-12/schema\",\n  \"title\": \"User Story Document\",\n  \"type\": \"object\",\n  \"properties\": {    \n    \"user_stories\": {\n      \"type\": \"array\",\n      \"description\": \"List of User Stories derived from the Epic.\",\n      \"items\": {\n        \"type\": \"object\",\n        \"properties\": {\n          \"title\": {\n            \"type\": \"string\",\n            \"description\": \"A concise and user-oriented name for the User Story.\"\n          },\n          \"description\": {\n            \"type\": \"string\",\n            \"description\": \"Standard User Story format: 'As a [user role], I want [feature or action], so that [expected benefit]'.\"\n          }\n        },\n        \"required\": [\"title\", \"description\"]\n      }\n    }\n  },\n  \"required\": [\"user_stories\"]\n}"
  },
  "type": "@n8n/n8n-nodes-langchain.outputParserStructured",
  "typeVersion": 1.2,
  "position": [944, 2720],
  "id": "d5a26d53-74ae-426c-a8b3-1fc9abee3791",
  "name": "User Stories Structured Output Parser"
}
```

---

## 6) Node — **LLM Chain – Write User Stories**&#x20;

### 🎯 Rôle

Combiner **Prompt + Modèle + Parser** pour générer un document de User Stories **structuré** à partir de l’initiative.

### 🔎 Où le trouver / ajouter

- **Add node → AI → Chains → LLM Chain**.

### ⚙️ Configuration

- **Prompt type** : `define`
- **Messages** : collez le message structuré (section *Prompt* ci-dessous), en utilisant les variables :
  - `{{ $('chat').item.json.chatInput }}` ← texte saisi par l’utilisateur dans le chat
  - `{{ $json.ProductDescription }}` ← contexte du nœud *Set*
- **Has output parser** : `ON` → reliez ce nœud au **Structured Output Parser** (connexion AI)
- **Language Model** : reliez au nœud **OpenAI Chat Model** (connexion AI)
- **On error** : `Continue` (pour éviter de bloquer le flux si une sortie est partielle)

### ✍️ Focus « Prompt pertinent » (structure simple & rigoureuse)

Utilisez un prompt en 6 parties claires :

1. **Role** — qui est le modèle ? (ex. *Product Owner expérimenté*)
2. **Mission** — transformer une initiative en User Stories (avec contraintes INVEST)
3. **Framework** — gabarit de Story (Title + Description en « As a / I want / So that »)
4. **Product Description** — injectez `{{ $json.ProductDescription }}` pour cadrer le périmètre
5. **Output Structure** — explicitez la forme attendue (ici, la liste `user_stories[]`)
6. **Tone & Style** — concision, valeur business, actionnable

> Gardez un **ton direct**, évitez le jargon, et limitez à **≤ 5 stories** pour rester *small*/*testable*.

### 🧱 Code du nœud (incluant le prompt)

```json
{
  "parameters": {
    "promptType": "define",
    "text": "=The initiative to split:\n{{ $('chat').item.json.chatInput }}",
    "hasOutputParser": true,
    "messages": {
      "messageValues": [
        {
          "message": "=## 1. Role  \nAct as an **experienced Scrum Product Owner**, specializing in **Agile backlog management, User Story writing, and SAFe Epic decomposition**.  \nEnsure that **User Stories align with the Epic’s business objectives, UX Design, UI Design, and Agile development best practices**.\n\n---\n\n## 2. Mission  \nTransform a User initiative into a **structured set of User Stories** in the context of the Product\n- **Delivery of incremental value** through well-defined User Stories.  \n- **INVEST principles** (Independent, Negotiable, Valuable, Estimable, Small, Testable).  \n- **Clarity and feasibility** for Agile development teams.  \n\n### Objectives:\n- Break down the **Initiative** into **clear, actionable User Stories** in relevance with existing product. \n- Ensure that **each story provides tangible user value**.  \n- Incorporate **UX Design insights** to optimize usability.  \n- Maintain **UI consistency** for seamless design integration.  \n- Structure the backlog **to facilitate Sprint planning and iterative delivery**.  \n\n### Limitations:\n- Do **not** generate technical specifications.  \n- Do **not** define tasks or sub-tasks—only User Stories.  \n- Do **not** introduce dependencies that prevent independent development.  \n- Do **not** exceed 5 well-structured User Stories per Initiatives.  \n\n---\n\n## 3. User Story Structuring Framework  \n\n### **User Story Format**  \nEach User Story must follow this structure:  \n\n**Title**: A concise, explicit, user-oriented summary.  \n**Description**:  \n*_As a_* [user role],  \n*_I want_* [feature or action],  \n*_So that_* [expected benefit].  \n\n### **INVEST Compliance**  \n- **Independent**: Each story should be deliverable on its own.  \n- **Negotiable**: Open to refinement based on team discussions.  \n- **Valuable**: Provides user or business value.  \n- **Estimable**: Teams can estimate effort with confidence.  \n- **Small**: Achievable within a single Sprint.  \n- **Testable**: Clear acceptance criteria ensure verification.  \n\n---\n\n## 4. Desciption of the product\n{{ $json.ProductDescription }}\n\n---\n\n## 5. Output Structure  \n\n# USER STORY DOCUMENT  \n\n## **Epic Name:** [Epic Title]  \n\n## **User Stories**  \n\n**Title**: [User Story Name]  \n**Description**:  \n_As a_ [user role],  \n_I want_ [feature or action],  \n_So that_ [expected benefit].  \n\n(Repeat for 5 User Stories)  \n\n---\n\n## 6. Tone & Style  \n- **Clear, structured, and business-value-driven**.  \n- **Simple, user-focused language**—avoiding unnecessary complexity.  \n- **Directly actionable for Agile teams**.  "
        }
      ]
    }
  },
  "type": "@n8n/n8n-nodes-langchain.chainLlm",
  "typeVersion": 1.5,
  "position": [800, 2512],
  "id": "451c56b8-fcc0-48f1-ba28-3094f1a40450",
  "name": "Write User Stories",
  "alwaysOutputData": true,
  "retryOnFail": true,
  "onError": "continueRegularOutput"
}
```

---

## 7) Node — **Split Out** (`n8n-nodes-base.splitOut`)

### 🎯 Rôle

Parcourir **chaque élément** du tableau `output.user_stories` produit par la LLM Chain.

### 🔎 Où le trouver / ajouter

- **Add node → Data transformation → Split Out**.

### ⚙️ Configuration

- **Field to split out** : `output.user_stories`
- Chaque exécution downstream recevra un item `{ title, description }`.

### 🧱 Code du nœud

```json
{
  "parameters": {
    "fieldToSplitOut": "output.user_stories",
    "options": {}
  },
  "type": "n8n-nodes-base.splitOut",
  "typeVersion": 1,
  "position": [1152, 2512],
  "id": "72ba1fcf-a970-475a-9ace-324d05dbd52f",
  "name": "Split User Stories 1"
}
```

---

## 8) Node — **Jira – Create Issue** (`n8n-nodes-base.jira`)

### 🎯 Rôle

Créer une **issue Story** dans Jira pour chaque item en sortie du `Split Out`.

### 🔎 Où le trouver / ajouter

- **Add node → Apps → Atlassian → Jira Software Cloud** (ou "Jira").

### ⚙️ Configuration

- **Resource / Operation** : *Issue → Create*
- **Project** : sélectionnez votre projet (ID/clé)
- **Issue type** : `Story`
- **Summary** : `={{ $json.title }}`
- **Additional fields → Description** : `={{ $json.description }}`
- **Custom fields** (ex. `customfield_10058: player`) : ajoutez selon vos besoins
- **Credentials** : vos identifiants **Jira SW Cloud** configurés.

> 💡 Si l’API renvoie *“field cannot be set / not on screen”*, ajoutez le champ aux **écrans** Create/Edit de votre *Story* côté Jira.

### 🧱 Code du nœud

```json
{
  "parameters": {
    "project": {
      "__rl": true,
      "value": "10033",
      "mode": "list",
      "cachedResultName": "AeS 2025"
    },
    "issueType": {
      "__rl": true,
      "value": "10005",
      "mode": "list",
      "cachedResultName": "Story"
    },
    "summary": "={{ $json.title }}",
    "additionalFields": {
      "description": "={{ $json.description }}",
      "customFieldsUi": {
        "customFieldsValues": [
          {
            "fieldId": {
              "__rl": true,
              "value": "customfield_10058",
              "mode": "list",
              "cachedResultName": "player"
            },
            "fieldValue": "TeamA"
          }
        ]
      },
      "labels": []
    }
  },
  "type": "n8n-nodes-base.jira",
  "typeVersion": 1,
  "position": [1360, 2512],
  "id": "b74a9f1b-057a-40c3-bd4a-6045e1f180a6",
  "name": "Create User Story In Jira",
  "executeOnce": false,
  "credentials": {
    "jiraSoftwareCloudApi": {
      "id": "LRm2emiUirJcPQFI",
      "name": "Jira SW Cloud Thiga"
    }
  }
}
```

---

## 9) Connexions entre les nœuds (wiring)

1. **chat → Product Desciption** : sortie *main* du Chat Trigger vers *Set*.
2. **Product Desciption → Write User Stories** : sortie *main* vers la LLM Chain (le *ProductDescription* sera disponible via `$json.ProductDescription`).
3. **gpt-4o-for-Automated-Workflow → Write User Stories** : connexion **AI Language Model**.
4. **User Stories Structured Output Parser → Write User Stories** : connexion **AI Output Parser**.
5. **Write User Stories → Split User Stories 1** : sortie *main* vers *Split Out*.
6. **Split User Stories 1 → Create User Story In Jira** : sortie *main* vers *Jira*.

> Astuce : les connexions AI (modèle & parser) se font depuis les **ports AI** de la *LLM Chain* (distincts des ports *main*).

---

## 10) Tests, validation & dépannage

### Test end-to-end

1. **Exécute** le workflow.
2. Dans **Chat**, colle une initiative :
   > « Permettre aux nouveaux utilisateurs de trouver plus vite des réponses via un avatar vocal. KPI : réduire le temps‑à‑réponse de 30%. »
3. Vérifie la sortie de **Write User Stories** → `output.user_stories` (array).
4. Vérifie que **Split Out** émet 3–5 items `{ title, description }`.
5. Confirme la création des issues dans **Jira** (projet cible, type *Story*).

### Contrôle qualité

- **INVEST** : stories *Small* & *Testable* (acceptance criteria ajoutables ensuite).
- **Clarté** : titres orientés *user value*, descriptions en *As a / I want / So that*.

### Dépannage rapide

- **Sortie non structurée** → vérifie la **connexion Output Parser** + `temperature` bas.
- **Champ Jira non modifiable** → ajoute le champ aux **Screens** *Create/Edit* de *Story*.
- **Erreurs d’auth** → regénère **API token Jira** / vérifie *Credentials* OpenAI.

---

## 11) Annexe — Bonnes pratiques de prompt (LLM)

- **Rôle** clair (ex. *Product Owner expérimenté*).
- **Mission** concise avec **contraintes** (INVEST, 3–5 stories max, pas de specs techniques).
- **Framework** de sortie (gabarit *Title + Description*).
- **Contexte produit** séparé du besoin ponctuel (variable `ProductDescription`).
- **Structure de sortie** alignée avec le **JSON Schema** (évite les ":", bullets non désirées).
- **Paramètres LLM** : `temperature` bas (0.1–0.3) pour cohérence ; limite de longueur si besoin.
- **Validation** : ajoute un nœud de **validation JSON** (optionnel) avant Jira.

---

### Flow complet (référence JSON) — *extrait*

> Les IDs de credentials/projet peuvent différer chez vous.

```json
{
  "nodes": [
    { … Chat Trigger … },
    { … Set Product Desciption … },
    { … OpenAI Chat Model … },
    { … Structured Output Parser … },
    { … LLM Chain – Write User Stories … },
    { … Split Out … },
    { … Jira – Create Issue … }
  ],
  "connections": {
    "chat": { "main": [[{ "node": "Product Desciption", "type": "main", "index": 0 }]] },
    "Product Desciption": { "main": [[{ "node": "Write User Stories", "type": "main", "index": 0 }]] },
    "gpt-4o-for-Automated-Workflow": { "ai_languageModel": [[{ "node": "Write User Stories", "type": "ai_languageModel", "index": 0 }]] },
    "User Stories Structured Output Parser": { "ai_outputParser": [[{ "node": "Write User Stories", "type": "ai_outputParser", "index": 0 }]] },
    "Write User Stories": { "main": [[{ "node": "Split User Stories 1", "type": "main", "index": 0 }]] },
    "Split User Stories 1": { "main": [[{ "node": "Create User Story In Jira", "type": "main", "index": 0 }]] }
  }
}
```

---

**Tip final** : conservez `temperature` bas et un **schema strict**, puis itérez sur le prompt *Role/Mission/Framework/Output/Tone*. Une fois stable, versionnez votre flow et partagez-le aux participants de l’atelier.





