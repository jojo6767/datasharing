# Panoptes et sa couche générative — note d'architecture

*Révision 4 — 30 août 2026 — enveloppe 20 000 € TTC, périmètre complet*

> **Ce qui change en révision 4.** Le corpus froid et tiède tient en ~400 documents et la vidéo devient marginale : le dimensionnement du stockage s'effondre et libère de la marge (§01). La grille de cotation est arrêtée, avec son problème central — le faux recoupement — qui devient le point technique le plus intéressant du dossier (§04). Le référentiel d'entités est explicité (§03). Comparaison RTX PRO 6000 contre deux RTX 5090 (§07). Cadre pour démarrer minimal sans créer de dette (§08). L'homologation sort du périmètre : bac à sable sous responsabilité du chef de corps.

---

## 0. Invariants

**Qwen 3.8-Max reste hors d'atteinte** — 2 400 milliards de paramètres, ~1,2 To en 4 bits. La cible est **Qwen 3.8-27B**, et pour ce dispositif c'est le bon choix : encodeur visuel de 27 couches, compréhension native image et vidéo, Apache 2.0, ~18 Go en 4 bits. Un seul poids couvre le texte et l'image du corpus.

**Le marché matériel est à un pic historique.** DRAM +90 % au T1 2026 puis +58 % au T2. Normalisation attendue en 2027-2028. Toute cotation a une validité de quatre semaines.

**Un seul nœud de calcul, plusieurs clients légers.** Et pour un usage documentaire, la latence de préremplissage prime sur la taille du modèle : 1 à 3 secondes avant le premier mot sur NVIDIA, contre 40 à 70 sur Apple Silicon.

---

## 1. Le corpus est petit — recadrage du dimensionnement

400 documents PDF pour les strates froide et tiède, et une vidéo marginale. C'est un ordre de grandeur en dessous de ce sur quoi la révision 3 dimensionnait, et cela change trois choses.

| | Estimation |
|---|---|
| 400 PDF × ~50 pages × ~500 mots | ~10 M mots, soit ~13 M tokens |
| Fragments à 500 tokens | ~26 000 |
| Index vectoriel correspondant | **~100 Mo** |
| Sources markdown + images | ~5 à 20 Go |

**Le corpus froid et tiède tient sur une clé USB.** L'index tient en mémoire vive. Ce n'est pas un problème de volume, c'est un problème de qualité.

Trois conséquences.

**Le stockage n'est plus un poste de coût.** On passe de ~30 To à 2 × 8 To, ce qui libère environ 2 500 € — voir §09.

**Le référentiel d'entités devient faisable à la main.** Sur 400 documents, deux à trois jours de travail humain suffisent à le constituer proprement. C'est ce qui rend la recommandation du §03 réaliste plutôt que théorique.

**Le vrai volume viendra du flux chaud**, qui est le seul poste non borné. C'est lui qu'il faut instrumenter dès le départ, et c'est pour lui que la détection de faux recoupement compte (§04).

---

## 2. Le contrat de franchissement

Inchangé sur le fond : la TF EYLAU collecte, Panoptes reçoit, et **tout ce qui n'est pas écrit dans le lot au moment du franchissement est définitivement perdu**. Le livrable n'est pas « des fichiers » mais un lot normalisé.

```
LOT-2026-08-30-001/
├── bordereau.txt          émetteur, date, volume, périmètre, visa
├── manifeste.jsonl        une ligne = un document = l'enveloppe
├── empreintes.sha256      intégrité de chaque fichier
└── contenu/
```

```json
{
  "id":             "doc-2026-08-30-0001",
  "fichier":        "contenu/doc-2026-08-30-0001.md",
  "media_type":     "texte",
  "strate":         "reseau_social",
  "temperature":    "chaud",
  "source_uri":     "https://…",
  "acteur":         "compte ou organisme émetteur",
  "acteur_id":      "ACT-0231",
  "date_pub":       "2026-08-29T14:22:00Z",
  "date_collecte":  "2026-08-30T06:10:00Z",
  "fiabilite_src":  "C",
  "veracite_info":  "3",
  "langue":         "ru",
  "entites":        ["ENT-0412"],
  "origine_declaree": "doc-2026-08-28-0117",
  "relations_collecte": [
    {"type": "reprise_de", "cible": "doc-2026-08-28-0117"},
    {"type": "publie_par", "cible": "ACT-0231"}
  ],
  "hash":           "sha256:…"
}
```

Quatre champs qu'on oublie et qu'on ne récupère jamais : **`date_pub` distincte de `date_collecte`** (un post republié aujourd'hui mais écrit il y a trois ans n'a pas le même statut) ; **`acteur_id`** et pas seulement un nom d'affichage, pour suivre un compte qui change de nom ; **la cotation**, qui doit être posée à la collecte par qui connaît la source ; **`relations_collecte` et `origine_declaree`**, données de graphe natives et irrécupérables après.

**À faire cette semaine, avant tout achat.** Une demi-journée, gratuite, et la seule chose du dossier qui soit vraiment irrattrapable.

---

## 3. Le référentiel d'entités, concrètement

Vous m'avez demandé d'être précis. Voici exactement de quoi il s'agit.

### Ce que c'est

**Une liste contrôlée des choses dont votre corpus parle, où chaque chose a un identifiant stable et tous ses noms.** Rien de plus. Un fichier tabulaire, ou une collection de notes, tenu à la main puis étendu.

### Le problème qu'il résout

Dans un corpus en français, anglais, russe et ukrainien, un même système apparaît sous une demi-douzaine de formes :

```
R-330Zh        R-330Ж        Житель        Zhitel        Jitel
« le brouilleur Zhitel »     R330ZH        1L269 (index distinct, souvent confondu)
```

Une recherche sur « Zhitel » manque les occurrences en cyrillique et celles écrites « R-330Zh ». La recherche vectorielle en rattrape une partie, de façon peu fiable et sans que vous sachiez ce qu'elle a manqué. **Le référentiel est ce qui fait de ces sept chaînes une seule chose.**

### À quoi ressemble une entrée

```
id             ENT-0412
designation    R-330Zh « Zhitel »
type           brouilleur
alias          R-330Ж ; Житель ; Zhitel ; Jitel ; R330ZH
translit       zhitel
rattachement   ENT-0088
fiche          FIC-0207                    → renvoi à la base relationnelle
premiere_vue   2024-03
statut         confirmé
note           ne pas confondre avec ENT-0455
```

Types utiles dans votre domaine : matériel, plateforme, émetteur, bande ou fréquence, unité, acteur ou compte, programme, lieu, doctrine.

### Ce que ça vous donne, immédiatement

**Rappel.** Une requête sur `ENT-0412` retrouve tous les documents, quel que soit le nom employé, en cyrillique comme en latin. C'est le gain le plus direct et le plus mesurable.

**Filtrage déterministe.** « Tout sur ENT-0412, strates froide et tiède, cotation A à C » devient une requête exacte et non une recherche approximative. C'est ce qui rend l'usage *debunk* praticable : vous confrontez une assertion à un sous-ensemble précis du corpus, pas à ce que la similarité a bien voulu remonter.

**Les nœuds du graphe futur.** Quand vous déciderez de construire le graphe sémantique, les nœuds existeront déjà, étiquetés depuis des mois. Il ne manquera que les arêtes.

### Ce que ce n'est pas

Ce n'est **pas une ontologie** — aucune hiérarchie formelle de concepts. Ce n'est **pas une taxonomie exhaustive** — on ne référence que ce dont le corpus parle réellement. Ce n'est **pas un graphe** — pas de relations riches, un rattachement optionnel au plus. Et ce n'est **pas automatique** — la machine propose, l'humain arbitre.

### Comment le construire sur 400 documents

1. Passer les 400 documents au petit modèle avec la consigne : « liste les désignations de matériels, unités, acteurs et programmes ».
2. Dédupliquer, trier par fréquence d'apparition.
3. **Un humain valide le haut de la liste.** Les 150 premières entrées couvriront l'essentiel du corpus.
4. **Regrouper les alias à la main.** C'est le vrai travail, et c'est irremplaçable : aucun modèle ne sait de façon fiable que « Житель » et « R-330Zh » sont la même chose dans *votre* usage.
5. Figer les identifiants. Un `ENT-` attribué ne se réattribue jamais.

**Deux à trois jours de travail humain.** C'est la meilleure dépense du projet, et elle ne coûte pas un euro de matériel.

Ensuite, à chaque ingestion, le petit modèle propose des rattachements contre le référentiel et l'humain n'arbitre que les nouveautés — quelques minutes par lot.

---

## 4. Cotation et faux recoupement — le point technique central

### La grille

| Fiabilité de la source | | Véracité de l'information | |
|---|---|---|---|
| **A** | Complètement fiable | **1** | Confirmée par d'autres sources |
| **B** | Habituellement fiable | **2** | Probablement vraie |
| **C** | Assez fiable | **3** | Possiblement vraie |
| **D** | Pas habituellement fiable | **4** | Douteuse |
| **E** | Non fiable | **5** | Improbable |
| **F** | Ne peut être évaluée | **6** | Ne peut être évaluée |

Les deux axes sont indépendants : une source excellente peut rapporter une information invérifiable (**A6**), une source douteuse peut dire vrai (**E1**). Le système doit toujours restituer les deux caractères, jamais un seul.

### Pourquoi le faux recoupement est *le* problème

Vous l'avez identifié, et c'est le point le plus important de cette révision.

Le **1** — donc le **A1** — ne s'obtient que par recoupement : deux canaux indépendants qui disent la même chose. Le faux recoupement, c'est trois canaux qui remontent en réalité à une source unique. C'est le mode de tromperie le plus économique pour un adversaire : arroser plusieurs relais, laisser la structure de collecte faire le reste.

Et voici le point qu'il faut voir clairement :

> **Un RAG naïf est structurellement un amplificateur de faux recoupement.**

La raison est mécanique. Plus une affirmation est reprise, plus elle produit de fragments textuellement proches dans le corpus. Plus il y a de fragments proches, plus la recherche par similarité en remonte. Plus elle en remonte, plus le contexte envoyé au modèle est saturé de la même affirmation — et plus le modèle la restitue avec assurance. **La similarité vectorielle récompense la redondance**, exactement l'inverse de ce que demande le renseignement.

```
CE QUE LE SYSTÈME VOIT              CE QU'IL Y A RÉELLEMENT

doc-A ─┐                            doc-A ─┐
doc-B ─┤                            doc-B ─┼─► reprise de doc-Z
doc-C ─┼─► « 5 documents            doc-C ─┘
doc-D ─┤     concordants »          doc-D ─────► source indépendante
doc-E ─┘                            doc-E ─────► reprise de doc-Z

                                    → 2 origines, pas 5
                                    → le « 1 » n'est pas justifié
```

### La parade, en trois couches

**1 · Le déclaré.** `relations_collecte` et `origine_declaree` : quand la reprise est explicite — citation, partage, « selon X » — la TF EYLAU l'enregistre au moment de la collecte. Gratuit, mais ne couvre que ce qui s'annonce.

**2 · L'empreinte de contenu.** Deux mécanismes complémentaires, parce qu'ils attrapent des choses différentes :
- **MinHash ou SimHash** sur le texte normalisé — attrape le copier-coller et la reformulation légère, dans une même langue. Rapide, déterministe.
- **Similarité d'embedding multilingue** (BGE-M3) — attrape la traduction et la reformulation lourde. **C'est celui qui compte pour vous** : une affirmation qui part en russe, passe en anglais et revient en français ne partage aucun n-gramme avec elle-même.

Le seuil est à calibrer sur vos données : trop bas, vous fusionnez des faits distincts ; trop haut, vous ratez les reprises. C'est un réglage empirique, à faire sur le jeu d'évaluation.

**3 · La convergence de graphe.** Remonter les arêtes depuis chaque document appuyant l'affirmation, et **compter les origines distinctes, pas les documents**. C'est une requête d'ancêtres communs.

### La règle de restitution — non négociable

> **Le système ne prononce jamais « recoupé ».**
>
> Il affiche : *« 7 documents appuient cette affirmation, se ramenant à 2 origines distinctes après déduplication. Origine 1 : ACT-0231, le 12/08. Origine 2 : ACT-0417, le 14/08. »*
>
> **Le système compte. L'analyste cote.**

La raison n'est pas de la prudence de principe. Une cotation engage, et le **1** engage particulièrement : c'est lui qui fait passer une affirmation du statut d'hypothèse à celui de fait, dans un système qui sera lu comme faisant autorité. Un modèle qui l'attribue de lui-même produit exactement l'erreur que le dispositif est censé prévenir.

En pratique, deux champs : `origine_declaree` vient du manifeste, `cluster_origine` est calculé côté Panoptes après déduplication. La différence entre les deux est un signal en soi — un document dont l'origine réelle diffère de l'origine déclarée mérite un regard.

**Et c'est la justification définitive du graphe de collecte** : sans lui, la détection de faux recoupement est impossible. Ce n'est plus un confort, c'est la fonction qui empêche le système de devenir dangereux.

---

## 5. Le graphe de collecte — volumétrie et choix technique

Vous m'avez demandé ce que voulait dire ma question sur la volumétrie. La voici, avec sa réponse.

**Ce que je demandais** : combien de nœuds et combien d'arêtes le graphe contiendra. Un nœud = un document, un compte, un média. Une arête = une relation entre deux d'entre eux.

**Pourquoi ça compte** : c'est le seul critère qui décide entre deux options d'implémentation très différentes en charge d'exploitation.

| Volumétrie | Solution | Charge d'exploitation |
|---|---|---|
| Jusqu'à ~1 million d'arêtes | **Deux tables dans PostgreSQL** + requêtes récursives | nulle, la base existe déjà |
| Au-delà | Serveur graphe dédié (Neo4j, Memgraph) | un service de plus à exploiter |

**Estimation pour votre cas.** 400 documents froids, plus un flux chaud de quelques centaines de publications par semaine : sur un an, quelques dizaines de milliers de nœuds et peut-être 100 000 arêtes. Vous êtes **un ordre de grandeur en dessous** du seuil.

**Décision : deux tables dans PostgreSQL.** Une table `noeuds`, une table `aretes`, et des requêtes récursives — le mécanisme SQL standard qui remonte une chaîne de parenté. C'est exactement ce qu'il faut pour la détection de faux recoupement, et **cela supprime un serveur du dispositif**. PostgreSQL héberge déjà les fiches matérielles et le catalogue ; il hébergera le graphe.

Vous migrerez vers un serveur graphe le jour où une requête de remontée dépassera la seconde. Ce jour n'arrivera probablement pas, et s'il arrive, la migration est mécanique : les données sont déjà sous forme de nœuds et d'arêtes.

Le **graphe sémantique** — entités et relations extraites des documents — reste différé, pour la raison inchangée : figer un schéma de relations à l'aveugle est le mode d'échec dominant de ces projets. Son déclencheur reste le jeu d'évaluation.

---

## 6. Traiter chaque nature de donnée

**Texte** — markdown en entrée, segmentation **par titre** grâce au `heading_path`. Un PDF converti en texte brut perd sa hiérarchie et oblige à découper à l'aveugle. Sur 400 PDF, prévoir une passe de contrôle qualité de la conversion : c'est là que se perdent les tableaux et les figures.

**Images** — Qwen 3.8-27B les traite nativement. Description automatique par lots, indexation de la description, question directe du traitant sur une image.

**Vidéo** — devenue marginale, elle sort du chemin critique. Le traitement reste le même quand il y en a : transcription horodatée, images-clés par changement de plan décrites par Qwen, indexation des dérivés, vidéo brute en pièce jointe jamais versée au contexte. Sur vos langues — français, anglais, russe, ukrainien — **Whisper large-v3 est le choix par défaut** : il couvre les quatre sans réserve. Parakeet-TDT v3 est environ 49 fois plus rapide, mais son intérêt était le volume, et le volume n'est plus là. Gardez-le en tête si la vidéo prend de l'ampleur.

**Fiches matérielles** — l'erreur à ne pas commettre : les vectoriser. Une fiche est une table. À la question « quelle est la bande de tel matériel », le système doit **lire une ligne**, pas retrouver un fragment de texte qui parle de bandes. La recherche vectorielle sur du tabulaire ramène la fiche d'un matériel voisin, ou une valeur d'une autre colonne — des réponses plausibles et fausses. Base relationnelle, et un outil que le modèle appelle.

**Métadonnées de réseau** — §04 et §05.

---

## 7. Le matériel : RTX PRO 6000 contre deux RTX 5090

Vous demandiez le gain. Voici la comparaison honnête, et la conclusion est peut-être contre-intuitive.

| | 1 × 5090 + GPU de service | 2 × 5090 | RTX PRO 6000 Max-Q |
|---|---|---|---|
| VRAM pour le modèle | 32 Go | 64 Go **fractionnés** | 96 Go **d'un seul tenant** |
| Bande passante | 1 792 Go/s | 1 792 Go/s par carte | ~1 800 Go/s |
| Interconnexion | — | **PCIe 5.0, pas de NVLink** | sans objet |
| Mémoire ECC | non | non | **oui** |
| Consommation en pointe | ~700 W | ~1 250 W | **~450 W** |
| Encombrement | 2 + 1 emplacements | 4+ emplacements | 2 emplacements |
| Coût des cartes | 4 100 – 4 900 € | 7 000 – 8 000 € | ~11 500 € |
| **Budget total du dispositif** | **14 200 – 18 700 €** | 16 100 – 19 800 € | 20 100 – 22 600 € |
| Qwen 3.8-27B (18 Go) | confortable | confortable | confortable |
| Modèle de 40 à 60 Go | non | oui, avec pénalité | **oui, nativement** |
| Modèle de 64 à 96 Go | non | non | **oui, exclusivement** |

### Le point technique qui décide

**Les RTX 5090 n'ont pas de NVLink.** NVIDIA l'a retiré des cartes grand public. Deux 5090 communiquent donc par le bus PCIe 5.0 x16, soit environ 64 Go/s par sens — **près de trente fois moins que la bande passante interne d'une carte**.

Conséquence pour un modèle réparti sur les deux cartes : à chaque couche, les activations transitent par ce goulot. En pratique, le parallélisme tensoriel sur deux 5090 rend **1,4 à 1,7 fois** le débit d'une carte seule, pas deux fois, et il ajoute de la latence par token. Les 64 Go sont réels, mais ils ne se comportent pas comme 64 Go d'un seul tenant.

La RTX PRO 6000 n'a pas ce problème : 96 Go dans un seul espace d'adressage, aucune pénalité d'interconnexion. Elle ajoute la mémoire ECC — qui compte sur des traitements par lots de plusieurs heures — et surtout **~450 W contre ~1 250 W**, sur deux emplacements au lieu de quatre.

### Le verdict

**La RTX PRO 6000 est une meilleure carte pour un problème que vous n'avez pas.**

Votre modèle fait 18 Go. Ses 96 Go ne servent rien aujourd'hui. Vous paieriez ~7 400 € de plus qu'un 5090 pour une capacité dont vous n'aurez l'usage que le jour où vous voudrez un modèle de 60 à 90 Go — et ce jour-là, le marché aura probablement bougé, dans le bon sens.

Le seul argument qui tienne en sa faveur, indépendamment de la capacité, c'est **l'énergie et la chaleur** : 450 W contre 1 250 W est une différence d'exploitation réelle. Vous m'avez dit que le local suivrait, ce qui le rend secondaire — mais si le local s'avérait juste, il redeviendrait décisif.

**Recommandation : un seul RTX 5090, plus un petit GPU de service de 16 Go à 600–900 € qui accueille en permanence l'embedding, le réordonnanceur et l'ASR.** Cette asymétrie libère les 32 Go du 5090 pour le seul modèle de dialogue et son cache d'attention. Elle résout le problème de mémoire résidente pour un cinquième du prix d'une seconde grosse carte.

Et le châssis reste prévu pour deux cartes — voir §08.

---

## 8. Démarrer minimal sans créer de dette

Votre question est la bonne, et la réponse tient à une distinction simple.

**Ce qui est structurel se fait bien du premier coup. Ce qui est modulaire se fait au minimum et se renforce.**

| Poste | Démarrer minimal ? | Ce qu'on risque |
|---|---|---|
| Châssis, alimentation, carte mère | **Non** | **Vraie dette.** Remplacer une carte mère ou une alimentation, c'est remonter la machine. Le surcoût du châssis bi-GPU — 800 à 1 300 € — est une assurance, pas un luxe. |
| Câblage du plancher technique | **Non** | Travail de génie civil. Tirer deux fois coûte plus cher que la différence de câble. **Tirez large du premier coup**, y compris des liens que vous n'utiliserez pas tout de suite. |
| Sauvegarde | **Non — pour une autre raison** | Pas de dette technique : un **risque de perte**. Une sauvegarde absente ne se rattrape pas rétroactivement. |
| GPU | **Oui** | Aucune dette si le châssis est prévu. On ajoute une carte, on ne remonte rien. |
| Disques, NAS | **Oui** | Aucune. On ajoute des disques, on migre, les données se copient. |
| Postes traitants | **Oui** | Aucune. Du consommable, reconditionné assumé. |
| Switch, réseau actif | **Oui** | Faible. Un switch se remplace en dix minutes. |
| Onduleur | **Oui, avec prudence** | Sous-dimensionné, il ne protège pas — mais il se remplace sans rien démonter. |

### Sur la sauvegarde en particulier

C'est le seul poste où « peu vite mal mais à temps » ne s'applique pas, et la distinction mérite d'être posée nettement : **la dette se rembourse, la perte ne se rattrape pas.** Un corpus constitué à la main pendant des mois, un référentiel d'entités qui représente des jours de travail humain irremplaçable — c'est précisément ce qui n'a pas de valeur marchande et ne se rachète pas.

Bonne nouvelle : avec 400 documents, la sauvegarde correcte coûte **deux disques externes et un coffre, environ 500 €**. Ce n'était un arbitrage que tant qu'on dimensionnait pour 30 To. Ce n'en est plus un.

Faites-la simple mais faites-la complète : deux jeux, dont un hors du local, rotation hebdomadaire, et **une restauration testée une fois** — une sauvegarde jamais restaurée n'est pas une sauvegarde, c'est une hypothèse.

### Sur le bac à sable

L'homologation sort du périmètre, c'est noté et je n'y reviens pas. Deux remarques factuelles, sans rapport avec la conformité.

Le chiffrement des disques et la sauvegarde restent justifiés indépendamment de tout cadre : le risque n'est pas réglementaire, il est de perdre le travail ou de laisser partir une machine avec le corpus dessus.

Et si l'expérimentation réussit, elle sera régularisée. Rien de ce qui est proposé ici n'entrave cette régularisation — à une condition simple : **tenez un journal de ce que vous installez et de ce que vous ingérez**. Reconstituer a posteriori l'historique d'un système qu'on veut faire homologuer coûte bien plus cher que de l'avoir écrit au fil de l'eau.

---

## 9. Budget révisé — 20 000 € TTC, périmètre complet

| # | Fonction | Ce que ça paie | Montant |
|---|---|---|---|
| 1 | **Nœud de calcul** | RTX 5090, GPU de service, plateforme bi-GPU, NVMe 4 To | 8 000 – 10 000 € |
| 2 | **Stockage du corpus** | NAS 2 baies + 2 × 8 To | 700 – 1 000 € |
| 3 | **Sauvegarde** | 2 disques externes rotatifs + coffre | 400 – 600 € |
| 4 | **Postes traitants ×3** | clients légers reconditionnés, écrans, périphériques | 1 900 – 2 500 € |
| 5 | **Réseau et réception** | switch managé, câblage large, poste de réception du lot | 1 400 – 1 900 € |
| 6 | **Énergie** | onduleur 2200 VA | 600 – 900 € |
| 7 | **Divers et marge** | câbles, rails, étiquetage, aléas | 1 200 – 1 800 € |
| | **Total** | | **14 200 – 18 700 €** |
| | **Réserve sur 20 000 €** | | **1 300 – 5 800 €** |

### Détail du nœud

| Poste | Montant |
|---|---|
| 1 × RTX 5090 32 Go — modèle de dialogue | 3 500 – 4 000 € |
| 1 × GPU de service 16 Go — embedding, réordonnanceur, ASR | 600 – 900 € |
| Plateforme bi-GPU : CPU 16c, 128 Go DDR5, carte mère double PCIe espacé, alimentation 1600 W, refroidissement | 3 500 – 4 500 € |
| NVMe 4 To | 400 – 600 € |

### Ce que la réserve doit devenir

Ne la dépensez pas. Le marché matériel est à un pic et vous êtes en expérimentation : **la réserve est ce qui vous permettra d'acheter la bonne chose dans six mois**, quand vous saurez ce qui manque — une seconde carte, une carte plus grosse, du stockage si le flux chaud surprend, ou rien.

Un budget consommé intégralement à l'ouverture d'un projet exploratoire est un budget mal utilisé.

---

## 10. L'interface

### Ce qu'on voit

Open WebUI présente une interface de conversation classique : liste des échanges à gauche, conversation au centre, sélecteur de modèle en haut. Ce qui compte pour vous n'est pas là, c'est dans ce qu'on y ajoute : **le filtre par strate, la cotation affichée sur chaque citation, et le compteur d'origines distinctes**.

Une maquette de l'écran cible, avec ces éléments en place, est fournie séparément.

### Les quatre niveaux de personnalisation

| Niveau | Ce que c'est | Effort | Verdict |
|---|---|---|---|
| **1 · Thème** | CSS injecté dans l'image Docker : couleurs, typographie, logo, nom du produit | quelques heures | **Faites-le tout de suite** |
| **2 · Modifications ciblées** | Patcher l'affichage des citations pour porter strate, cotation et origines | quelques jours | **Faites-le en phase 1** |
| 3 · Fork | Le code est forkable sans restriction depuis la v0.6.5 | semaines | Seulement si le 2 ne suffit pas |
| 4 · Interface maison | L'API est compatible OpenAI : une interface propre est possible | mois | Pas avant d'avoir de l'usage |

**Sur la licence** : les contraintes de marque d'Open WebUI ne s'appliquent qu'au-delà de 50 utilisateurs. À trois postes, vous pouvez rebrander intégralement, sans restriction.

### La recommandation

**Niveaux 1 et 2 en phase 1, pas plus.**

Le niveau 1 pour une raison d'adoption, pas d'esthétique : un outil qui a l'air d'appartenir à la maison est utilisé, un outil qui a l'air d'un prototype de passage ne l'est pas. Quelques heures, et c'est rentable.

Le niveau 2 parce que c'est la **seule modification vraiment nécessaire**, et qu'elle est fonctionnelle : sans cotation visible à côté de chaque extrait, le dispositif du §04 n'existe pas pour l'utilisateur. C'est là qu'il faut mettre l'effort de développement, pas dans les couleurs.

Les niveaux 3 et 4 sont à écarter en phase 1, non pour une question de coût mais de méthode : **une interface écrite avant l'usage code des hypothèses fausses sur ce dont les traitants ont besoin.** Vous les découvrirez en six mois d'utilisation, et vous les découvrirez gratuitement.

---

## 11. Modèles et pile logicielle

| Rôle | Modèle | Empreinte | Note |
|---|---|---|---|
| Dialogue et vision | **Qwen 3.8-27B** | ~18 Go | Encodeur visuel de 27 couches, image et vidéo natives, 262K de contexte, Apache 2.0. Un seul poids pour le texte et l'image. |
| Extraction par lots | Qwen 3 à 8B | ~5 Go | Étiquetage d'entités contre le référentiel, propositions de cotation, résumés. |
| Embeddings | **BGE-M3** | ~2 Go | Multilingue, et c'est lui qui porte la **détection de reprise translinguistique** du §04. Ce n'est plus un détail de confort. |
| Réordonnancement | bge-reranker-v2-m3 | ~1 Go | Empêche un post de réseau social de passer devant un manuel technique. |
| Transcription | **Whisper large-v3** | ~3 Go | Couvre français, anglais, russe, ukrainien sans réserve. |

Plafonnez le contexte de session à **32K–64K** : les 262K sont un argument commercial, au-delà la latence de préremplissage remonte et la qualité ne suit pas.

| Couche | Choix | Pourquoi |
|---|---|---|
| Inférence | **vLLM** | Traitement continu par lots, plusieurs modèles servis, API compatible OpenAI. |
| Interface | **Open WebUI** | Multi-utilisateur, droits, citations. Personnalisable — §10. |
| Index vectoriel | **Qdrant** | Filtrage sur payload : strate, cotation, entité, dates, **avant** le classement. |
| Relationnel **et graphe** | **PostgreSQL** | Fiches matérielles, catalogue, **et le graphe de collecte en deux tables** — §05. Un serveur de moins. |
| Orchestration | **LlamaIndex** | Filtrage par strate, jeu d'outils, déduplication. |
| Conteneurisation | **Docker Compose** | Un fichier décrit toute la pile. |

Les outils que le modèle appelle : `chercher_documents`, `interroger_fiches`, `explorer_reseau`, `lire_document`, `chercher_media`. Un modèle qui cite une ligne de table est **vérifiable** ; un modèle qui paraphrase un fragment retrouvé par similarité ne l'est pas.

---

## 12. Séquencement

**Cette semaine, avant tout achat** — figer le format de lot avec la TF EYLAU, en y intégrant la grille de cotation et `origine_declaree`. Une demi-journée, gratuite, irrattrapable.

**Phase 1 — 0 à 4 mois**
1. Monter le nœud : vLLM, Open WebUI, Qdrant, PostgreSQL.
2. Convertir les 400 PDF en markdown, avec **une passe de contrôle qualité sur un échantillon de vingt** avant d'industrialiser.
3. **Constituer le référentiel d'entités** — §03, deux à trois jours.
4. Ingérer les strates froides et tièdes, étiquetées contre le référentiel.
5. Câbler `chercher_documents`, `interroger_fiches`, `lire_document`.
6. **Thème et affichage de la cotation** dans l'interface — §10, niveaux 1 et 2.
7. **Écrire le jeu d'évaluation** — 40 questions, répartition ci-dessous.
8. Recevoir un premier lot réel et vérifier que le manifeste tient à l'usage.

**Phase 2 — 4 à 12 mois** — ouverture du flux chaud, déduplication et graphe de collecte, détection de faux recoupement, chaîne médias si la vidéo prend de l'ampleur, décision sur le graphe sémantique.

### Le jeu d'évaluation

| Type | Nombre | Ce qu'il teste |
|---|---|---|
| Fait technique vérifiable | 10 | Lit-il la bonne ligne de la bonne fiche ? |
| Rappel par alias | 5 | Une question posée avec un alias retrouve-t-elle les documents écrits avec un autre ? **Teste le référentiel.** |
| Principe scientifique | 5 | Restitue-t-il un invariant sans le déformer ? |
| Confrontation chaud / froid | 10 | Le cœur du dispositif : oppose-t-il une revendication à un principe, avec les deux cotations ? |
| Faux recoupement | 5 | **Sur un cas fabriqué exprès** : compte-t-il les origines ou les documents ? |
| Question à sauts multiples | 5 | Détecteur de graphe sémantique. |
| Question piège | 5 | Réponse absente du corpus : dit-il qu'il ne sait pas, ou invente-t-il ? |

Les deux catégories nouvelles — rappel par alias, faux recoupement — testent précisément les deux mécanismes qui font la valeur du système. **Fabriquez le cas de faux recoupement à la main** : prenez une affirmation réelle, créez cinq documents qui la reprennent depuis une origine unique, et vérifiez que le système annonce une origine et non cinq sources.

---

## 13. Risques

| Risque | Probabilité | Effet | Parade |
|---|---|---|---|
| **Faux recoupement non détecté** | **Élevée sans dispositif** | **Affirmation adverse promue au rang de fait** | §04 : déclaré, empreinte, convergence — et le système ne cote jamais |
| Lots reçus sans métadonnées complètes | Élevée sans contrat | Perte définitive de traçabilité | Figer le format avant l'industrialisation de la collecte |
| Injection indirecte par la strate chaude | Élevée | Manipulation des réponses | Contenu = donnée jamais instruction ; quarantaine ; ingestion sans droit de lecture sur la production |
| Fiches matérielles vectorisées | Élevée si non traitée | Réponses plausibles et fausses | Base relationnelle + outil dédié |
| Conversion PDF dégradée | **Élevée** | Tableaux et figures perdus, silencieusement | Contrôle qualité sur 20 documents avant d'industrialiser |
| Absence de jeu d'évaluation | Très élevée | Qualité impilotable | 40 questions dès la phase 1 |
| Sauvegarde repoussée | Moyenne | **Perte irrattrapable du corpus et du référentiel** | 500 € et une restauration testée |
| Confiance excessive dans les réponses | Élevée | Erreur d'analyse propagée | Cotation restituée avec chaque citation ; questions pièges |
| Budget consommé en totalité à l'ouverture | Moyenne | Plus de marge quand on saura quoi acheter | Garder la réserve |
| Prix matériel volatils | Certaine | Devis périmé en quatre semaines | Validité courte, achats différés |

---

## 14. Points restant ouverts

1. **Cadence et volume du flux chaud** — quelques dizaines ou quelques centaines de publications par semaine ? C'est le seul poste non borné, et il décide de la trajectoire de stockage comme du dimensionnement de la déduplication.
2. **Qui cote, et quand** — la cotation doit être posée à la collecte, côté TF EYLAU. Est-ce dans leur pratique et dans leur outillage, ou faut-il l'y introduire ? À traiter en même temps que le format de lot.
3. **Existe-t-il déjà un embryon de référentiel** — une liste de désignations, un glossaire, une nomenclature en service ? Repartir de l'existant économiserait l'essentiel des deux à trois jours du §03.
