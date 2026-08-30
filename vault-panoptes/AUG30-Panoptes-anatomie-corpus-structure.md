---
titre: Anatomie d'un corpus structuré
projet: Panoptes
type: annexe
date: 2026-08-30
tags: [panoptes, corpus, referentiel, ontologie, rag]
source: conversation Claude Code du 2026-08-30
---

# Anatomie d'un corpus structuré

*Annexe à la note d'architecture Panoptes*

Référentiel, taxonomie, ontologie, graphe, thésaurus, embedding : ces mots circulent ensemble et désignent des choses très différentes. Voici ce que chacun veut dire, comment ils s'emboîtent, et lesquels construire.

> **La règle de lecture.** Ces objets forment un empilement : chacun ne fonctionne que si celui du dessous existe. On ne fait pas d'ontologie sans référentiel, on ne fait pas de référentiel sans métadonnées. **L'erreur classique est de commencer par le haut** — c'est plus intellectuellement satisfaisant, et ça ne produit rien.

---

## I — Nommer

*Quels mots emploie-t-on, et comment sait-on que deux mots désignent la même chose ?*

**Glossaire** — *accessoire.* Une liste de termes avec leur définition. Écrit pour des humains, sans identifiants ni structure exploitable par une machine. C'est de la documentation, pas de l'infrastructure. → Utile pour former les traitants, ne sert pas le système.

**Nomenclature** — *existante.* Les règles de formation des noms. Pas une liste mais une grammaire : les index GRAU, les désignations OTAN obéissent à des règles de construction. Connaître la règle permet de reconnaître un nom jamais rencontré. → Elles préexistent, vous les subissez.

**Vocabulaire contrôlé** — *phase 1.* Une liste fermée de termes autorisés. On n'écrit pas ce qu'on veut, on choisit dans la liste. C'est ce qui empêche qu'un même champ contienne « brouilleur », « jammer », « EW jammer » et « brouilleur de comm ». → Les valeurs de `strate`, `type`, `langue`. Déjà en place sans le nommer.

**Référentiel d'entités** — *pièce maîtresse.* La liste des *choses* dont le corpus parle, chacune avec un identifiant stable et tous ses noms. Différence avec le vocabulaire contrôlé : les termes ne sont pas des catégories mais des individus du monde réel — ce matériel-là, cette unité-là, ce compte-là. → `ENT-0412` rassemble R-330Zh, R-330Ж, Житель, Zhitel, Jitel. Deux à trois jours de travail, le meilleur euro du projet.

**Thésaurus** — *plus tard.* Un vocabulaire contrôlé enrichi de relations : synonymie, hiérarchie (terme plus large / plus étroit), association. L'ancêtre documentaire du graphe. → Le référentiel avec ses alias en fait déjà 80 %.

**Résolution d'entités** — *continu.* Le travail qui décide que deux mentions désignent la même chose. Un processus, pas un objet : à chaque ingestion, rattacher les nouvelles mentions aux entités connues ou en créer une. → La machine propose, l'humain arbitre.

---

## II — Classer

*Comment range-t-on les choses les unes par rapport aux autres ?*

**Taxonomie** — *version légère.* Un classement en arbre : chaque chose a une place, et une seule. Simple et lisible, mais rigide : dès qu'une chose relève de deux branches, l'arbre casse ou se duplique. → Une liste plate d'une dizaine de types suffit. **Pas d'arbre profond.**

**Facettes** — *déjà en place.* Plusieurs axes de classement indépendants appliqués en même temps : type × bande × plateforme × pays × strate. Beaucoup plus souple qu'un arbre unique, et c'est presque toujours ce dont on a réellement besoin. → Strate, cotation, langue, entité, dates : **votre filtrage est déjà à facettes.** C'est le bon choix.

**Ontologie** — *à éviter.* Un modèle formel des *types* de choses, de leurs propriétés et des relations possibles, avec des règles logiques permettant d'inférer. Elle ne dit pas « voici les brouilleurs connus » mais « un brouilleur est un émetteur qui possède une bande ; donc si X emploie Y et Y est un brouilleur, alors X a une capacité de brouillage ». Un modèle du monde en logique, capable de déduire des faits non écrits. → Effort considérable pour un bénéfice non démontré à cette échelle. Une ontologie se justifie quand plusieurs organisations doivent partager un modèle commun, ou pour de l'inférence automatique auditée. Ni l'un ni l'autre ici.

---

## III — Relier

*Qu'est-ce qui tient à quoi, et par quel chemin ?*

**Graphe** — *phase 1.* Des nœuds reliés par des arêtes. Une structure de données, rien de plus. Le mot ne dit rien du contenu : un plan de métro est un graphe, un arbre généalogique aussi. → Deux tables PostgreSQL et des requêtes récursives. Pas de serveur graphe.

**Graphe de collecte** — *phase 1.* Le graphe des publications et de leurs reprises. Nœuds : documents, comptes, médias. Arêtes : publie, reprend, cite, répond. **Cette information est native** — connue au moment de la collecte, perdue ensuite. → C'est ce qui permet de compter les *origines* et non les documents, donc de détecter le faux recoupement. Non négociable.

**Graphe de connaissances** — *différé.* Un graphe dont les nœuds sont les entités du référentiel et les arêtes des relations typées par une ontologie. La combinaison des trois familles : nommer + classer + relier. D'où son coût. → Sur déclencheur : quand le jeu d'évaluation révèle des questions à sauts multiples que la recherche filtrée échoue systématiquement à traiter.

**Triplet, RDF, SPARQL** — *à éviter.* La forme canonique d'une arête — sujet, prédicat, objet — et l'écosystème bâti autour. Standard puissant, pile logicielle lourde, courbe d'apprentissage forte. → Deux colonnes dans une table font la même chose à ces volumes. Gardez le concept, laissez la technologie.

**Propriété contre relation** — *à trancher tôt.* Une propriété décrit une chose ; une relation relie deux choses. « Bande de fréquence » est une propriété tant qu'elle n'est qu'une valeur ; elle devient une relation le jour où « bande UHF » devient elle-même une entité qu'on veut interroger. → Gardez les caractéristiques en propriétés, donc dans la base relationnelle. N'en faites des entités que pour poser la question « qu'est-ce qui d'autre opère sur cette bande ».

---

## IV — Retrouver

*Comment remet-on la main sur le bon passage au bon moment ?*

**Fragment** — *phase 1.* Le morceau de document qu'on indexe. On n'indexe pas un document entier : trop gros pour le contexte, trop imprécis pour la recherche. La qualité du découpage décide de la qualité des réponses. → Découpage par titre grâce au markdown : des unités de sens, pas des comptes de caractères.

**Index lexical** — *phase 1.* Un dictionnaire inversé : quel mot apparaît dans quels fragments. Rapide, exact, totalement aveugle au sens. → Indispensable pour les désignations exactes, les numéros, les codes.

**Embedding** — *phase 1.* Une représentation numérique du sens d'un texte. Deux textes de sens voisin ont des représentations voisines, même sans partager un seul mot. → C'est ce qui permet de retrouver un passage en russe à partir d'une question en français, et de détecter les reprises traduites.

**Index vectoriel** — *phase 1.* La base qui range les embeddings et retrouve les plus proches. Sensible au sens, insensible à l'exactitude. → Qdrant, choisi pour son filtrage par métadonnées avant classement.

**Recherche hybride** — *phase 1.* Les deux index interrogés ensemble, résultats fusionnés. Le lexical rattrape ce que le vectoriel manque (une désignation rare) ; le vectoriel rattrape ce que le lexical manque (une reformulation, une traduction). Chacun couvre l'angle mort de l'autre. → Le mode nominal sur un corpus multilingue.

**Réordonnanceur** — *phase 1.* Un petit modèle qui relit les cinquante premiers résultats et les reclasse finement. La recherche est rapide et grossière, le réordonnanceur lent et précis, donc appliqué à peu de candidats. → C'est lui qui empêche un post de réseau social de passer devant un manuel technique.

**RAG** — *c'est le système.* On cherche dans le corpus, on injecte les passages trouvés dans le contexte du modèle, il répond à partir d'eux. Le modèle ne « connaît » pas le corpus : on le lui donne à lire à chaque question. → D'où l'importance de ce qu'on lui donne, et de ce qu'on lui refuse. La qualité dépend du filtrage bien plus que du modèle.

**GraphRAG** — *plus tard.* Une variante où la recherche passe aussi par le graphe. Permet de répondre aux questions de chemin — « qu'est-ce qui relie A à B » — dont la réponse n'est écrite dans aucun document isolé. → L'outil `explorer_reseau` en est une forme minimale et suffisante.

---

## V — Décrire

*Que sait-on du document lui-même, indépendamment de ce qu'il raconte ?*

**Métadonnée** — *fondation.* Une donnée sur la donnée. Pas le contenu, mais ce qu'on sait de lui : d'où il vient, quand, de qui, en quelle langue, avec quelle fiabilité. → La couche la plus basse, et la seule vraiment irrattrapable.

**Schéma** — *cette semaine.* La définition de quels champs existent et quelles valeurs ils acceptent. Sans schéma, chacun remplit ce qu'il veut, et le filtrage devient impossible. → Le format de manifeste à figer avec la TF EYLAU.

**Enveloppe, manifeste** — *cette semaine.* L'enveloppe est la fiche d'identité d'un document ; le manifeste est le bordereau qui les rassemble pour un lot. C'est ce qui traverse la rupture avec le contenu. → Ce qui n'y figure pas au franchissement est perdu définitivement.

**Provenance** — *phase 1.* D'où vient l'information et par quelle chaîne. Plus fort que « la source » : la provenance décrit le *trajet*. Un document repris quatre fois a une source immédiate et une origine, qui ne sont pas la même chose. → `origine_declaree` et `relations_collecte`, matière première de la détection de faux recoupement.

**Cotation** — *cette semaine.* Le jugement porté sur la source et sur l'information, en deux caractères indépendants. Métadonnée *évaluative* et non descriptive : elle engage celui qui la pose. → Posée à la collecte, par qui connaît la source. Reconstituée six mois plus tard, elle ne vaut rien.

---

## VI — Comment tout cela s'emboîte

```
  ┌──────────────────────────────────────────────────────────┐
  │  ONTOLOGIE — règles logiques, inférence                   │  ← à éviter
  ├──────────────────────────────────────────────────────────┤
  │  GRAPHE SÉMANTIQUE — relations typées entre entités       │  ← différé
  ├──────────────────────────────────────────────────────────┤
  │  GRAPHE DE COLLECTE — qui publie, qui reprend             │  ← phase 1
  ├──────────────────────────────────────────────────────────┤
  │  RÉFÉRENTIEL D'ENTITÉS — les choses, leurs identifiants   │  ← phase 1
  │  et tous leurs alias         ▲ résolution d'entités       │
  ├──────────────────────────────────────────────────────────┤
  │  FACETTES — strate · cotation · langue · entité · dates   │  ← phase 1
  ├──────────────────────────────────────────────────────────┤
  │  INDEX — lexical + vectoriel + réordonnanceur             │  ← phase 1
  ├──────────────────────────────────────────────────────────┤
  │  FRAGMENTS — découpage par titre                          │  ← phase 1
  ├──────────────────────────────────────────────────────────┤
  │  ENVELOPPE DE MÉTADONNÉES — schéma, provenance, cotation  │  ← cette semaine
  ├──────────────────────────────────────────────────────────┤
  │  DOCUMENTS — markdown, images, médias                     │
  └──────────────────────────────────────────────────────────┘
```

Chaque étage n'a de sens que si celui du dessous existe. L'ontologie sans référentiel ne décrit rien. Le référentiel sans enveloppe ne se rattache à rien.

**L'erreur classique.** Commencer par le haut est intellectuellement séduisant : modéliser le monde avant de le peupler donne le sentiment de maîtriser le sujet. En pratique on produit un schéma élégant que rien n'alimente, et on découvre six mois plus tard que les relations modélisées ne sont pas celles dont on avait besoin. **Les étages du haut se recalculent ; ceux du bas, non.**

### Le parcours d'un document

Un post en russe du 14 août arrive dans un lot.

1. **Enveloppe** — il reçoit son identité : identifiant, source, acteur `ACT-0231`, date de publication et date de collecte distinctes, cotation `C3`, langue `ru`. Sans cette étape, tout le reste est impossible.
2. **Résolution d'entités** — le petit modèle repère « Житель », propose un rattachement, l'humain confirme `ENT-0412`. Le document devient retrouvable par une question posée en français.
3. **Fragmentation** — le texte est découpé par titres, chaque fragment gardant le chemin complet de sa section.
4. **Indexation** — chaque fragment entre dans l'index lexical et reçoit un embedding.
5. **Facettes** — il devient filtrable : strate chaude, cotation C3, langue ru, entité ENT-0412, 14 août.
6. **Graphe de collecte** — on enregistre `reprise_de → doc-0117` et `publie_par → ACT-0231`.
7. **À la question posée** — le filtrage le retient ou l'écarte, la recherche hybride le remonte ou non, le réordonnanceur le classe, et le graphe dit qu'il **n'est pas une source indépendante**. C'est cette dernière information qui fait la valeur du dispositif, et elle vient de l'étage le plus bas.

---

## VII — Le tableau de tri

| Concept | Verdict | Quand | Pourquoi |
|---|---|---|---|
| Enveloppe de métadonnées | Indispensable | Cette semaine | Irrattrapable : ce qui n'est pas écrit au franchissement est perdu |
| Schéma et cotation | Indispensable | Cette semaine | Figurent au format de lot, à arrêter avant la collecte |
| Référentiel d'entités | Indispensable | Phase 1 — 2 à 3 j | Porte le rappel multilingue, pose les nœuds du graphe futur |
| Résolution d'entités | Indispensable | Continu | Regroupement des alias, jamais entièrement automatisable |
| Fragments, index, réordonnanceur | Indispensable | Phase 1 | C'est le RAG lui-même |
| Facettes | Indispensable | Phase 1 | Le filtrage par strate et cotation — déjà en place |
| Graphe de collecte | Indispensable | Phase 1 à 2 | Faux recoupement ; données natives et périssables |
| Taxonomie des types | Utile, léger | Phase 1 | Une liste plate d'une dizaine de types. Pas d'arbre. |
| Vocabulaire contrôlé | Utile | Phase 1 | Les valeurs autorisées des champs du manifeste |
| Thésaurus | Optionnel | Si besoin démontré | Le référentiel avec alias en fait l'essentiel |
| Graphe sémantique | Différé | Sur déclencheur | Coûteux, et les relations utiles restent inconnues |
| GraphRAG | Différé | Avec le précédent | Suppose le graphe sémantique |
| Ontologie formelle | À éviter | — | Se justifie pour partager un modèle entre organisations ou pour de l'inférence auditée |
| RDF, SPARQL, triplestore | À éviter | — | Pile lourde, gain marginal à ces volumes |

---

## VIII — Six confusions à ne pas faire

**Taxonomie ≠ ontologie** — une taxonomie range des choses dans des cases ; une ontologie définit ce qu'*est* une case et ce qu'on peut en déduire. Le second coûte dix fois le premier.

**Graphe ≠ graphe de connaissances** — un graphe est une structure ; un graphe de connaissances est un graphe dont les nœuds sont des entités identifiées et les arêtes des relations typées. Le mot « graphe » seul n'engage à rien.

**Référentiel ≠ glossaire** — un glossaire définit des *mots* pour des humains ; un référentiel identifie des *choses* pour une machine, avec un identifiant qui ne change jamais.

**Embedding ≠ recherche exacte** — un embedding trouve ce qui « parle de la même chose », il ne garantit pas de trouver un numéro de série précis. D'où la recherche hybride.

**Métadonnée ≠ contenu** — la cotation d'un document n'est pas dans son texte. Elle est portée par l'enveloppe, elle vient d'un humain, aucun modèle ne peut la reconstituer après coup.

**Structurer ≠ compliquer** — tout ce qui précède tient dans deux fichiers tabulaires et deux tables de base. La complexité vient de la discipline d'usage, pas de l'outillage.
