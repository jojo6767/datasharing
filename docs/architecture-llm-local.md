# Panoptes et sa couche générative — note d'architecture

*Révision 3 — 30 août 2026 — enveloppe 20 000 € TTC, périmètre complet*

> **Ce qui change en révision 3.** Le périmètre est désormais complet : les 20 000 € couvrent le stockage du corpus, l'indexation, la couche générative et la sécurisation de l'ensemble. La collecte est assurée par la **TF EYLAU** sur des outils connectés, puis versée dans **Panoptes** par franchissement de rupture. Le corpus est multimodal — texte, images, vidéos, fiches matérielles structurées, métadonnées de réseau. Trois positions sont révisées : le contrat d'interface du franchissement devient le point critique (§02), la position sur les graphes se dédouble (§05), et le second GPU sort de l'enveloppe (§07).

---

## 0. Les invariants — ce qui ne bouge plus

Trois points établis aux révisions précédentes et qui restent vrais.

**Qwen 3.8-Max reste hors d'atteinte.** 2 400 milliards de paramètres, ~1,2 To de poids en 4 bits, 150 000 à 400 000 € de matériel. La cible est **Qwen 3.8-27B** — et votre préférence pour la famille Qwen est bien fondée : voir §08, ce modèle est le meilleur choix disponible pour ce dispositif, pour des raisons qui vont au-delà de ses performances générales.

**Le marché matériel est à un pic historique.** DRAM +90 % au T1 2026 puis +58 % au T2, Apple a retiré ses options 256 et 512 Go, la RTX 5090 se négocie vers 4 200 $ contre 1 999 $ annoncés. Normalisation attendue en 2027-2028. Toute cotation a une validité de quatre semaines.

**Un seul nœud de calcul, plusieurs clients légers.** L'inférence répartie sur Ethernet ne fonctionne pas ; la puissance s'additionne dans un châssis, par le bus PCIe.

Et l'invariant de conception : **pour un usage documentaire, la latence de préremplissage prime sur la taille du modèle.** NVIDIA est 3 à 8 fois plus rapide qu'Apple Silicon sur cette phase, ce qui donne 1 à 3 secondes avant le premier mot au lieu de 40 à 70. C'est le facteur qui décide de l'adoption.

---

## 1. Le dispositif d'ensemble

```
  ══════════════ ZONE CONNECTÉE ══════════════
   ┌──────────────────────────────────────┐
   │  TF EYLAU — collecte                 │
   │  réseaux sociaux, publications,      │
   │  sites, médias                       │
   └──────────────────┬───────────────────┘
                      │
                      │  LOT NORMALISÉ
                      │  contenu + manifeste + empreintes
  ════════════════════╪═══ RUPTURE ════════════════════
                      │
  ══════════════ ZONE ISOLÉE ══════════════
   ┌──────────────────┴───────────────────┐
   │  RÉCEPTION — contrôle, quarantaine,  │
   │  vérification du manifeste           │
   └──────────────────┬───────────────────┘
                      │
   ┌──────────────────┴───────────────────────────────┐
   │  PANOPTES — le corpus et ses index               │
   │                                                  │
   │   ① documents      ② index vectoriel + lexical   │
   │      markdown         Qdrant                     │
   │      + médias                                    │
   │                                                  │
   │   ③ base relationnelle  ④ graphe de collecte     │
   │      fiches matérielles    comptes, reprises,    │
   │      catalogue             propagation           │
   └──────────────────┬───────────────────────────────┘
                      │  outils (tool-calling)
   ┌──────────────────┴───────────────────┐
   │  COUCHE GÉNÉRATIVE                   │
   │  Qwen 3.8-27B + embedding + reranker │
   │  + ASR · orchestration · garde-fous  │
   └──────────────────┬───────────────────┘
                      │
   ┌──────────────────┴───────────────────┐
   │  3 POSTES TRAITANTS — navigateur     │
   └──────────────────────────────────────┘
```

Le point important de ce schéma : **Panoptes n'est pas une base, c'est quatre magasins de natures différentes**, et la couche générative les atteint par des outils distincts, pas par un unique canal de recherche vectorielle. Voir §04.

---

## 2. Le contrat d'interface du franchissement — le point critique

C'est la décision la plus urgente de tout le dossier, et la seule qui soit vraiment irrattrapable.

La TF EYLAU collecte d'un côté, Panoptes reçoit de l'autre, et entre les deux il y a une rupture. **Tout ce qui n'est pas écrit dans le lot au moment du franchissement est définitivement perdu** : vous ne pourrez pas revenir interroger l'outil de collecte, et pour la strate chaude, la source elle-même aura souvent disparu.

Conséquence directe : le livrable de la TF EYLAU n'est pas « des fichiers ». C'est un **lot normalisé**, dont le format doit être arrêté maintenant, avant que la TF EYLAU n'outille sa collecte. Un format défini après coup impose une recollecte — c'est-à-dire, sur la strate chaude, une perte sèche.

### Structure d'un lot

```
LOT-2026-08-30-001/
├── bordereau.txt          émetteur, date, volume, périmètre, visa
├── manifeste.jsonl        une ligne = un document = l'enveloppe complète
├── empreintes.sha256      intégrité de chaque fichier
└── contenu/
    ├── doc-2026-08-30-0001.md
    ├── doc-2026-08-30-0002.jpg
    └── doc-2026-08-30-0003.mp4
```

### Une ligne de manifeste

```json
{
  "id": "doc-2026-08-30-0001",
  "fichier": "contenu/doc-2026-08-30-0001.md",
  "media_type": "texte",
  "strate": "reseau_social",
  "temperature": "chaud",
  "source_uri": "https://…",
  "source_plateforme": "…",
  "acteur": "compte ou organisme émetteur",
  "acteur_id": "ACT-0231",
  "date_pub": "2026-08-29T14:22:00Z",
  "date_collecte": "2026-08-30T06:10:00Z",
  "fiabilite_src": "C",
  "fiabilite_info": "3",
  "langue": "ru",
  "entites": ["ENT-0412"],
  "relations_collecte": [
    {"type": "reprise_de", "cible": "doc-2026-08-28-0117"},
    {"type": "publie_par", "cible": "ACT-0231"}
  ],
  "classification": "…",
  "hash": "sha256:…"
}
```

Quatre champs méritent qu'on s'y arrête, parce que ce sont eux qu'on oublie et qu'on ne récupère jamais.

**`date_pub` distincte de `date_collecte`.** Un post republié aujourd'hui mais écrit il y a trois ans n'a pas le même statut. Sans les deux dates, vous ne pouvez pas dater un fait.

**`acteur_id` et non seulement `acteur`.** Un identifiant stable, réconcilié côté TF EYLAU, permet de suivre un compte qui change de nom. Un simple nom d'affichage ne le permet pas.

**`fiabilite_src` et `fiabilite_info`.** La cotation à double entrée classique. Elle doit être posée **au moment de la collecte**, par qui connaît la source. Reconstituée six mois plus tard par quelqu'un d'autre, elle ne vaut rien. C'est deux caractères par document, et c'est ce qui rend l'usage *debunk* possible.

**`relations_collecte`.** Qui reprend qui, qui publie quoi. C'est de la donnée de graphe **native**, disponible gratuitement au moment de la collecte et irrécupérable après. Voir §05.

### Ce qu'il faut faire cette semaine

Écrire ce format, le soumettre à la TF EYLAU, et le figer d'un commun accord **avant** qu'ils n'industrialisent leur collecte. C'est une demi-journée de travail qui conditionne tout le reste. Une interface entre deux organisations est la chose la plus coûteuse à changer une fois qu'elle tourne.

---

## 3. Le corpus : quatre strates × quatre natures

Les strates gouvernent la **confiance**, les natures gouvernent le **traitement**. Les deux axes sont indépendants et il faut les traiter séparément.

| Strate | Contenu | Température | Validité | Fiabilité |
|---|---|---|---|---|
| Principes scientifiques | physique, propagation, traitement du signal | froide | décennies | très haute, vérifiable |
| Documentation technique | fiches matérielles, manuels, spécifications | froide à tiède | années | haute, traçable |
| Doctrine et RETEX | enseignements, doctrine d'emploi | tiède | mois à années | haute mais contextuelle |
| Réseaux sociaux | annonces, revendications, images, vidéos | chaude | jours | variable à nulle |

Ces quatre strates ne peuvent pas vivre dans un index indifférencié. Si un principe de propagation et une revendication non sourcée d'hier sont deux vecteurs voisins dans la même base, le modèle les traite comme deux extraits de même statut, et produit une synthèse qui mélange un invariant physique et une affirmation d'acteur — sans le signaler, parce que rien dans les données ne le lui permet.

**Pour vos usages de critique et de *debunk*, c'est l'inverse de ce que vous cherchez : l'outil deviendrait un amplificateur de rumeur avec l'accent de l'autorité.**

Vos deux usages à plus forte valeur reposent sur le même mouvement : **confronter une assertion chaude à un référentiel froid**. Le système doit pouvoir dire « ceci est revendiqué par tel compte le tel jour, coté C3 ; cela contredit tel principe établi dans telle source cotée A1 ». C'est de la mise en regard, pas de la synthèse. Et cela n'est possible que si le filtrage par strate et par cotation intervient **avant** le classement des résultats.

---

## 4. Traiter chaque nature de donnée

C'est ici que se joue la différence entre un système qui marche et une démonstration qui impressionne trois semaines.

### Texte et documents — le cas nominal

Markdown en entrée, segmentation **par titre** grâce au `heading_path` que le markdown préserve, restitution du chemin complet du titre au modèle. Un PDF converti en texte brut perd cette hiérarchie et oblige à découper à l'aveugle par nombre de caractères. Votre choix du markdown natif est un vrai gain de pertinence, à coût nul.

### Images — Qwen s'en charge

Qwen 3.8-27B embarque un encodeur visuel de 27 couches avec compréhension native de l'image. Concrètement : description automatique par lots de nuit, indexation de la description, et possibilité pour le traitant de poser une question directement sur une image. **Un seul poids à charger pour le texte et l'image** — c'est le principal argument technique en faveur de ce modèle dans votre dispositif.

### Vidéo — incluse, mais avec un traitement précis

Vous demandiez si la vidéo est « trop problématique ». Réponse : **non pour le stockage, oui si vous la traitez naïvement.**

Le piège serait de vouloir « comprendre » la vidéo de bout en bout. Le bon traitement est de l'exploiter par ses dérivés :

```
vidéo.mp4
   ├─► transcription horodatée      (Parakeet-TDT v3)
   ├─► images-clés par changement de plan  (détection de coupe)
   │      └─► description de chaque image-clé  (Qwen 3.8-27B, par lots)
   └─► la vidéo brute reste une pièce jointe, jamais dans le contexte
```

On indexe la transcription et les descriptions, chacune horodatée, ce qui permet de pointer vers l'instant précis. La vidéo elle-même n'entre jamais dans le contexte du modèle — elle est consultée par le traitant, à la seconde indiquée.

**Sur l'ASR** : Parakeet-TDT-0.6B-v3 est environ 49 fois plus rapide que Whisper large-v3 pour un taux d'erreur inférieur, ce qui compte quand on traite des heures de vidéo par lots. Sa couverture linguistique est en revanche plus étroite : **gardez Whisper large-v3 en voie de secours pour les langues qu'il ne couvre pas** — à vérifier contre vos langues cibles réelles avant de figer la chaîne.

**Sur le stockage** : la vidéo représentera environ 90 % du volume pour une fraction marginale de la valeur analytique par octet. C'est une raison de lui appliquer une **politique de rétention** distincte du reste — les dérivés textuels sont légers et se gardent indéfiniment, la vidéo brute peut être purgée au-delà d'une fenêtre.

### Fiches matérielles — surtout pas dans l'index vectoriel

C'est l'erreur la plus coûteuse et la plus fréquente.

Une fiche matérielle est une **table**. À la question « quelle est la bande de tel matériel », le système doit **lire une ligne**, pas retrouver un fragment de texte qui parle de bandes. La recherche vectorielle sur des données tabulaires donne des réponses plausibles et fausses : elle ramène la fiche d'un matériel voisin, ou une valeur d'une autre colonne.

Le bon traitement : une **base relationnelle**, et un outil que le modèle appelle pour l'interroger. Le modèle formule la requête, lit le résultat, et le cite. C'est déterministe, vérifiable, et incomparablement plus fiable.

### Métadonnées de réseau — de la donnée de graphe native

Voir §05 : c'est ce qui fait évoluer ma position.

---

## 5. Les deux graphes — position à nouveau révisée

En révision 2, je vous disais : le graphe est un artefact dérivé, faites le schéma et le référentiel maintenant, différez le moteur. Vous m'aviez opposé un risque de dette technique. La réponse tenait à la distinction entre données primaires et artefacts dérivés.

**Votre indication que Panoptes reçoit des métadonnées de réseau change cette réponse à moitié.** Il n'y a pas un graphe, il y en a deux, de natures opposées.

| | Graphe de collecte | Graphe sémantique |
|---|---|---|
| Nœuds | comptes, publications, médias | matériels, acteurs, programmes, concepts |
| Arêtes | publie, reprend, cite, répond | emploie, contredit, dérive de, s'observe avec |
| Origine | **native** — capturée à la collecte | **dérivée** — extraite des documents par le LLM |
| Coût de constitution | quasi nul, c'est du transport | élevé, extraction + validation |
| Si omis au départ | **perte définitive** | recalculable |
| Décision | **faire dès le jour 1** | **différer, sur déclencheur** |

**Le graphe de collecte se fait maintenant.** Il ne s'extrait pas, il se transporte : les relations « publié par », « reprise de », « en réponse à » sont connues de la TF EYLAU au moment de la collecte et disparaissent ensuite. Les stocker dans une base graphe plutôt que dans des colonnes coûte le prix du bon choix de magasin, rien de plus. Et il porte directement des questions que vous voudrez poser : qui a lancé cette affirmation, par quelle chaîne s'est-elle propagée, quels comptes reprennent systématiquement quels autres.

**Le graphe sémantique reste différé.** Il exige extraction d'entités et de relations, schéma, résolution d'entités, maintenance. Le mode d'échec dominant de ces projets est de figer un schéma de relations à l'aveugle sur un corpus qu'on n'a pas encore vu, puis de découvrir que les relations utiles n'étaient pas celles-là.

Ce qui se fait en revanche dès la phase 1, et qui coûte peu : **le référentiel d'entités**. Une table plate, les 100 à 200 entités qui comptent dans votre domaine, avec leurs alias. Chaque document ingéré est étiqueté contre elle. Cela vous donne immédiatement la recherche filtrée par entité, et cela pose les nœuds du graphe sémantique futur. Quand vous déciderez de le construire, il ne manquera que les arêtes — semaines de travail au lieu de mois.

**Le déclencheur** : quand votre jeu d'évaluation contient une classe de questions à sauts multiples que la recherche filtrée échoue systématiquement à traiter. D'où l'intérêt d'y mettre cinq questions de ce type dès le départ, comme détecteur.

---

## 6. Comment le modèle atteint les données

Panoptes ayant quatre magasins, l'assistant ne doit pas avoir un seul canal de recherche mais **un jeu d'outils**. Qwen 3.8-27B est bon sur l'appel d'outils, c'est ce qui rend ce schéma praticable.

| Outil | Ce qu'il fait | Magasin |
|---|---|---|
| `chercher_documents` | recherche hybride, filtrée par strate, fiabilité, entité, fenêtre de dates | index vectoriel + lexical |
| `interroger_fiches` | requête structurée sur les caractéristiques matérielles | base relationnelle |
| `explorer_reseau` | qui publie, qui reprend, chaîne de propagation | graphe de collecte |
| `lire_document` | restitution intégrale d'un document identifié | fichiers |
| `chercher_media` | recherche sur descriptions d'images et transcriptions | index, filtré sur média |

L'intérêt dépasse la propreté architecturale. Un modèle qui appelle `interroger_fiches` et cite une ligne de table est **vérifiable** ; un modèle qui paraphrase un fragment retrouvé par similarité ne l'est pas. Pour un usage de *debunk*, cette différence est la valeur du système.

**Règle de restitution, à imposer dans le prompt système** : tout extrait présenté au modèle arrive avec sa strate et sa cotation, et toute réponse restitue la cotation avec la citation. Un extrait coté C3 ne doit jamais apparaître dans une réponse comme un fait établi.

---

## 7. Ventilation budgétaire — 20 000 € TTC, périmètre complet

Vous m'avez dit ne pas comprendre mon découpage précédent. Il était fait par scénario matériel ; celui-ci est fait par **fonction**, et il couvre tout ce que vous avez énuméré.

| # | Fonction | Ce que ça paie | Montant |
|---|---|---|---|
| 1 | **Nœud de calcul** | GPU de dialogue, GPU de service, plateforme, NVMe | 8 000 – 9 800 € |
| 2 | **Stockage du corpus** | NAS 4 baies + disques, ~30 To utiles | 2 200 – 2 800 € |
| 3 | **Sauvegarde** | second jeu, règle 3-2-1, coffre | 1 200 – 1 800 € |
| 4 | **Postes traitants ×3** | clients légers reconditionnés, écrans, périphériques | 1 900 – 2 500 € |
| 5 | **Réseau et réception** | switch managé, câblage plancher, poste de réception du lot | 1 400 – 1 900 € |
| 6 | **Énergie** | onduleur 2200 VA | 600 – 900 € |
| 7 | **Divers et marge** | câbles, rails, étiquetage, aléas | 1 500 – 2 000 € |
| | **Total** | | **16 800 – 21 700 €** |

### Détail du nœud de calcul

| Poste | Montant |
|---|---|
| 1 × RTX 5090 32 Go — modèle de dialogue | 3 500 – 4 000 € |
| 1 × GPU de service 16 Go — embedding, reranker, ASR | 600 – 900 € |
| Plateforme bi-GPU : CPU 16c, 128 Go DDR5, carte mère double PCIe espacé, alimentation 1600 W, boîtier, refroidissement | 3 500 – 4 500 € |
| NVMe 4 To — poids des modèles, index chaud | 400 – 600 € |

### Les deux décisions à retenir de ce tableau

**Le second RTX 5090 ne rentre pas.** Avec le périmètre élargi au stockage, à la sauvegarde et à la sécurisation, l'enveloppe ne finance plus 64 Go de VRAM. C'est le principal effet de la clarification de périmètre, et c'est une bonne nouvelle déguisée : le goulot d'étranglement de votre dispositif n'est pas la VRAM, il est dans la structuration du corpus et dans la chaîne de traitement des médias. La plateforme reste dimensionnée pour deux cartes — le surcoût est de 800 à 1 300 € — et vous ajouterez la seconde plus tard, probablement moins cher.

**Le petit GPU de service est le meilleur euro du dossier.** Une carte 16 Go à 600–900 € accueille en permanence l'embedding, le réordonnanceur et l'ASR. Elle libère les 32 Go du 5090 pour le seul modèle de dialogue et son cache d'attention — ce qui résout le problème de mémoire résidente identifié en révision 2, pour un cinquième du prix d'un second 5090. C'est de l'asymétrie assumée : une grosse carte pour ce qui demande de la latence, une petite pour ce qui tourne en fond.

### Sur le stockage

Ordres de grandeur, pour que le chiffre ne soit pas arbitraire :

| Nature | Volume typique | Part du stockage |
|---|---|---|
| Documents markdown | 1 M documents ≈ 20 Go | négligeable |
| Index vectoriel | 1 M fragments ≈ 1–4 Go | négligeable |
| Images | 100 000 images ≈ 100–500 Go | modéré |
| **Vidéo brute** | **1 000 heures ≈ 600–3 000 Go** | **~90 %** |
| Dérivés vidéo | transcriptions, images-clés | négligeable |

30 To utiles laissent une marge confortable pour démarrer. C'est la vidéo qui décidera de la trajectoire, d'où l'importance d'une politique de rétention arrêtée tôt.

---

## 8. Les modèles — Qwen au centre, et pourquoi

Votre préférence est fondée, et pour ce dispositif elle l'est plus qu'ailleurs.

| Rôle | Modèle | Empreinte | Justification |
|---|---|---|---|
| **Dialogue et vision** | **Qwen 3.8-27B** | ~18 Go en 4 bits | 27B dense, Apache 2.0, encodeur visuel de 27 couches, compréhension native image **et** vidéo, contexte 262K. Classé 52 à l'Intelligence Index d'Artificial Analysis et en tête des modèles de raisonnement de 4 à 40B. **Un seul poids couvre le texte et l'image de votre corpus** — c'est l'argument décisif ici. |
| Extraction par lots | Qwen 3 à 8B | ~5 Go | Étiquetage d'entités, cotation assistée, résumé d'ingestion. N'a pas besoin du grand modèle et traite bien plus de documents par heure. |
| Embeddings | Qwen3-Embedding ou BGE-M3 | ~2 Go | Multilingues. BGE-M3 produit dense, lexical et multi-vecteurs dans un seul modèle, ce qui donne l'hybride sans deux systèmes. |
| Réordonnancement | bge-reranker-v2-m3 | ~1 Go | Sur un corpus hétérogène, empêche un post de réseau social de passer devant un manuel technique. |
| Transcription | Parakeet-TDT-0.6B-v3 | ~2 Go | ~49× Whisper large-v3 pour un WER inférieur. Décisif sur du volume vidéo. |
| Transcription, secours | Whisper large-v3 | ~3 Go | 99+ langues, pour ce que Parakeet ne couvre pas. |

Deux remarques.

La **fenêtre de 262K** est un argument commercial, pas un régime de croisière. Plafonnez le contexte de session à 32K–64K : au-delà, la latence de préremplissage remonte et la qualité de la réponse ne suit pas.

Sur la **provenance chinoise des poids** : je l'avais signalée en révision 1, vous avez tranché, je n'y reviens pas. Un seul point subsiste, factuel : si le système doit être homologué, la provenance peut devenir une contrainte externe à votre préférence. Le dispositif y est préparé — le serveur d'inférence charge un fichier de poids, quel qu'il soit, et Mistral Small 4 est un remplaçant de taille équivalente sous la même licence. C'est un fichier à changer, pas une architecture à refaire. Autant le savoir avant d'avoir à le faire.

---

## 9. Pile logicielle

| Couche | Choix | Pourquoi |
|---|---|---|
| Serveur d'inférence | **vLLM** | Traitement continu par lots — débit très supérieur sur l'enrichissement nocturne. Sert plusieurs modèles simultanément. API compatible OpenAI, consommée directement par l'interface. |
| Interface | **Open WebUI** | Multi-utilisateur, droits, historique, **citations des sources** — indispensable pour afficher la cotation à côté de chaque extrait. |
| Index vectoriel | **Qdrant** | Choisi pour le **filtrage sur payload** : filtrer par strate, cotation, entité et fenêtre de dates *avant* le classement. C'est exactement ce qu'exige un corpus à quatre strates. |
| Base relationnelle | **PostgreSQL** | Fiches matérielles, catalogue des documents, journal. Requêtable de façon déterministe par l'outil `interroger_fiches`. |
| Graphe de collecte | **Neo4j Community** ou extension graphe de PostgreSQL | Si le volume de relations reste modeste, l'extension PostgreSQL évite d'exploiter un serveur de plus. À trancher sur la volumétrie réelle. |
| Orchestration | **LlamaIndex** | Le filtrage par strate et le jeu d'outils demanderont du code propre assez vite. |
| Chaîne médias | ffmpeg, détection de coupe, Parakeet | Extraction d'images-clés et transcription, par lots. |
| Conteneurisation | **Docker Compose** | Un fichier décrit toute la pile ; sauvegarde et restauration triviales. |

### Contraintes de zone isolée

Téléchargement hors ligne des poids et des images conteneur — à faire transiter par le même canal que les lots, avec la même rigueur. Miroir de paquets local. Chiffrement intégral des disques du nœud et du NAS. **Procédure de restauration documentée et testée**, pas seulement écrite.

**Journalisation des requêtes** : nécessaire, mais prévenez les utilisateurs. Un outil dont on découvre après coup qu'il est journalisé perd la confiance de ses utilisateurs, et un traitant qui se méfie de l'outil ne lui pose plus les vraies questions.

---

## 10. Séquencement

### Cette semaine — avant tout achat

**Figer le format de lot avec la TF EYLAU** (§02). C'est une demi-journée, c'est gratuit, et c'est la seule chose du dossier qui soit vraiment irrattrapable. Tout le reste peut attendre le matériel.

### Phase 1 — 0 à 4 mois

1. Monter le nœud, vLLM, Open WebUI, Qdrant, PostgreSQL.
2. **Amorcer le référentiel d'entités** : les 100 à 200 entités qui comptent, avec leurs alias.
3. Ingérer d'abord les **strates froides** — principes, documentation technique, fiches matérielles. Elles sont stables, propres, et constituent le référentiel de confrontation dont les usages *debunk* auront besoin. On ne peut pas contredire une rumeur sans référentiel.
4. Câbler les outils `chercher_documents`, `interroger_fiches`, `lire_document`.
5. **Écrire le jeu d'évaluation** — 30 à 50 questions avec réponses attendues.
6. Recevoir un premier lot réel de la TF EYLAU et vérifier que le manifeste tient à l'usage. C'est le moment où l'on découvre les champs manquants, tant qu'il est encore temps.
7. Faire tester par un traitant volontaire.

### Phase 2 — 4 à 12 mois

Chaîne médias — ASR, images-clés, descriptions. Ingestion des strates tièdes puis chaudes. Graphe de collecte alimenté depuis les `relations_collecte` des manifestes. Décision sur le graphe sémantique, sur observation du jeu d'évaluation. Seconde carte si la charge par lots la justifie.

### Le jeu d'évaluation — structure proposée

C'est l'étape que tout le monde saute et celle qui décide de tout : sans elle, vous ne saurez jamais si un changement améliore ou dégrade le système. Répartition suggérée :

| Type | Nombre | Ce qu'il teste |
|---|---|---|
| Fait technique vérifiable | 10 | Le système lit-il la bonne ligne de la bonne fiche ? |
| Principe scientifique | 5 | Restitue-t-il correctement un invariant, sans le déformer ? |
| Confrontation chaud/froid | 10 | **Le cœur du dispositif.** Sait-il opposer une revendication à un principe, avec les deux cotations ? |
| Question à sauts multiples | 5 | **Détecteur de graphe.** Échoue-t-il systématiquement ? |
| Synthèse rédactionnelle | 5 | Fidélité aux sources, citations correctes |
| Question piège | 5 | Absence de réponse dans le corpus : dit-il qu'il ne sait pas, ou invente-t-il ? |

Les cinq dernières sont les plus importantes à écrire et les plus souvent oubliées. Un système qui invente proprement est plus dangereux qu'un système qui ne répond pas.

### Sur l'accompagnement bénévole

Gratuit ne veut pas dire illimité, et c'est justement pourquoi il faut choisir où le dépenser. Un appui bénévole est irrégulier et peut s'interrompre sans préavis. Mettez-le sur ce que vous ne pourrez pas refaire : le format de lot, le référentiel, la discipline d'ingestion. Le socle technique est documenté, reproductible, rattrapable seul ; la structuration du corpus ne l'est pas.

**Priorisez par irréversibilité, pas par difficulté.**

Et identifiez **une personne en interne** — pas un expert, quelqu'un de méthodique — comme référent : réception des lots, tenue du référentiel, surveillance des sauvegardes, évolution du jeu d'évaluation. Un demi-jour par semaine en régime de croisière. Sans ce rôle, le corpus vieillit en silence.

---

## 11. Risques

| Risque | Probabilité | Effet | Parade |
|---|---|---|---|
| **Lots reçus sans métadonnées complètes** | **Élevée sans contrat** | **Perte définitive de traçabilité et de cotation** | Figer le format de lot avant l'industrialisation de la collecte |
| Injection indirecte par la strate chaude | Élevée | Manipulation des réponses | Trois règles ci-dessous |
| Fiches matérielles vectorisées | Élevée si non traitée | Réponses plausibles et fausses sur des caractéristiques | Base relationnelle + outil dédié |
| Strates mélangées dans un index unique | Élevée sans filtrage | Rumeur restituée avec l'autorité d'un principe | Filtrage par strate et cotation avant classement |
| Absence de jeu d'évaluation | Très élevée | Qualité impilotable | 30 à 50 questions dès la phase 1 |
| Vidéo traitée naïvement | Moyenne | Chaîne coûteuse, stockage saturé, faible rendement | Indexer les dérivés, rétention distincte sur la brute |
| Confiance excessive dans les réponses | Élevée | Erreur d'analyse propagée | Cotation restituée avec chaque citation ; questions pièges dans l'évaluation |
| Provenance des poids et homologation | À déterminer | Modèle à changer en cours de route | Pile agnostique, Mistral Small 4 en remplaçant identifié |
| Appui bénévole interrompu | Moyenne | Chantier à l'arrêt | Dépenser l'appui sur l'irréversible d'abord |
| Prix matériel volatils | Certaine | Devis périmé en quatre semaines | Validité courte, marge de 10 %, seconde carte différée |

### Sur l'injection indirecte

La TF EYLAU collecte du contenu public, qui traverse la rupture et finit lu par un modèle qui répond à vos traitants. C'est un chemin complet entre un attaquant et votre système. Du texte publié, rédigé pour être ingéré, peut porter des instructions destinées au modèle — et dans votre domaine, l'adversaire sait que vous observez. La rupture protège le réseau, elle ne protège pas le modèle : elle laisse passer le contenu, et c'est le contenu qui porte l'attaque.

Trois règles couvrent l'essentiel, et elles coûtent peu si elles sont prévues dès la conception :

1. **Le contenu ingéré est une donnée, jamais une instruction.** Les extraits arrivent au modèle dans une enveloppe explicitement marquée comme citation non fiable, jamais concaténés au message système.
2. **Le lot atterrit en quarantaine, pas dans l'index de production.** Promotion après contrôle — automatique pour les strates froides, avec revue humaine pour la strate chaude, au moins au démarrage.
3. **La chaîne d'ingestion n'a aucun droit de lecture sur le corpus de production.** Un traitement compromis par le contenu qu'il lit ne doit pas pouvoir en extraire autre chose.

---

## 12. Points restant à clarifier

1. **Volume et cadence des lots** — combien de documents à l'amorçage, combien par semaine ensuite, et quelle part de vidéo ? C'est ce qui dimensionne le stockage et décide de la seconde carte.
2. **Langues cibles réelles** — pour arbitrer Parakeet contre Whisper sur la chaîne de transcription.
3. **Grille de cotation** — en existe-t-il une en vigueur à reprendre, ou faut-il la définir ? Elle doit figurer au format de lot, donc être arrêtée cette semaine.
4. **Politique de rétention de la vidéo brute** — conservation indéfinie ou fenêtre glissante ? Cela change la trajectoire de stockage.
5. **Volumétrie du graphe de collecte** — quelques milliers de relations, ou quelques millions ? Cela tranche entre l'extension PostgreSQL et un serveur graphe dédié.
6. **Cadre d'homologation** — le système doit-il être homologué, à quel niveau ? Contraint possiblement la provenance des modèles et impose des délais à anticiper.
7. **Local d'accueil** — circuit électrique et climatisation disponibles pour un nœud qui montera à ~1 200 W si la seconde carte est ajoutée ?
