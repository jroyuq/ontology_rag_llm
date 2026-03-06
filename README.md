# Projet Graph-RAG : IA Juridique augmentée par Ontologie

Ce projet démontre comment améliorer les réponses d'un modèle de langage (LLM) en utilisant **Neo4j** et une **ontologie**. L'objectif est de permettre à l'IA de naviguer dans les structures complexes des lois (clauses, définitions, sous-sections) qu'un système RAG classique ne pourrait pas interpréter correctement.

## 1. Prérequis

* **Python** : 3.10+ 
* **Ollama** : Téléchargeable sur [ollama.com](https://ollama.com). Modèle requis : `ollama run mistral`.
* **Neo4j Desktop** : Version 5.26 recommandée pour une compatibilité optimale avec l'extension n10s.

## 2. Configuration de l'instance Neo4j

Avant de lancer la base de données, suivez ces étapes de configuration :

* **Création de l'instance** : Dans Neo4j Desktop, cliquez sur "Create Instance".
* **Version** : Choisissez la version **5.26.0**.
* **Import des données** : Avant de valider la création, utilisez l'option "**Load from .dump**" en bas de la fenêtre et sélectionnez le fichier `legislation.dump` fourni.
* **Plugins** : Téléchargez et placez les fichiers `.jar` Neosemantics (n10s) dans le "load from" avec le .dump. Si jamais vous avez déjà fait "create", déposez simplement votre fichier `neosemantics-5.26.0.jar` dans `C:\...\.Neo4jDesktop2\Data\dbmss\votrebase\plugins`.
* Une fois l'instance créée, allez dans l'onglet **Plugins** (via les 3 points à droite de votre base) et assurez-vous d'installer également **APOC**.

## 3. Configuration critique (neo4j.conf)

Pour autoriser l'utilisation des procédures étendues, vous devez modifier le fichier de configuration :

* Ouvrez le fichier `neo4j.conf` (situé dans le dossier `conf` de votre base).
* Recherchez les lignes concernant les procédures autorisées (vers la ligne 690).
* Modifiez les lignes comme suit pour inclure **n10s** :

```conf
dbms.security.procedures.unrestricted=apoc.*,n10s.*
dbms.security.procedures.allowlist=apoc.*,n10s.*
```

## 4. Initialisation sémantique

Lancez l'instance Neo4j et ouvrez l'outil **Query** (Neo4j Browser). Exécutez ensuite les commandes suivantes dans l'ordre :

### A. Initialiser n10s
```cypher
// Création de la contrainte d'unicité pour les ressources RDF
CREATE CONSTRAINT n10s_unique_uri FOR (r:Resource) REQUIRE r.uri IS UNIQUE; 

// Initialisation de la configuration du graphe n10s
CALL n10s.graphconfig.init();
```

### B. Charger la structure sémantique (Inline Turtle)
Cette commande définit les relations logiques (:CONTAINS, :INCLUDES) indispensables à la navigation de l'IA. Elle permet de lier les définitions aux clauses correspondantes.

```cypher
CALL n10s.rdf.import.inline("
@prefix : [http://goingmeta.live/onto/gm24/legislation/](http://goingmeta.live/onto/gm24/legislation/) .
@prefix owl: [http://www.w3.org/2002/07/owl#](http://www.w3.org/2002/07/owl#) .
@prefix rdf: [http://www.w3.org/1999/02/22-rdf-syntax-ns#](http://www.w3.org/1999/02/22-rdf-syntax-ns#) .
@prefix rdfs: [http://www.w3.org/2000/01/rdf-schema#](http://www.w3.org/2000/01/rdf-schema#) .
@base [http://goingmeta.live/onto/gm24/legislation/](http://goingmeta.live/onto/gm24/legislation/) .

[http://goingmeta.live/onto/gm24/legislation/](http://goingmeta.live/onto/gm24/legislation/) rdf:type owl:Ontology ;
    owl:imports [http://www.nsmntx.org/2024/01/rag](http://www.nsmntx.org/2024/01/rag) .

:CONTAINS rdf:type owl:ObjectProperty ;
    rdfs:subPropertyOf [http://www.nsmntx.org/2024/01/rag#furtherDetailRelationship](http://www.nsmntx.org/2024/01/rag#furtherDetailRelationship) ;
    rdfs:domain :Definition ;
    rdfs:range :Clause .

:INCLUDES rdf:type owl:ObjectProperty ;
    rdfs:subPropertyOf [http://www.nsmntx.org/2024/01/rag#furtherDetailRelationship](http://www.nsmntx.org/2024/01/rag#furtherDetailRelationship) .

:Clause rdf:type owl:Class .
:Definition rdf:type owl:Class .
", "Turtle");
```

### 5. Validation du fonctionnement (Tests)
Pour vérifier que l'importation et la configuration sont correctes, effectuez les tests suivants :

Test 1 : Vérification de la base (Cypher)
```cypher
// Doit retourner environ 460 nœuds au total
MATCH (n) RETURN count(n) AS total_nodes;

// Doit retourner 74 nœuds indexables (Embeddable) par l'IA
MATCH (n:Embeddable) RETURN count(n) AS entry_points;
```

Test 2 : Vérification de la récupération
```cypher
// Test de récupération de la définition pour le terme spécifique
MATCH (n:Embeddable) 
WHERE n.term CONTAINS 'unavailable deposit' 
RETURN n.definition
```
Note pour l'utilisateur : Si la recherche ne renvoie aucun résultat, vérifiez que l'index vectoriel a bien été créé sur le label :Embeddable et la propriété definition.

### 6. Exécution du Notebook Python
### A. Installation des bibliothèques nécessaires

```python
pip install neo4j langchain-neo4j langchain-ollama langchain-community
```

### B. Configuration du Notebook
Ouvrez le fichier Ontology_Driven_RAG_patterns.ipynb et configurez vos accès Neo4j dans la cellule dédiée :

```python
# À MODIFIER DANS LE CODE :
os.environ["NEO4J_URL"] = "bolt://localhost:7687"
os.environ["NEO4J_USERNAME"] = "neo4j"
os.environ["NEO4J_PASSWORD"] = "VOTRE_MOT_DE_PASSE"
```

### C. Points d'attention lors de l'exécution
Modèle LLM : Assurez-vous qu'Ollama est lancé avec mistral (ollama run mistral).

Index Vectoriel : Le code créera l'index legislation lors de la première exécution.

Navigation Graphe : Le notebook utilise l'ontologie injectée pour naviguer de la définition vers les clauses spécifiques (a, b, c).

### 7. Validation finale
Si la configuration est correcte, une question sur un dépôt indisponible ("What is an unavailable deposit and what are the conditions a, b and c?") permettra à l'IA de lister précisément les conditions extraites du graphe grâce à l'ontologie.









