---
titre: Panoptes — note d'index
projet: Panoptes
type: moc
date: 2026-08-30
tags: [panoptes, moc, llm-local, corpus]
---

# Panoptes — note d'index

Point d'entrée du dossier **Panoptes** : corpus documentaire isolé exploité par un LLM local, pour trois postes traitants. Enveloppe 20 000 € de matériel. Expérimentation en bac à sable sous responsabilité du chef de corps, sans homologation à ce stade.

## Notes du dossier

- [[AUG30-Panoptes-note-architecture]] — la note d'architecture complète, révision 4 : dimensionnement, budget ventilé par fonction, contrat de franchissement, cotation et faux recoupement, choix de modèles, séquencement, risques.
- [[AUG30-Panoptes-anatomie-corpus-structure]] — l'annexe conceptuelle : référentiel, taxonomie, ontologie, graphe, thésaurus, embedding. Ce que chacun veut dire, comment ils s'emboîtent, lesquels construire.
- [[AUG30-Panoptes-fiche-consigne-modele-de-donnees]] — le brief autoportant à coller pour ouvrir la conversation de modélisation, avant passage à la cellule transformation.

## Livrables en ligne

- Note d'architecture, version consultable — https://claude.ai/code/artifact/d8d2d0ab-a312-40d4-931c-50493590c49a
- Anatomie d'un corpus structuré — https://claude.ai/code/artifact/6e2db163-3ec8-4eea-8386-5f3cba2ff73a
- Maquette de l'interface traitant — https://claude.ai/code/artifact/9f34a096-0833-44ae-bcd9-904eeeaa177a

## Le dispositif en bref

La collecte est assurée par la **TF EYLAU** sur outils connectés, puis versée dans Panoptes par franchissement de rupture, sous forme de lots normalisés — contenu, manifeste de métadonnées, empreintes. Le corpus compte quatre strates de confiance décroissante : principes scientifiques, documentation technique, doctrine et RETEX, réseaux sociaux. Environ 400 documents PDF pour le froid et le tiède, plus un flux chaud non borné. Français, anglais, russe, ukrainien.

Cinq usages : questionner la pertinence d'une idée, en faire émerger une nouvelle, aider à la rédaction, donner un avis technique sur une annonce publique, produire un *debunk* technique. Les deux derniers reposent sur le même mouvement — confronter une assertion chaude à un référentiel froid.

## Décisions arrêtées

- **Matériel** : un RTX 5090 32 Go pour le modèle de dialogue, plus un petit GPU de service 16 Go pour l'embedding, le réordonnanceur et l'ASR. Châssis prévu pour deux cartes, une seule installée. RTX PRO 6000 écartée.
- **Modèle** : Qwen 3.8-27B, en Q6 plutôt qu'en Q4. Contexte de session plafonné à 32–64K.
- **Filtrage à facettes** — strate, cotation, langue, entité, dates — appliqué **avant** le classement des résultats.
- **Graphe de collecte** en deux tables PostgreSQL avec requêtes récursives. Pas de serveur graphe dédié.
- **Graphe sémantique différé**, sur déclencheur observable dans le jeu d'évaluation.
- **Ontologie formelle, RDF, SPARQL : écartés.**

## Prochaine action, irrattrapable

**Figer le format de lot avec la TF EYLAU** — enveloppe de métadonnées, grille de cotation, `origine_declaree` et `relations_collecte` — **avant** que la collecte ne soit industrialisée. Une demi-journée, gratuite. Ce qui n'y figure pas au franchissement est perdu définitivement, et sur la strate chaude la source aura disparu.

## La règle à ne pas perdre

**Le système compte les origines distinctes ; il ne cote jamais.** Il affiche « 7 documents, 2 origines après déduplication ». L'analyste cote. Un modèle qui attribue lui-même un A1 produit exactement l'erreur que le dispositif est censé prévenir.

## Questions ouvertes

1. Cadence et volume du flux chaud — seul poste non borné.
2. Qui pose la cotation, et quand : est-ce dans la pratique et l'outillage de la TF EYLAU, ou faut-il l'y introduire ?
3. Existe-t-il un glossaire ou une nomenclature en service dont partir pour le référentiel d'entités ?
