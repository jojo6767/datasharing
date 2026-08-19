# Déploiement d'un LLM local sur site — analyse et propositions

*Note d'architecture — août 2026*

---

## 0. Trois corrections préalables, avant tout chiffrage

Ces trois points changent la structure de la décision. Il faut les poser avant de parler matériel.

### 0.1 Qwen 3.8-Max n'est pas déployable chez vous

Qwen 3.8-Max est un modèle à **2 400 milliards de paramètres**. Alibaba en a effectivement publié les poids — donc, contrairement à Qwen 3.7-Max qui était strictement API, il est *techniquement* téléchargeable. Mais l'ordre de grandeur matériel est sans rapport avec votre budget :

| Quantisation | Poids seuls | Matériel minimal | Coût indicatif |
|---|---|---|---|
| 4 bits | ~1,2 To | ~10 machines à 128 Go chaînées, ou un nœud serveur HBM | 150 000 – 400 000 € |
| 8 bits | ~2,4 To | nœud datacenter multi-GPU | > 500 000 € |

À 10 000 – 20 000 €, on est deux ordres de grandeur en dessous. Ce n'est pas une question d'optimisation, c'est une question de physique : les poids doivent tenir en mémoire.

**Le bon modèle dans la même famille, c'est Qwen 3.8-27B** : dense, ~27 milliards de paramètres, fenêtre de contexte 256K, encodeur visuel intégré, licence Apache 2.0. Il tient dans **17 à 19 Go en 4 bits**, soit une seule carte grand public. Les évaluations publiées le placent au niveau de modèles 10 à 15 fois plus gros. C'est précisément le mouvement du marché depuis 18 mois : la densité de capacité par gigaoctet a progressé beaucoup plus vite que la capacité brute.

### 0.2 Le marché matériel traverse une crise mémoire historique

C'est le point le plus important pour votre calendrier d'achat.

- Les prix contrat DRAM ont progressé de **+90 à 95 % au T1 2026**, puis **+58 à 63 % au T2**.
- Un kit DDR5 32 Go est passé d'environ 100 $ (septembre 2025) à plus de 400 $.
- **Apple a purement et simplement retiré les options 512 Go (mars 2026) puis 256 Go (mai 2026)** du Mac Studio M3 Ultra. Le haut de gamme Apple plafonne aujourd'hui à **96 Go**, avec un prix d'entrée passé à ~5 299 $ (+1 300 $).
- La RTX 5090 se négocie autour de **4 200 $** contre 1 999 $ de tarif public initial.
- La RTX PRO 6000 Blackwell 96 Go est passée à **13 250 $**, soit +55 % en 16 mois.
- Le Mac Studio M5 est **repoussé à octobre 2026 au mieux**, explicitement à cause de la pénurie mémoire.
- Le retour à la normale n'est pas attendu avant **2027-2028**.

Conséquence directe : **votre budget achète aujourd'hui environ la moitié de ce qu'il achetait il y a un an**, et la stratégie « j'achète beaucoup de mémoire unifiée pour loger un très gros modèle » n'est tout simplement plus disponible à l'achat neuf.

### 0.3 « Plusieurs ordinateurs puissants branchés en série » ne fonctionne pas

L'inférence distribuée sur réseau Ethernet existe (llama.cpp RPC, exo, parallélisme de pipeline vLLM), mais elle reste un objet de laboratoire : la latence réseau entre les couches du modèle détruit le débit, et la complexité d'exploitation est incompatible avec votre contrainte « pas d'expert en interne ».

**La règle est : un seul nœud de calcul, plusieurs clients légers.** La puissance ne s'additionne pas par le réseau ; elle s'additionne dans un châssis, par le bus PCIe ou par la mémoire unifiée.

---

## 1. Ce que vous cherchez réellement à dimensionner

Vous avez formulé le besoin en « nombre de paramètres ». C'est l'intuition naturelle, mais ce n'est pas la variable qui déterminera votre satisfaction. Pour un usage RAG documentaire à trois utilisateurs, trois grandeurs comptent, dans cet ordre :

**1. La vitesse de préremplissage (*prefill*), c'est-à-dire le temps avant le premier mot de réponse.**

Une requête RAG, ce n'est pas une question de 20 mots. C'est votre question *plus* 10 000 à 30 000 tokens d'extraits documentaires injectés dans le contexte. Le modèle doit lire tout cela avant d'écrire son premier caractère. Cette phase est **limitée par la puissance de calcul brute**, pas par la bande passante mémoire.

C'est exactement la faiblesse structurelle d'Apple Silicon : les mesures publiées donnent **NVIDIA 3 à 8 fois plus rapide en préremplissage**. Concrètement, sur un contexte RAG de 15 000 tokens, un Mac Studio M3 Ultra faisant tourner un très gros modèle peut demander **40 à 70 secondes avant le premier mot**. Une station NVIDIA sur un modèle de 27B répond en **1 à 3 secondes**.

Cette différence n'est pas un détail de confort. C'est le facteur qui décide si vos traitants utilisent l'outil ou l'abandonnent au bout de trois semaines. Un outil qui répond en deux secondes devient un réflexe ; un outil qui répond en une minute devient une corvée que l'on contourne.

*À noter : la génération M5 d'Apple corrige spécifiquement ce point (accélérateurs neuronaux dans chaque cœur GPU, 3 à 4× sur le temps au premier token). Mais elle n'est pas encore disponible en Mac Studio.*

**2. La qualité de la recherche documentaire.**

Un modèle de 27B alimenté par une recherche propre (segmentation soignée, recherche hybride lexicale + vectorielle, réordonnancement) bat systématiquement un modèle de 235B alimenté par une recherche approximative. Le modèle ne peut pas raisonner sur ce qu'on ne lui a pas donné. C'est là que se joue 80 % de la valeur, et c'est là qu'il faut mettre l'effort humain.

**3. La taille du modèle.**

Elle arrive en troisième. Elle joue sur la finesse du raisonnement, la nuance rédactionnelle, la capacité à tenir un raisonnement contradictoire long — ce qui n'est pas rien pour vos usages de critique et de *debunk*. Mais son rendement est décroissant, et son coût matériel est explosif.

**Traduction pour votre décision : n'achetez pas le plus gros modèle possible. Achetez la latence la plus basse possible sur un très bon modèle de taille moyenne, et investissez la différence dans le corpus.**

---

## 2. Question de souveraineté sur les poids

Votre environnement (pas de sans-fil, planchers techniques, « traitants ») suggère un contexte où la provenance de la chaîne logistique numérique est un critère, pas une préférence.

Faire tourner un modèle en local règle la question de la **fuite de données** : rien ne sort. Cela ne règle pas la question des **poids eux-mêmes** : données d'entraînement non divulguées, aucune garantie sur l'intégrité de la chaîne d'entraînement, biais et angles morts non auditables. Le débat est actif : la NDAA FY2026 américaine interdit explicitement les modèles DeepSeek/High-Flyer dans les systèmes du DoD et chez ses sous-traitants.

Je ne me prononce pas sur votre cadre réglementaire — c'est à votre RSSI et à votre autorité d'homologation de trancher. Mais je vous signale que le choix est ouvert et qu'il existe une option française directement comparable :

| Modèle | Origine | Taille | Poids en 4 bits | Licence | Commentaire |
|---|---|---|---|---|---|
| **Qwen 3.8-27B** | Chine (Alibaba) | 27B dense, 256K ctx, vision | ~17–19 Go | Apache 2.0 | Meilleur rapport capacité/mémoire du marché |
| **Mistral Small 4** | **France** | ~24–30B | ~15–20 Go | Apache 2.0 | Option souveraine, excellent en français |
| **gpt-oss-120b** | États-Unis (OpenAI) | 120B MoE, ~5B actifs | ~63 Go (MXFP4) | Apache 2.0 | Bon raisonnement, si vous avez 96–128 Go |
| Qwen3-235B-A22B | Chine | 235B MoE | ~125–135 Go | Apache 2.0 | Hors budget en 2026 |
| Mistral Large 3 | France | 675B MoE, 41B actifs | ~340 Go | Apache 2.0 | Hors budget |
| Qwen 3.8-Max | Chine | 2 400B | ~1,2 To | — | Hors budget d'un facteur 20 |

**Recommandation pratique : montez la plateforme de façon agnostique** (le serveur d'inférence charge un fichier de poids, quel qu'il soit) et **évaluez Qwen 3.8-27B et Mistral Small 4 côte à côte sur votre propre jeu de tests**. Le coût marginal de tester les deux est nul ; la décision devient factuelle au lieu d'être idéologique. Gardez la possibilité de basculer : c'est un fichier à remplacer, pas une architecture à refaire.

---

## 3. Architecture logique (indépendante du matériel)

Quel que soit le scénario retenu, le système comporte cinq couches. C'est utile de les séparer, parce qu'elles ne se dimensionnent pas, ne se déploient pas et ne vieillissent pas au même rythme.

```
┌─────────────────────────────────────────────────────────┐
│  5. POSTES TRAITANTS  — écran + clavier + navigateur    │
│     verrouillé, aucun calcul local                      │
└───────────────────────┬─────────────────────────────────┘
                        │  Ethernet Cat6a (plancher technique)
                        │  VLAN dédié, switch managé
┌───────────────────────┴─────────────────────────────────┐
│  4. INTERFACE  — Open WebUI : comptes, droits,          │
│     historique, citations, espaces de travail           │
├─────────────────────────────────────────────────────────┤
│  3. ORCHESTRATION RAG  — recherche hybride              │
│     (lexicale + vectorielle), réordonnancement,         │
│     construction du contexte, garde-fous                │
├─────────────────────────────────────────────────────────┤
│  2. INDEX  — Qdrant (vecteurs) + index lexical          │
│     + métadonnées (source, date, classification)        │
├─────────────────────────────────────────────────────────┤
│  1. INFÉRENCE  — Ollama ou vLLM : LLM génératif         │
│     + modèle d'embedding + réordonnanceur               │
└─────────────────────────────────────────────────────────┘
         ▲
         │  chaîne d'ingestion (hors ligne, par lots)
┌────────┴────────────────────────────────────────────────┐
│  0. INGESTION  — PDF/DOCX → texte structuré (Docling),  │
│     OCR des scans, segmentation, vectorisation          │
│     + SAS D'IMPORT depuis l'extérieur                   │
└─────────────────────────────────────────────────────────┘
```

### Le point que votre cahier des charges ne couvre pas encore : le sas

Deux de vos cinq usages — **« avis technique sur une annonce publique »** et **« debunk technique »** — portent par nature sur de l'information *récente et externe*. Un système isolé ne peut pas les servir : le modèle est figé à sa date d'entraînement et votre corpus à sa dernière ingestion.

Il faut donc prévoir explicitement un **canal d'import contrôlé** : un poste connecté à Internet, physiquement séparé, qui collecte (veille, annonces, publications), passe le résultat au contrôle (antivirus, désarmement de format, revue humaine), et transfère vers le côté isolé par support amovible ou par diode. Cadence hebdomadaire ou bimensuelle selon vos usages.

C'est une brique à budgéter — en matériel c'est marginal (un poste, ~1 000 €), en procédure c'est structurant. Si elle est absente, deux de vos cinq cas d'usage ne fonctionneront pas, et vous ne vous en apercevrez qu'après la mise en service.

---

## 4. Topologie A — Socle simple, un accès

C'est la configuration pilote. Elle sert à valider les usages, monter le corpus et former l'équipe, avant d'engager le reste.

```
   ┌──────────────────────┐
   │   NŒUD DE CALCUL     │
   │   (le « serveur »)   │        ┌─────────────────┐
   │                      │───────►│  POSTE TRAITANT │
   │  Inférence + index   │  RJ45  │  écran+clavier  │
   │  + interface web     │        └─────────────────┘
   └──────────┬───────────┘
              │
   ┌──────────┴───────────┐
   │  Sauvegarde (NAS)    │
   └──────────────────────┘

   Hors zone : poste de veille Internet ──► sas d'import (support amovible)
```

Un seul câble, pas de switch, pas de VLAN. Le nœud de calcul héberge tout : serveur d'inférence, base vectorielle, interface web. Le poste traitant n'est qu'un navigateur.

**Ce que ça permet déjà :** l'intégralité de vos cinq usages, pour un utilisateur à la fois. C'est suffisant pour six mois de montée en compétence.

**Coût de la périphérie (hors nœud de calcul) : ~2 000 – 2 700 € TTC**
- Poste client (mini-PC reconditionné type OptiPlex Micro / EliteDesk Mini) : 250 – 350 €
- Écran 27" QHD + clavier/souris : 350 – 450 €
- Câblage Cat6a en plancher technique : 150 – 300 € (si réalisé en interne)
- Onduleur 1500 VA : 350 – 450 €
- NAS de sauvegarde 2 × 8 To en miroir : 700 – 900 €
- Poste de veille pour le sas : 200 – 300 € (reconditionné)

---

## 5. Topologie B — Étoile, trois accès

L'ajout est modeste : le nœud de calcul ne change pas, on ajoute un commutateur et deux postes.

```
                     ┌──────────────────────┐
                     │   NŒUD DE CALCUL     │
                     │  Inférence + index   │
                     │  + interface web     │
                     └──────────┬───────────┘
                                │ 2.5 GbE
                     ┌──────────┴───────────┐
                     │  SWITCH MANAGÉ       │
                     │  VLAN dédié, 8 ports │
                     └───┬──────┬───────┬───┘
                         │      │       │
                  ┌──────┴─┐ ┌──┴───┐ ┌─┴──────┐
                  │ POSTE 1│ │POSTE2│ │ POSTE 3│
                  └────────┘ └──────┘ └────────┘
                         │
              ┌──────────┴──────────┐
              │  Sauvegarde (NAS)   │
              └─────────────────────┘
```

**Surcoût par rapport à la topologie A : ~1 400 – 1 900 € TTC**
- 2 postes clients supplémentaires + écrans + périphériques : 1 100 – 1 500 €
- Switch managé 8 ports 2.5 GbE (VLAN, journalisation) : 200 – 400 €
- Câblage supplémentaire : compris dans le plancher technique

**Sur les trois utilisateurs simultanés.** Le plafond que vous proposez est raisonnable, et même plutôt conservateur : « trois traitants qui utilisent l'outil dans la journée » ne signifie presque jamais « trois requêtes au même instant ». En pratique, une file d'attente à 3 places sur un modèle de 27B avec une carte moderne est transparente pour l'utilisateur. Le paramètre à surveiller n'est pas le nombre d'utilisateurs mais le **cache d'attention** : trois sessions à 32 000 tokens de contexte consomment quelques gigaoctets de mémoire *en plus* des poids du modèle. Il faut le provisionner, et plafonner le contexte par session (32K à 64K est un bon compromis ; les 256K de Qwen 3.8-27B sont un argument commercial, pas un régime de croisière).

---

## 6. Scénarios matériels pour le nœud de calcul

Tous les prix sont TTC, indicatifs, **à revalider au devis** : le marché mémoire bouge de semaine en semaine en 2026. En achat public, raisonner HT et passer par un revendeur référencé.

### S0 — Mini-PC à mémoire unifiée AMD (pilote à bas coût)
**~2 000 – 2 700 €**

AMD Ryzen AI Max+ 395 « Strix Halo », 128 Go de mémoire unifiée, 2 To NVMe. Format mini-PC, silencieux, ~120 W.

- **Capacité mémoire** : 128 Go — techniquement de quoi charger gpt-oss-120b
- **Bande passante** : ~256 Go/s — c'est le point faible
- **Débit réel** : 40–80 tok/s sur les modèles < 10B, honorable sur les MoE 27–35B, faible sur les modèles denses lourds
- **Préremplissage** : médiocre, même problème structurel que le Mac
- **Déploiement** : moyen — ROCm/Vulkan sont matures mais moins qu'CUDA

**Verdict** : à considérer sérieusement **comme machine de validation**, pas comme cible. Pour 2 500 €, vous montez tout le pipeline, vous ingérez votre corpus, vous écrivez votre jeu d'évaluation, vous formez vos traitants — et vous décidez de la vraie machine dans six mois en connaissance de cause.

### S1 — Station NVIDIA mono-GPU
**~9 500 – 11 500 € — recommandé sur l'enveloppe 10 k€**

Station de travail + 1 × RTX 5090 (32 Go GDDR7, ~1 792 Go/s).

| Poste | Coût |
|---|---|
| RTX 5090 32 Go | 3 500 – 4 000 € |
| Plateforme (CPU 16c, 128 Go DDR5, carte mère, alim 1200 W, boîtier, refroidissement) | 3 000 – 4 000 € |
| Stockage 4 To NVMe + 8 To HDD | 600 – 900 € |
| **Sous-total nœud** | **7 100 – 8 900 €** |
| Périphérie topologie A | + 2 000 – 2 700 € |
| Périphérie topologie B | + 3 400 – 4 600 € |

- **Modèles** : Qwen 3.8-27B ou Mistral Small 4 en 8 bits *avec* contexte confortable ; ou en NVFP4 avec un très large cache d'attention
- **Préremplissage** : excellent — premier token en 1 à 3 s sur un contexte RAG de 15 000 tokens
- **Trois utilisateurs** : sans difficulté
- **Déploiement** : bon — CUDA est l'écosystème le mieux documenté ; Ollama fonctionne immédiatement, vLLM disponible si besoin

**Verdict** : c'est le meilleur rapport expérience-utilisateur / euro en août 2026. Le compromis assumé est le plafond de 32 Go : vous ne monterez pas au-delà de ~30B en bonne qualité. Compte tenu de ce que valent les 27B aujourd'hui, c'est un compromis très acceptable.

### S2 — Station NVIDIA bi-GPU
**~17 000 – 21 000 € — l'enveloppe 20 k€**

2 × RTX 5090 (64 Go de VRAM cumulée).

| Poste | Coût |
|---|---|
| 2 × RTX 5090 | 7 000 – 8 000 € |
| Plateforme renforcée (alim 1600 W, refroidissement, châssis) | 4 000 – 5 000 € |
| Stockage | 800 – 1 200 € |
| **Sous-total nœud** | **11 800 – 14 200 €** |
| Périphérie topologie B | + 3 400 – 4 600 € |
| Prestation d'intégration | + 3 000 – 5 000 € |

- **Modèles** : tout ce qui précède en pleine précision, plus les MoE ~70–80B en 4 bits
- **Contrainte physique** : ~1 200 W en pointe, dissipation thermique réelle, bruit. **À vérifier avant achat : le local dispose-t-il de la climatisation et du circuit électrique adéquats ?** C'est le piège classique de ce scénario.
- **Déploiement** : moyen — le parallélisme tensoriel sur deux cartes demande vLLM, donc un vrai paramétrage

**Verdict** : bon scénario si votre corpus s'avère exiger un modèle plus large. Mais **ne le prenez pas d'emblée** : vous doublez le coût pour un gain que vous ne savez pas encore mesurer.

### S3 — Mac Studio M3 Ultra 96 Go
**~9 500 – 11 000 €**

| Poste | Coût |
|---|---|
| Mac Studio M3 Ultra 28c/60c, 96 Go, 1 To | 6 500 – 7 500 € |
| Périphérie topologie B | + 3 400 – 4 600 € |

- **Bande passante** : 819 Go/s — excellent pour la *génération*
- **Consommation** : ~200–300 W, silencieux, format posé sur un bureau
- **Déploiement** : **le meilleur de tous les scénarios** — LM Studio ou Ollama fonctionnent en quelques minutes, MLX est optimisé Apple Silicon, aucun pilote à gérer
- **Préremplissage** : c'est le défaut, et il porte précisément sur votre usage RAG
- **Plafond** : 96 Go depuis la suppression des options 256/512 Go. gpt-oss-120b passe ; Qwen3-235B ne passe pas.

**Verdict** : je comprends l'attrait, et sur le critère « facilité de déploiement » il gagne nettement. Mais en août 2026 il souffre de deux problèmes conjoncturels : **l'argument de la grosse mémoire unifiée a disparu du catalogue**, et vous payez cher une bande passante qui optimise la phase (génération) qui n'est pas votre goulot d'étranglement, tandis que la phase qui l'est (préremplissage) est son point faible.

**Si la simplicité de déploiement est vraiment le critère dominant, ce scénario reste défendable** — assumez alors des temps de réponse de 10 à 30 secondes en usage RAG, et dites-le aux utilisateurs à l'avance.

### S4 — Le scénario qu'il faut écarter
**RTX PRO 6000 Blackwell 96 Go : ~11 500 € pour la seule carte** (13 250 $, +55 % en 16 mois), à quoi il faut ajouter la station. Techniquement le meilleur choix — 96 Go, ~1,8 To/s, une seule carte, donc simplicité d'exploitation. Mais l'inflation du prix la place hors de votre enveloppe, et le rapport capacité/prix s'est fortement dégradé. À réexaminer si le marché se détend.

### Synthèse comparative

| | S0 Mini-PC AMD | S1 NVIDIA 1 GPU | S2 NVIDIA 2 GPU | S3 Mac Studio |
|---|---|---|---|---|
| Budget total (3 postes) | 4 500 – 5 500 € | 10 500 – 13 500 € | 17 000 – 21 000 € | 9 900 – 12 100 € |
| Mémoire modèle | 128 Go (lente) | 32 Go (rapide) | 64 Go (rapide) | 96 Go (moyenne) |
| Temps au 1er mot (RAG 15K) | 20 – 45 s | **1 – 3 s** | **1 – 2 s** | 10 – 30 s |
| Modèle cible | Qwen 3.8-27B | Qwen 3.8-27B / Small 4 | + MoE 70–80B | + gpt-oss-120b |
| 3 utilisateurs simultanés | limite | **oui** | **oui** | acceptable |
| Facilité de déploiement | moyenne | bonne | moyenne | **excellente** |
| Consommation / bruit | **excellent** | moyen | contraignant | **excellent** |
| Évolutivité | aucune | +1 GPU possible | saturée | aucune |

---

## 7. Recommandation

### Le fond : phasez, ne dépensez pas tout maintenant

Trois raisons convergentes :

**1. Le marché est à un pic historique.** Vous achèteriez au plus mauvais moment des dix dernières années. Une normalisation est attendue en 2027-2028, et la génération Apple M5 — qui corrige précisément la faiblesse en préremplissage — arrive en fin d'année.

**2. Vos usages ne sont pas encore qualifiés.** Vous listez cinq cas d'usage assez différents. « Aide à la rédaction » et « debunk technique » n'ont ni les mêmes exigences de latence, ni les mêmes exigences de fraîcheur documentaire, ni le même profil de risque. Vous ne savez pas encore lequel portera la valeur. Dimensionner avant de savoir, c'est acheter au hasard.

**3. La valeur est dans le corpus, et le corpus se construit sans la grosse machine.** L'ingestion, la segmentation, le nettoyage, les métadonnées, le jeu d'évaluation : tout cela se fait sur une machine modeste. C'est aussi ce qui prend le plus de temps humain.

### Le plan que je propose

**Phase 1 — Valider (0 à 4 mois) — 5 500 à 7 000 €**

Scénario **S0** (mini-PC AMD 128 Go, ~2 500 €) ou **S1 dégradé** (station à une RTX 5080/5090 d'occasion), en **topologie A** : un seul poste.

Objectifs, dans l'ordre :
1. Monter la chaîne complète (ingestion → index → inférence → interface)
2. Ingérer le corpus réel — pas un échantillon
3. **Écrire un jeu d'évaluation de 30 à 50 questions** avec les réponses attendues, tirées de votre corpus. C'est l'étape que tout le monde saute et c'est celle qui décide de tout : sans elle, vous ne saurez jamais si un changement améliore ou dégrade le système.
4. Comparer Qwen 3.8-27B et Mistral Small 4 sur ce jeu
5. Faire tester par un traitant volontaire, en conditions réelles

À la fin de la phase 1, vous saurez précisément : quels usages fonctionnent, quelle latence est tolérable, quelle taille de modèle est nécessaire, et si le graphe de connaissances vaut l'effort.

**Phase 2 — Industrialiser (4 à 12 mois) — le reste de l'enveloppe**

Achat du nœud définitif, dimensionné sur des mesures et non sur des hypothèses. Passage en **topologie B**, trois postes. Mise en place du sas d'import, des sauvegardes, de la procédure d'exploitation.

Si vous devez absolument tout engager sur un seul exercice budgétaire : prenez **S1 en topologie B** (~11 000 – 13 500 €), gardez la réserve pour la prestation d'intégration, et n'achetez pas S2.

### Sur les graphes de connaissances

Vous les mentionnez à côté du RAG. Mon avis, sans ambiguïté : **ne les faites pas en phase 1.**

Construire un graphe de connaissances exploitable sur un corpus documentaire demande une extraction d'entités et de relations, un schéma, une résolution d'entités, et une maintenance continue. C'est un multiplicateur d'effort d'un ordre de grandeur, pour un gain que vous ne pouvez pas encore mesurer.

Commencez par une **recherche hybride bien réglée** : index lexical (BM25) + index vectoriel + réordonnanceur. Cela couvre l'essentiel du besoin pour une fraction du coût. Vous ajouterez le graphe en phase 2 *si* votre jeu d'évaluation montre une classe de questions que la recherche hybride échoue systématiquement à traiter — typiquement les questions à sauts multiples (« quels acteurs relient X à Y »). Vous saurez alors exactement quoi modéliser, au lieu de modéliser à l'aveugle.

---

## 8. Pile logicielle

Tout est open source, sans coût de licence, et installable hors ligne.

| Couche | Choix | Pourquoi |
|---|---|---|
| Serveur d'inférence | **Ollama** (simplicité) ou **vLLM** (débit) | À 3 utilisateurs, Ollama suffit ; vLLM n'ouvre l'écart qu'au-delà de ~8 requêtes concurrentes. Commencez par Ollama. |
| Interface | **Open WebUI** | Multi-utilisateur, comptes et droits, historique, citations des sources, espaces de travail. C'est la brique qui rend l'outil adoptable. |
| Base vectorielle | **Qdrant** | Conteneur unique, filtres riches sur métadonnées, exploitation simple |
| Embeddings | **BGE-M3** ou **Qwen3-Embedding** | Multilingues, bons en français |
| Réordonnancement | **bge-reranker-v2-m3** | Gain de pertinence très supérieur à son coût de calcul |
| Ingestion documentaire | **Docling** (+ OCR pour les scans) | PDF/DOCX → markdown structuré, conserve tableaux et titres |
| Orchestration | RAG intégré Open WebUI, puis **LlamaIndex** si besoin de contrôle | Ne sur-ingéniérez pas au départ |
| Conteneurisation | **Docker Compose** | Un seul fichier décrit toute la pile ; sauvegarde et restauration triviales |

### Contraintes propres à un environnement isolé

À traiter dès la phase 1, sous peine de blocage :
- **Téléchargement hors ligne** des poids de modèles (plusieurs dizaines de Go) et des images Docker, via le sas
- **Miroir de paquets** local ou procédure d'installation par archive
- **Chiffrement intégral des disques** sur le nœud et sur le NAS
- **Journalisation des requêtes** — nécessaire à la sécurité, mais **prévenez les utilisateurs** : un outil dont on découvre après coup qu'il est journalisé perd la confiance de ses utilisateurs, et un traitant qui se méfie de l'outil ne lui pose plus les vraies questions
- **Procédure de restauration documentée et testée** — pas seulement écrite
- **Homologation SSI** : à instruire en parallèle, pas après

---

## 9. La question de la compétence

C'est votre contrainte la plus sérieuse, et elle est sous-évaluée dans le budget.

### Estimation de la prestation externe

| Lot | Jours | Contenu |
|---|---|---|
| Socle technique | 3 j | OS, pilotes, Docker, serveur d'inférence, Open WebUI, Qdrant |
| **Chaîne d'ingestion** | **5 j** | **Le vrai travail : parsing de VOS documents, OCR, segmentation, métadonnées** |
| Réglage de la recherche | 2 j | Recherche hybride, réordonnancement, itération sur le jeu d'évaluation |
| Transfert de compétences | 2 j | Documentation d'exploitation, formation de l'exploitant interne |
| **Total** | **~12 j** | **8 000 – 11 000 € HT** à 700–900 €/jour |

**Ce chiffre est du même ordre que le matériel.** Si votre enveloppe de 10 000 € couvre matériel *et* prestation, il faut arbitrer — et dans ce cas je recommande sans hésiter de réduire le matériel (scénario S0) plutôt que la prestation. Une machine surdimensionnée mal configurée ne produit rien ; une machine modeste bien intégrée produit beaucoup.

**Point à clarifier de votre côté : les 10 000 / 20 000 € couvrent-ils la prestation, ou seulement le matériel ?** La réponse change la recommandation.

### Le rôle interne à créer

Identifiez **une personne** — pas un expert, mais quelqu'un de méthodique et curieux — qui devient le référent : elle ingère les nouveaux documents, surveille les sauvegardes, remonte les problèmes, fait évoluer le jeu d'évaluation. Comptez **un demi-jour par semaine** en régime de croisière. Sans ce rôle, le système se dégrade en silence : le corpus vieillit, personne ne s'en aperçoit, et l'outil perd sa pertinence sans que l'on sache pourquoi.

### Modèle de recours à l'expertise

Plutôt qu'une prestation en bloc, préférez **un socle de 5 jours puis des jalons d'une journée** répartis sur six mois. Vous apprenez entre deux interventions, vos questions deviennent précises, et le transfert de compétences se fait réellement au lieu d'être un chapitre de rapport.

---

## 10. Risques principaux

| Risque | Probabilité | Effet | Parade |
|---|---|---|---|
| Latence RAG jugée inacceptable → abandon | **Élevée** si scénario Mac | Projet mort | Privilégier le préremplissage rapide ; annoncer les temps de réponse à l'avance |
| Corpus mal ingéré (PDF scannés, tableaux) | **Élevée** | Réponses fausses ou vides | Auditer un échantillon de 20 documents *avant* d'industrialiser |
| Absence de jeu d'évaluation | **Très élevée** | Impossible de piloter la qualité | 30–50 questions dès la phase 1, non négociable |
| Confiance excessive dans les réponses | Élevée | Erreur d'analyse propagée | Citations obligatoires ; former les traitants à vérifier les sources ; le modèle est un contradicteur, pas une autorité |
| Usages « debunk » sans données fraîches | Certaine sans sas | 2 usages sur 5 inopérants | Sas d'import dès la phase 1 |
| Contrainte thermique/électrique (scénario S2) | Moyenne | Instabilité, bruit | Vérifier climatisation et circuit électrique avant achat |
| Prix matériel volatils | Certaine | Devis périmé en 4 semaines | Devis à validité courte ; phaser les achats |

---

## 11. Points à clarifier

1. **Votre dernier point est resté incomplet** : « Cœur de métier et contrainte supplémentaire ». C'est probablement le plus déterminant de la liste — de quoi s'agit-il ?
2. **Le budget couvre-t-il la prestation d'intégration**, ou seulement le matériel ?
3. **Volumétrie du corpus** : combien de documents, quel volume, quels formats, quelle proportion de PDF scannés ? Cela conditionne le dimensionnement du stockage et surtout la charge d'ingestion.
4. **Cadre d'homologation** : le système doit-il être homologué ? À quel niveau ? Cela peut contraindre le choix de provenance des modèles et imposer des délais qu'il vaut mieux anticiper.
5. **Le réseau est-il isolé d'Internet**, ou simplement sans Wi-Fi ? La réponse change complètement les procédures d'installation et de mise à jour.
6. **Existe-t-il déjà une base documentaire structurée** (GED, Obsidian, arborescence de fichiers) ? Un corpus déjà organisé peut diviser par deux le coût d'ingestion.
