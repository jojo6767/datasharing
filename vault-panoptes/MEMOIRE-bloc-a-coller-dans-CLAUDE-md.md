# Bloc mémoire à coller dans le CLAUDE.md du vault

Ajouter le bloc ci-dessous au `CLAUDE.md` du vault, pour que les sessions futures
connaissent le dossier sans avoir à le redécouvrir. Adapter les noms de champs et
les tags aux conventions du vault si elles diffèrent.

---

## Projet Panoptes — corpus documentaire et LLM local

Corpus documentaire sur réseau isolé, exploité par un LLM local pour trois postes
traitants. Enveloppe 20 000 € de matériel. Expérimentation en bac à sable sous
responsabilité du chef de corps, sans homologation à ce stade.

Point d'entrée : `[[AUG30-Panoptes-MOC]]`.

**Décisions arrêtées, à ne pas rouvrir sans motif nouveau :**

- Matériel : un RTX 5090 32 Go pour le dialogue, plus un petit GPU de service
  16 Go pour l'embedding, le réordonnanceur et l'ASR. Châssis prévu pour deux
  cartes, une seule installée. RTX PRO 6000 écartée : ses 96 Go ne servent pas un
  modèle de 18 Go, et la plupart des gros modèles qui y tiendraient activent moins
  de paramètres par token qu'un 27B dense.
- Modèle : Qwen 3.8-27B, en Q6 plutôt qu'en Q4. Contexte de session 32–64K.
- Filtrage à facettes — strate, cotation, langue, entité, dates — appliqué avant
  le classement des résultats, jamais après.
- Graphe de collecte en deux tables PostgreSQL avec requêtes récursives. Pas de
  serveur graphe dédié : les volumes sont un ordre de grandeur sous le seuil.
- Graphe sémantique différé, sur déclencheur observable dans le jeu d'évaluation.
- Ontologie formelle, RDF, SPARQL, triplestores : écartés.
- Fiches matérielles en base relationnelle, jamais dans l'index vectoriel.

**Priorité irrattrapable :** figer le format de lot avec la TF EYLAU — enveloppe
de métadonnées, cotation source/information, `origine_declaree`,
`relations_collecte` — avant que la collecte ne soit industrialisée.

**Règle de conception structurante :** le système compte les origines distinctes,
il ne cote jamais. L'analyste cote. Un RAG naïf amplifie mécaniquement le faux
recoupement, puisque la similarité récompense la redondance.

**Principe d'arbitrage :** prioriser par irréversibilité, pas par difficulté. Ce
qui est structurel — châssis, alimentation, câblage, enveloppe de métadonnées,
référentiel — se fait bien du premier coup. Ce qui est modulaire — GPU, disques,
postes, switch — se fait au minimum et se renforce.
