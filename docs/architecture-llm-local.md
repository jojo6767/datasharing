# Déploiement d'un LLM local sur site — analyse et propositions

*Note d'architecture — août 2026 — révision 2*

> **Révision 2.** Trois éléments nouveaux ont été intégrés : le budget porte sur le matériel seul (l'accompagnement est bénévole) ; la composition du corpus est précisée et s'avère structurante ; le sas d'import est traité hors périmètre, par agents. La position sur les graphes de connaissances a été révisée en conséquence — voir §07.

---

## 0. Trois corrections préalables, avant tout chiffrage

### 0.1 Qwen 3.8-Max n'est pas déployable chez vous

Qwen 3.8-Max est un modèle à **2 400 milliards de paramètres**. Alibaba en a publié les poids — il est donc *techniquement* téléchargeable, contrairement à Qwen 3.7-Max qui était strictement API. Mais l'ordre de grandeur matériel est sans rapport avec votre budget :

| Quantisation | Poids seuls | Matériel minimal | Coût indicatif |
|---|---|---|---|
| 4 bits | ~1,2 To | ~10 machines à 128 Go chaînées, ou un nœud serveur HBM | 150 000 – 400 000 € |
| 8 bits | ~2,4 To | nœud datacenter multi-GPU | > 500 000 € |

**Le bon modèle dans la même famille, c'est Qwen 3.8-27B** : dense, ~27 milliards de paramètres, contexte 256K, encodeur visuel intégré, licence Apache 2.0. Il tient dans **17 à 19 Go en 4 bits**, soit une seule carte. Les évaluations publiées le placent au niveau de modèles 10 à 15 fois plus gros.

### 0.2 Le marché matériel traverse une crise mémoire historique

- Prix contrat DRAM : **+90 à 95 % au T1 2026**, puis **+58 à 63 % au T2**.
- **Apple a retiré les options 512 Go (mars 2026) puis 256 Go (mai 2026)** du Mac Studio M3 Ultra. Le haut de gamme Apple plafonne à **96 Go**.
- La RTX 5090 se négocie autour de **4 200 $** contre 1 999 $ de tarif public initial.
- La RTX PRO 6000 Blackwell 96 Go est passée à **13 250 $**, soit +55 % en 16 mois.
- Le Mac Studio M5 est **repoussé à octobre 2026 au mieux**, à cause de la pénurie mémoire.
- Retour à la normale attendu **en 2027-2028**.

La stratégie « beaucoup de mémoire unifiée pour loger un très gros modèle » n'est plus disponible à l'achat neuf.

### 0.3 « Plusieurs ordinateurs branchés en série » ne fonctionne pas

L'inférence répartie sur Ethernet existe (llama.cpp RPC, exo, parallélisme de pipeline vLLM), mais la latence réseau entre couches du modèle détruit le débit. **Un seul nœud de calcul, plusieurs clients légers.** La puissance s'additionne dans un châssis, par le bus PCIe, pas sur un câble.

---

## 1. Ce que vous cherchez réellement à dimensionner

Vous avez formulé le besoin en nombre de paramètres. Ce n'est pas la variable qui décidera de votre satisfaction. Trois grandeurs comptent, dans cet ordre.

**1. Le temps avant le premier mot.** Une requête RAG, c'est votre question *plus* 10 000 à 30 000 tokens d'extraits injectés. Le modèle lit tout cela avant d'écrire son premier caractère. Cette phase de préremplissage est limitée par le **calcul brut**, pas par la bande passante mémoire — et c'est la faiblesse structurelle d'Apple Silicon (NVIDIA y est 3 à 8 fois plus rapide). Comptez **40 à 70 s** sur un Mac Studio chargé d'un gros modèle, contre **1 à 3 s** sur une station NVIDIA avec un 27B. C'est le facteur qui décide de l'adoption.

**2. La qualité de la recherche documentaire.** Un 27B alimenté par une recherche propre bat un 235B alimenté par une recherche approximative. Le modèle ne raisonne pas sur ce qu'on ne lui a pas donné.

**3. La taille du modèle.** Rendement décroissant, coût matériel explosif.

**Traduction : achetez la latence la plus basse sur un très bon modèle moyen, et investissez la différence dans la structuration du corpus.**

---

## 2. Le corpus est à quatre températures — c'est le fait structurant

Vous avez précisé la composition visée :

| Strate | Contenu | Température | Durée de validité | Fiabilité typique |
|---|---|---|---|---|
| **Principes scientifiques** | physique, propagation, traitement du signal | **froide** | décennies | très haute, vérifiable |
| **Documentation technique** | caractéristiques, manuels, spécifications | **froide à tiède** | années | haute, traçable |
| **Doctrine et RETEX** | enseignements, doctrine d'emploi | **tiède** | mois à années | haute mais contextuelle |
| **Réseaux sociaux** | annonces, revendications, images, rumeurs | **chaude** | jours | variable à nulle |

C'est l'information la plus importante que vous m'ayez donnée, et elle a une conséquence directe : **ces quatre strates ne peuvent pas vivre dans un index indifférencié.**

Si un principe de propagation et une revendication non sourcée publiée hier sont deux vecteurs voisins dans la même base, le modèle les traitera comme deux extraits de même statut. Il produira une synthèse qui mélange un invariant physique et une affirmation d'acteur — sans le signaler, parce que rien dans les données ne lui permet de faire la différence. Pour vos usages de critique et de *debunk*, c'est exactement l'inverse de ce que vous cherchez : **l'outil deviendrait un amplificateur de rumeur avec l'accent de l'autorité.**

La parade n'est pas un meilleur modèle. C'est une **enveloppe de métadonnées appliquée dès l'ingestion**, et un étage de recherche qui sait filtrer et pondérer dessus. Voir §04.

### Conséquence sur les usages

| Usage | Strates mobilisées | Exigence dominante |
|---|---|---|
| Pertinence d'une idée | froide + tiède | profondeur, contradiction |
| Idéation | toutes | rappel large, associations |
| Aide à la rédaction | froide + tiède | fidélité, citation |
| Avis sur une annonce publique | chaude, confrontée à froide | **fraîcheur + traçabilité** |
| Debunk technique | chaude, confrontée à froide | **cotation de source, séparation stricte** |

Les deux derniers usages — ceux qui font la valeur du système — sont précisément ceux qui exigent que la séparation des strates soit rigoureuse. Ils reposent tous deux sur le même mouvement : **confronter une assertion chaude à un référentiel froid.** Le système doit pouvoir dire « ceci est revendiqué par tel compte le tel jour ; cela contredit tel principe établi dans telle source ». C'est un travail de mise en regard, pas de synthèse.

---

## 3. Architecture logique

```
┌─────────────────────────────────────────────────────────┐
│  5. POSTES TRAITANTS — écran + clavier + navigateur     │
│     verrouillé, aucun calcul local                      │
└───────────────────────┬─────────────────────────────────┘
                        │  Ethernet Cat6a (plancher technique)
                        │  VLAN dédié, switch managé
┌───────────────────────┴─────────────────────────────────┐
│  4. INTERFACE — Open WebUI : comptes, droits,           │
│     historique, citations avec cotation de source       │
├─────────────────────────────────────────────────────────┤
│  3. ORCHESTRATION RAG                                   │
│     filtrage par strate/fraîcheur/fiabilité             │
│     → recherche hybride → réordonnancement              │
│     → construction du contexte (strates séparées)       │
├─────────────────────────────────────────────────────────┤
│  2. INDEX — Qdrant, payload = enveloppe complète        │
│     + référentiel d'entités                             │
├─────────────────────────────────────────────────────────┤
│  1. INFÉRENCE — vLLM : modèle de dialogue               │
│     + petit modèle d'extraction + embedding + reranker  │
└─────────────────────────────────────────────────────────┘
         ▲
         │
┌────────┴────────────────────────────────────────────────┐
│  0. INGESTION — sources → markdown → enveloppe →        │
│     segmentation par titres → étiquetage d'entités      │
│     (agents — hors périmètre de la présente note)       │
└─────────────────────────────────────────────────────────┘
```

Le sas d'import et la collecte agentique sont traités hors de cette note, selon votre indication. Deux points d'interface subsistent néanmoins, et ils sont contraignants : **l'enveloppe de métadonnées que la chaîne d'ingestion doit produire** (§04), et **le risque d'injection indirecte** que la collecte automatisée introduit (§10).

---

## 4. L'enveloppe de métadonnées — la vraie décision irréversible

C'est le cœur de la révision 2, et la réponse à votre inquiétude sur la dette technique.

### Enveloppe au niveau du document

```yaml
id:             doc-2026-08-19-0043     # stable, jamais réattribué
source_uri:     https://… | file://…
strate:         principe | technique | doctrine_retex | reseau_social
temperature:    froid | tiede | chaud
acteur:         auteur, organisme ou compte émetteur
date_pub:       2026-05-14              # date de la source
date_ingest:    2026-08-19              # date d'entrée au corpus
fiabilite_src:  A…F                     # cotation de la source
fiabilite_info: 1…6                     # cotation de l'information
classification: <votre grille>
langue:         fr | en | ru | …
entites:        [ENT-0412, ENT-0088]    # renvoi au référentiel
hash:           sha256                  # doublons et révisions
```

La cotation à double entrée (source / information) est le schéma classique du renseignement. Elle vous coûte deux caractères par document et elle est ce qui rend l'usage *debunk* possible : sans elle, le système ne peut pas dire « cette affirmation vient d'un compte non évalué et contredit une source technique cotée B2 ». Avec elle, c'est une simple contrainte de restitution.

### Enveloppe au niveau du fragment

```yaml
chunk_id:     doc-2026-08-19-0043#007
doc_id:       doc-2026-08-19-0043
heading_path: ["2. Architecture", "2.3 Chaîne de réception"]
entites:      [ENT-0412]
```

`heading_path` est **gratuit parce que vous passez en markdown**. C'est un bénéfice réel de votre choix, et il faut le nommer : un PDF converti en texte brut perd sa hiérarchie, ce qui oblige à segmenter à l'aveugle par nombre de caractères. Un markdown propre permet de segmenter **par titre**, donc de produire des fragments qui correspondent à des unités de sens, et de restituer au modèle le chemin complet du titre. Le gain de pertinence est important et il ne coûte rien.

### Le référentiel d'entités

Une table plate, tenue dès le premier jour :

```
ENT-0412 | <désignation> | type=capteur | alias=[…] | premiere_vue=2024-03 | statut=confirmé
ENT-0088 | <désignation> | type=acteur  | alias=[…] | premiere_vue=2023-11 | statut=hypothèse
```

Types utiles dans votre domaine : matériel, plateforme, émetteur, bande/fréquence, unité, acteur, programme, doctrine, lieu.

**C'est l'objet le plus important de tout le système**, et c'est celui qui coûte le moins cher à démarrer : un fichier tabulaire ou un ensemble de notes suffit au début. Chaque document ingéré est étiqueté contre ce référentiel. À partir de là, vous obtenez immédiatement deux choses :

1. **Une recherche filtrée** — « tout ce qui concerne ENT-0412, strates froide et tiède uniquement » — sans aucun graphe. C'est déjà un gain de pertinence considérable.
2. **La moitié du graphe, sans l'avoir construit.** Les nœuds sont là. Il ne manquera que les arêtes.

---

## 5. Topologies

### Topologie A — socle simple, un accès (pilote)

```
   ┌──────────────────────┐
   │   NŒUD DE CALCUL     │        ┌─────────────────┐
   │  Inférence + index   │───────►│  POSTE TRAITANT │
   │  + interface web     │  RJ45  │  écran+clavier  │
   └──────────┬───────────┘        └─────────────────┘
              │
   ┌──────────┴───────────┐
   │  Sauvegarde (NAS)    │
   └──────────────────────┘
```

**Périphérie, hors nœud de calcul : ~1 800 – 2 400 € TTC**
- Poste client (mini-PC reconditionné) : 250 – 350 €
- Écran 27" QHD + clavier/souris : 350 – 450 €
- Câblage Cat6a en plancher technique : 150 – 300 €
- Onduleur 1500 VA : 350 – 450 €
- NAS de sauvegarde 2 × 8 To en miroir : 700 – 900 €

*(Le poste de veille du sas relève du périmètre agents, traité ailleurs.)*

### Topologie B — étoile, trois accès

```
                     ┌──────────────────────┐
                     │   NŒUD DE CALCUL     │
                     └──────────┬───────────┘
                                │ 2.5 GbE
                     ┌──────────┴───────────┐
                     │  SWITCH MANAGÉ       │
                     │  VLAN dédié, 8 ports │
                     └───┬──────┬───────┬───┘
                  ┌──────┴─┐ ┌──┴───┐ ┌─┴──────┐
                  │ POSTE 1│ │POSTE2│ │ POSTE 3│
                  └────────┘ └──────┘ └────────┘
```

**Surcoût : ~1 400 – 1 900 € TTC** — deux postes clients avec écrans (1 100 – 1 500 €), switch managé 8 ports 2.5 GbE avec VLAN (200 – 400 €).

**Sur les trois utilisateurs simultanés.** Le plafond est conservateur : trois traitants qui utilisent l'outil dans la journée, ce n'est presque jamais trois requêtes au même instant. Le paramètre à surveiller n'est pas le nombre d'utilisateurs mais le **cache d'attention** : trois sessions à 32 000 tokens consomment plusieurs gigaoctets *en plus* des poids. Plafonnez le contexte par session à 32K–64K ; les 256K de Qwen 3.8-27B sont un argument commercial, pas un régime de croisière.

---

## 6. Deux charges, pas une — conséquence de votre plan agentique

Votre intention d'ingérer par agents change le profil de la machine. Elle porte désormais **deux charges de nature opposée** :

| | Dialogue interactif | Enrichissement par lots |
|---|---|---|
| Déclencheur | un traitant pose une question | ingestion nocturne, reprise de corpus |
| Métrique | **latence** (temps au 1er mot) | **débit** (documents/heure) |
| Concurrence | 3 sessions | 1 file saturée |
| Modèle | grand (27B) | petit suffit (3–8B) |
| Tolérance | quelques secondes | quelques heures |

Cela a trois conséquences pratiques :

1. **Séparez les modèles.** L'extraction d'entités et le résumé d'ingestion ne demandent pas le modèle de dialogue. Un modèle de 3 à 8 milliards de paramètres fait le travail à une fraction du coût, et vous pouvez en traiter beaucoup plus par heure.
2. **Séquencez.** Le lot tourne la nuit, l'interactif le jour. Une file de priorité suffit ; il n'y a pas besoin de deux machines.
3. **Prévoyez la mémoire résidente.** En régime nominal vous aurez simultanément en VRAM : le modèle de dialogue (~17-19 Go), l'embedding (~2 Go), le réordonnanceur (~1 Go), et éventuellement le modèle d'extraction (~5 Go). **32 Go deviennent justes.** C'est le principal argument nouveau en faveur de plus de VRAM.

---

## 7. Sur le graphe de connaissances — position révisée

Votre objection est juste : il y a bien un sujet de dette technique. Mais **elle ne se situe pas où vous la placez**, et cette distinction change entièrement l'arbitrage.

### Ce qui est reconstructible, et ce qui ne l'est pas

| Objet | Nature | Reconstruction si omis |
|---|---|---|
| Moteur de graphe, schéma de relations, traversées | **dérivé** | recalcul sur le corpus — jours de machine |
| Extraction de relations | **dérivée** | re-passe sur le corpus — jours de machine |
| Embeddings, index vectoriel | **dérivé** | ré-indexation — heures |
| Segmentation | **dérivée** | ré-ingestion depuis le markdown — heures |
| **Enveloppe de métadonnées** | **primaire** | **ré-ingestion depuis les sources — souvent impossible** |
| **Référentiel d'entités** | **primaire** | **retravail humain intégral — mois** |
| **Cotation de fiabilité** | **primaire** | **irrécupérable a posteriori** |

Un graphe est un **artefact dérivé**. Vous pouvez le reconstruire à volonté tant que vous avez conservé les nœuds, les identifiants et la provenance. Le reconstruire est un travail de machine, pas de migration.

En revanche, si vous ingérez 5 000 documents sans enveloppe : la date de publication d'un post supprimé depuis est perdue, le compte émetteur d'une image reprise n'est plus retrouvable, la cotation que l'analyste avait en tête au moment de la lecture n'a jamais été écrite. **Cette information-là ne se reconstitue pas.** Et pour la strate chaude, elle se dégrade en jours.

### La position révisée

> **Faites le schéma et le référentiel en phase 1. Différez le moteur de graphe et la couche de traversée.**

Ce n'est pas « pas de graphe ». C'est : construisez maintenant, à coût faible, tout ce dont le graphe aura besoin, et n'engagez le graphe lui-même que lorsque vous saurez quelles relations modéliser.

Ce que cela coûte en phase 1 : quelques jours pour figer l'enveloppe et amorcer le référentiel, puis une discipline d'ingestion. Ce que cela évite : modéliser un schéma de relations à l'aveugle, sur un corpus que vous n'avez pas encore vu, et découvrir en phase 2 que les relations utiles ne sont pas celles que vous aviez prévues — ce qui est le mode d'échec dominant des projets de graphe de connaissances.

Ce que cela vous donne entre-temps, sans graphe : la recherche filtrée par entité, strate, fraîcheur et fiabilité. C'est déjà l'essentiel du gain, et c'est ce qui rend les usages *debunk* possibles.

### Le déclencheur du passage au graphe

Ne décidez pas au calendrier, décidez sur une observation. Le graphe se justifie quand votre jeu d'évaluation contient une classe de questions que la recherche filtrée échoue **systématiquement** à traiter. En pratique, ce sont les questions à sauts multiples : « quels acteurs relient ce matériel à ce théâtre », « par quelle chaîne cette caractéristique s'est-elle propagée d'une source à l'autre », « qu'est-ce qui a changé entre ces deux RETEX ».

Quand ces questions apparaissent et échouent, vous saurez exactement quelles arêtes construire — et vos nœuds seront déjà là, étiquetés, depuis des mois. La construction devient un travail de semaines au lieu de mois.

---

## 8. Scénarios matériels

Le budget portant désormais **sur le seul équipement**, l'arbitrage matériel/prestation disparaît. Prix TTC indicatifs, **à revalider au devis** : le marché mémoire bouge de semaine en semaine.

### S1 — Station NVIDIA mono-GPU
**Nœud 7 100 – 8 900 € · avec topologie B : 10 300 – 13 200 €**

1 × RTX 5090 (32 Go GDDR7, ~1 792 Go/s).

| Poste | Coût |
|---|---|
| RTX 5090 32 Go | 3 500 – 4 000 € |
| Plateforme (CPU 16c, 128 Go DDR5, carte mère, alim, boîtier, refroidissement) | 3 000 – 4 000 € |
| Stockage 4 To NVMe + 8 To HDD | 600 – 900 € |

Premier token en 1 à 3 s. Trois utilisateurs sans difficulté. Mais 32 Go deviennent justes une fois l'embedding, le réordonnanceur et le modèle d'extraction résidents (§06).

### S1+ — Châssis bi-GPU, une seule carte installée
**Nœud 8 100 – 10 200 € · avec topologie B : 11 300 – 14 500 € — recommandé**

Identique à S1, mais la plateforme est dimensionnée pour deux cartes dès l'achat : alimentation 1600 W, carte mère à double emplacement PCIe espacé, boîtier et refroidissement adaptés. Une seule carte est installée.

Surcoût immédiat : **800 à 1 300 €**. Coût d'ajout de la seconde carte plus tard : le prix de la carte seule, sans remplacer la machine.

### S2 — Station NVIDIA bi-GPU complète
**Nœud 11 800 – 14 200 € · avec topologie B : 15 000 – 18 500 €**

2 × RTX 5090, 64 Go cumulés. Ouvre les MoE 70–80B en 4 bits et permet de garder tous les modèles résidents avec un large cache d'attention.

**Contrainte physique : ~1 200 W en pointe.** Vérifier climatisation et circuit électrique du local **avant** achat — c'est le piège classique de ce scénario. Le parallélisme sur deux cartes impose vLLM.

### S3 — Mac Studio M3 Ultra 96 Go
**Nœud 6 500 – 7 500 € · avec topologie B : 9 700 – 11 800 €**

819 Go/s, 200–300 W, silencieux, déploiement le plus simple de tous les scénarios. Mais le préremplissage est son défaut, et il porte précisément sur votre usage RAG. **Écarté** compte tenu du profil de charge décrit au §06 : l'enrichissement par lots y serait particulièrement lent, et c'est une charge que vous allez avoir en volume.

### S4 — RTX PRO 6000 Blackwell 96 Go
**~11 500 € la carte seule.** Techniquement le meilleur choix — 96 Go, ~1,8 To/s, une seule carte. Mais +55 % en seize mois. Avec le nœud complet, on dépasse 15 000 € pour une machine mono-carte. À réexaminer si le marché se détend.

### Synthèse

| | S1 | **S1+** | S2 | S3 Mac |
|---|---|---|---|---|
| Budget topologie B | 10 300 – 13 200 € | **11 300 – 14 500 €** | 15 000 – 18 500 € | 9 700 – 11 800 € |
| VRAM | 32 Go | 32 Go | 64 Go | 96 Go unifiés |
| 1er mot · RAG 15K | 1 – 3 s | 1 – 3 s | 1 – 2 s | 10 – 30 s |
| Charge par lots | correcte | correcte | **bonne** | lente |
| Modèles résidents | justes | justes | confortables | confortables |
| Consommation pointe | ~700 W | ~700 W | ~1 200 W | ~300 W |
| Évolutivité | carte à changer | **+1 carte** | saturée | aucune |

---

## 9. Recommandation

### Matériel : S1+ — le châssis de S2, la facture de S1

Achetez la plateforme dimensionnée pour deux cartes, n'en installez qu'une. **11 300 – 14 500 € en topologie B**, dans votre enveloppe réaliste.

Le raisonnement tient en trois points. Le marché est à un pic historique et une normalisation est attendue en 2027-2028 : **acheter la seconde carte plus tard, c'est probablement l'acheter moins cher.** Vous ne savez pas encore si votre charge par lots la justifie — le §06 dit qu'elle pourrait, votre corpus dira si elle le fait. Et le surcoût de l'option est de 800 à 1 300 €, contre plusieurs milliers si vous devez remplacer la machine entière.

C'est le seul poste où je vous conseille de payer pour de l'optionnalité plutôt que pour de la capacité.

### Séquence

**Phase 1 — 0 à 4 mois**

1. Monter le nœud, vLLM, Open WebUI, Qdrant. *(quelques jours)*
2. **Figer l'enveloppe de métadonnées** (§04) — avant toute ingestion de masse.
3. **Amorcer le référentiel d'entités** sur les 100 à 200 entités qui comptent vraiment dans votre domaine.
4. Ingérer d'abord les strates **froides** : principes, documentation technique. Elles sont stables, propres, et constituent le référentiel de confrontation dont les usages *debunk* auront besoin.
5. **Écrire un jeu d'évaluation de 30 à 50 questions** avec réponses attendues, en couvrant explicitement les cinq usages — dont au moins cinq questions à sauts multiples, qui serviront de déclencheur pour la décision sur le graphe.
6. Comparer Qwen 3.8-27B et Mistral Small 4 sur ce jeu.
7. Faire tester par un traitant volontaire.

**Phase 2 — 4 à 12 mois**

Ingestion des strates tièdes puis chaudes, une fois la séparation par strate validée. Décision sur le graphe, sur observation du jeu d'évaluation. Seconde carte si la charge par lots la justifie. Passage en topologie B si vous avez démarré en A.

### Sur l'accompagnement bénévole

Le fait qu'il soit gratuit ne le rend pas illimité, et c'est justement pourquoi il faut choisir où le dépenser. **Un appui bénévole est irrégulier et peut s'interrompre sans préavis.** Dépensez-le en priorité sur ce que vous ne pourrez pas refaire : l'enveloppe, le référentiel, la discipline d'ingestion. L'installation du socle technique est documentée, reproductible, et rattrapable seul ; la structuration du corpus ne l'est pas.

Formulé autrement : **priorisez par irréversibilité, pas par difficulté.**

Identifiez par ailleurs **une personne en interne** — pas un expert, quelqu'un de méthodique — comme référent : ingestion des nouveaux documents, tenue du référentiel, surveillance des sauvegardes, évolution du jeu d'évaluation. Un demi-jour par semaine en régime de croisière. Sans ce rôle, le corpus vieillit en silence et l'outil perd sa pertinence sans qu'on sache pourquoi.

---

## 10. Pile logicielle — et pourquoi ces choix

Vos deux précisions — markdown natif et ingestion agentique — modifient deux choix de la révision 1.

| Couche | Choix | Justification |
|---|---|---|
| Serveur d'inférence | **vLLM** *(révisé)* | La révision 1 recommandait Ollama, calibré sur « pas d'expert disponible ». Avec un appui bénévole et surtout une **charge par lots**, vLLM devient le bon choix : le traitement continu par lots lui donne un débit très supérieur sur l'enrichissement, il sert plusieurs modèles simultanément, et son API compatible OpenAI est consommée directement par Open WebUI et par vos agents. |
| Interface | **Open WebUI** | Multi-utilisateur, comptes et droits, historique, **citations des sources** — indispensable pour afficher la cotation à côté de chaque extrait. C'est la brique qui rend l'outil adoptable et vérifiable. |
| Base vectorielle | **Qdrant** | Choisi pour le **filtrage sur payload**, qui est exactement ce qu'exige votre corpus à quatre strates : filtrer par température, fiabilité, entité et fenêtre de dates *avant* le classement. Chroma est plus simple mais nettement plus faible sur ce point précis. |
| Embeddings | **BGE-M3** | Multilingue — vous aurez de l'anglais, du russe et d'autres langues en source ouverte. Il produit dense, lexical et multi-vecteurs **dans un seul modèle**, ce qui donne la recherche hybride sans faire tourner deux systèmes. |
| Réordonnancement | **bge-reranker-v2-m3** | Sur un corpus de qualité hétérogène, les cinquante premiers résultats vectoriels contiennent du bruit. Le réordonnanceur est ce qui empêche un post de réseau social de passer devant un manuel technique. Gain de pertinence très supérieur à son coût de calcul. |
| Ingestion | **markdown natif** *(révisé)* | La révision 1 proposait Docling, calibré sur un corpus majoritairement PDF. Vous passant en markdown en amont, Docling devient marginal : gardez-le en **voie de secours PDF** uniquement. L'effort se déplace vers la segmentation par titres et l'étiquetage d'entités. |
| Segmentation | **par `heading_path`** | Bénéfice direct du markdown : segmenter par titre plutôt que par nombre de caractères, et restituer le chemin complet du titre au modèle. Meilleure pertinence, coût nul. |
| Orchestration | RAG intégré, puis **LlamaIndex** | Le filtrage par strate demandera assez vite du code propre. Ne sur-ingéniérez pas au départ. |
| Conteneurisation | **Docker Compose** | Un fichier décrit toute la pile ; sauvegarde et restauration triviales. |

### Contraintes d'environnement isolé

Téléchargement hors ligne des poids et des images ; miroir de paquets local ; **chiffrement intégral des disques** du nœud et du NAS ; procédure de restauration documentée **et testée**.

**Journalisation des requêtes** : nécessaire à la sécurité, mais prévenez les utilisateurs. Un outil dont on découvre après coup qu'il est journalisé perd la confiance de ses utilisateurs — et un traitant qui se méfie de l'outil ne lui pose plus les vraies questions.

---

## 11. Risques

| Risque | Probabilité | Effet | Parade |
|---|---|---|---|
| **Injection indirecte par la strate chaude** | **Élevée** | Manipulation des réponses, exfiltration de la logique d'analyse | Voir ci-dessous |
| Strates mélangées dans un index unique | Élevée sans enveloppe | Rumeur restituée avec l'autorité d'un principe | Enveloppe §04, filtrage avant classement |
| Ingestion sans métadonnées | Élevée | **Dette irréversible** | Figer l'enveloppe avant l'ingestion de masse |
| Absence de jeu d'évaluation | Très élevée | Qualité impilotable | 30–50 questions dès la phase 1 |
| Confiance excessive dans les réponses | Élevée | Erreur d'analyse propagée | Citations et cotation obligatoires ; le modèle est un contradicteur, pas une autorité |
| VRAM saturée par les modèles résidents | Moyenne (S1) | Éviction, ralentissements | Châssis S1+, seconde carte si nécessaire |
| Appui bénévole interrompu | Moyenne | Chantier à l'arrêt | Dépenser l'appui sur l'irréversible d'abord |
| Contrainte thermique et électrique | Moyenne (S2) | Instabilité, bruit | Vérifier climatisation et circuit avant achat |
| Prix matériel volatils | Certaine | Devis périmé en 4 semaines | Validité courte ; différer la seconde carte |

### Sur l'injection indirecte

Le sas relève d'un autre chantier, mais un point le traverse et doit être posé ici, parce qu'il contraint la conception du corpus autant que celle du sas.

Vous prévoyez des agents qui collectent depuis les réseaux sociaux et écrivent dans le corpus, lequel est ensuite lu par un modèle qui répond à vos traitants. C'est un chemin complet entre un attaquant et votre système. Du texte publié publiquement, rédigé pour être ingéré, peut contenir des instructions à destination du modèle — et dans votre domaine, l'adversaire sait que vous observez.

Trois règles suffisent à couvrir l'essentiel :

1. **Le contenu ingéré est une donnée, jamais une instruction.** Les extraits doivent arriver au modèle dans une enveloppe explicitement marquée comme citation non fiable, jamais concaténés au message système.
2. **L'agent de collecte écrit en quarantaine, pas dans le corpus.** Promotion vers l'index de production après contrôle — automatique pour les strates froides, avec revue humaine pour la strate chaude, au moins au démarrage.
3. **L'agent de collecte n'a aucun droit de lecture sur le corpus de production.** Un agent compromis par le contenu qu'il lit ne doit pas pouvoir en extraire autre chose.

C'est peu coûteux si c'est prévu dès la conception, et très difficile à rattraper ensuite.

---

## 12. Points restant à clarifier

1. **Volumétrie et débit par strate** : combien de documents à l'amorçage, et surtout combien par jour sur la strate chaude ? C'est ce qui dimensionne la charge par lots et donc la décision sur la seconde carte.
2. **Politique de rétention de la strate chaude** : la conservez-vous indéfiniment, ou purgez-vous au-delà d'une fenêtre ? Cela change la taille de l'index et la conception du référentiel.
3. **Grille de cotation** : utilisez-vous une cotation source/information existante, ou faut-il en définir une ? Elle doit être figée avant l'ingestion de masse.
4. **Cadre d'homologation** : le système doit-il être homologué, et à quel niveau ? Cela peut contraindre la provenance des modèles (Qwen chinois contre Mistral français, §0.1) et imposer des délais à anticiper.
5. **Réseau isolé ou seulement sans Wi-Fi ?** La réponse change les procédures d'installation et de mise à jour.
6. **Local d'accueil** : circuit électrique et climatisation disponibles ? Contraignant seulement si vous visez S2 à terme, mais autant le vérifier avant d'acheter le châssis.
