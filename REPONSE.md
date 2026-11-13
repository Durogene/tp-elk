
```markdown
# Compte-rendu TP OpenSearch

---

## Partie 1 : Gestion de données simples (Comptes bancaires)

```text
// Ajout du compte 10001
PUT /bank/_doc/10001
{
  "account_number": 10001,
  "balance": 25000,
  "firstname": "Judyanne",
  "lastname": "Durogene",
  "age": 24,
  "gender": "F",
  "address": "50 Boulevard Chave",
  "employer": "CMA CGM",
  "email": "judyanne.durogene@mail.com",
  "city": "Marseille",
  "state": "FR"
}

// Mise à jour de l'adresse
POST /bank/_update/10001
{
  "doc": {
    "address": "120 Rue de la République"
  }
}

// Suppression du compte
DELETE /bank/_doc/10001

// Suppression des comptes de Nicholson
POST /bank/_delete_by_query
{
  "query": {
    "match": {
      "city": "Nicholson"
    }
  }
}

// Obtenir le compte ID 2
GET /bank/_doc/2

// Obtenir uniquement le _source
GET /bank/_source/2

// Obtenir uniquement firstname et lastname
GET /bank/_doc/2?_source_includes=firstname,lastname

// Retrouver tous les comptes
GET /bank/_search
{
  "query": {
    "match_all": {}
  }
}

// Retrouver les comptes de Belvoir
GET /bank/_search
{
  "query": {
    "match": {
      "city": "Belvoir"
    }
  }
}

// Comptes de Belvoir ET employeur Xurban
GET /bank/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "city": "Belvoir" } },
        { "match": { "employer": "Xurban" } }
      ]
    }
  }
}
```

### Visualisations et Dashboard
Création de l'index pattern `bank` et de trois visualisations (somme des soldes, moyenne par état, nuage de mots des villes) assemblées dans un Dashboard.

---

## Partie 2 : Recherche plein texte (Shakespeare)

```text
// Mot KING dans text_entry OU play_name (poids x2)
GET /shakespeare/_search?q=text_entry:KING OR play_name:KING^2

// L'orateur CAESAR parle de Brutus
GET /shakespeare/_search?q=speaker:CAESAR AND text_entry:Brutus

// L'orateur CAESAR ne parle PAS de Brutus
GET /shakespeare/_search?q=speaker:CAESAR AND NOT text_entry:Brutus

// Recherche par défaut (OR implicite)
GET /shakespeare/_search?q=caesar brutus calpurnia

// Recherche stricte (AND)
GET /shakespeare/_search?q=caesar AND brutus AND calpurnia

// Faute d'orthographe (0 résultat)
GET /shakespeare/_search?q=cesar

// Recherche floue avec tilde
GET /shakespeare/_search?q=cesar~
```

* **Classement de `caesar brutus calpurnia` :** L'opérateur implicite est `OR`. Le score BM25 favorise les répliques courtes contenant des termes rares. L'ajout de `AND` isole le document contenant les trois mots.
* **Rôle de `~` :** `cesar` ne renvoie rien (terme inexistant). `cesar~` active la recherche floue (distance de Levenshtein) et tolère l'erreur orthographique.

### Agrégations sur les pièces

```text
GET /shakespeare/_search
{
  "size": 0,
  "aggs": {
    "total_pieces": {
      "cardinality": {
        "field": "play_name.keyword"
      }
    },
    "lignes_par_piece": {
      "terms": {
        "field": "play_name.keyword",
        "size": 50
      },
      "aggs": {
        "nb_lignes": {
          "value_count": {
            "field": "line_id"
          }
        }
      }
    }
  }
}
```

---

## Partie 3 : Recherche sémantique

### 1. Mise en place du modèle et de l'infrastructure

```text
// 1. Autoriser le téléchargement du modèle via URL
PUT _cluster/settings
{
  "persistent": {
    "plugins.ml_commons.allow_registering_model_via_url": true
  }
}

// 2. Créer un groupe de modèles
POST /_plugins/_ml/model_groups/_register
{
  "name": "sentence_transformers",
  "description": "Groupe pour les modèles sentence transformers"
}

// 3. Enregistrer le modèle (remplacer <group_id> par l'ID retourné)
POST /_plugins/_ml/models/_register
{
  "name": "huggingface/sentence-transformers/all-MiniLM-L6-v2",
  "version": "1.0.1",
  "model_group_id": "<group_id>",
  "model_format": "TORCH_SCRIPT"
}

// 4. Déployer le modèle (remplacer <model_id> par l'ID retourné)
POST /_plugins/_ml/models/<model_id>/_deploy

// 5. Créer la pipeline d'ingestion
PUT _ingest/pipeline/news-nlp-pipeline
{
  "description": "Pipeline calculant l'embedding sur le texte du champ 'short_description'",
  "processors" : [
    {
      "text_embedding": {
        "model_id": "<model_id>",
        "field_map": {
           "short_description": "text_embedding"
        }
      }
    }
  ]
}

// 6. Créer l'index 'news' avec le mapping knn_vector
PUT /news
{
  "settings": {
    "index.knn": true,
    "default_pipeline": "news-nlp-pipeline"
  },
  "mappings": {
    "properties": {
      "text_embedding": {
        "type": "knn_vector",
        "dimension": 384,
        "method": {
          "name": "hnsw",
          "space_type": "l2",
          "engine": "lucene",
          "parameters": {
            "ef_construction": 128,
            "m": 24
          }
        }
      },
      "link": { "type": "keyword" },
      "headline": { "type": "text" },
      "category": { "type": "keyword" },
      "short_description": { "type": "text" },
      "authors": { "type": "keyword" },
      "date": { "type": "date" }
    }
  }
}
```

### 2. Requêtes sémantiques

```text
// Recherche "Usage of artificial intelligence in medicine"
GET /news/_search?size=10
{
  "query": {
    "neural": {
      "text_embedding": {
        "query_text": "Usage of artificial intelligence in medicine",
        "model_id": "<model_id>",
        "k": 10
      }
    }
  },
  "fields": ["headline", "short_description", "date", "link"],
  "_source": false
}

// Recherche d'articles similaires (début de l'article sur les documentaires)
GET /news/_search?size=10
{
  "query": {
    "neural": {
      "text_embedding": {
        "query_text": "Netflix, to its credit, has a phenomenal archive of documentaries available to watch whenever you want. Even better, they have some really interesting comedy-related documentaries available to watch whenever you want.",
        "model_id": "<model_id>",
        "k": 10
      }
    }
  },
  "fields": ["headline", "short_description"],
  "_source": false
}
```

```