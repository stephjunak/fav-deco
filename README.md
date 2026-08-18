# fav-deco — les meubles repérés, rangés par catégorie

> Une liste partagée entre Stéphanie et Frédéric, pour remplacer la note Apple où les
> liens de meubles s'empilaient sans classement.

**Statut :** v1 fonctionnelle, **en ligne et partagée** entre Stéphanie et Frédéric
(Firestore branché, signet Mac et raccourci iPhone installés des deux côtés). Créé le
2026-08-18.
**Type :** web app perso, partagée à deux.
**Notion :** aucun, projet volontairement hors Notion.
**En ligne :** https://stephjunak.github.io/fav-deco/
**Dépôt :** https://github.com/stephjunak/fav-deco

---

## Objectif

Repérer des meubles sur Kave Home, La Redoute, Sklum, IKEA et autres, et les retrouver
dans une grille de photos filtrable par catégorie, sur Mac comme sur iPhone, à deux.

Volontairement **hors périmètre** : votes de chacun, budget et total, statuts
d'avancement, commentaires, multi-pièces. Écartés pour aller vite. Le schéma de données
les accepte sans migration, ce sont des colonnes en plus.

## La contrainte qui a déterminé toute l'architecture

Les boutiques bloquent la lecture par un serveur. Mesuré le 18/08/2026 :

| Boutique | Fetch serveur | microlink |
|---|---|---|
| Sklum | 200 | ✅ |
| IKEA | 200 | ❌ `EPROXYNEEDED` |
| La Redoute | 403 DataDome | ❌ `EPROXYNEEDED` |
| Kave Home | 429 | ❌ `EPROXYNEEDED` |

Aucun serveur ne peut donc récupérer la photo sur 3 des 4 boutiques visées. Le seul
client qui voit ces pages est le navigateur déjà dessus, d'où la capture par **signet**
(Mac) et **raccourci Partager** (iPhone), qui lisent le DOM sur place.

Extraction, dans l'ordre : JSON-LD `Product` (le plus fiable en e-commerce), puis
OpenGraph, puis la plus grande image de la page (signet uniquement, un serveur ne peut
pas faire cette étape). Vérifié en conditions réelles sur les 4 boutiques visées.

Le signet et le raccourci **ne contiennent pas la clé de la base**. Ils extraient les
données et ouvrent l'app avec un `#add=`. L'app est le seul endroit qui parle à la base.
Les instructions d'installation (Mac et iPhone) sont dans `signet.md`, et aussi
directement dans le panneau **Commencer à utiliser l'app** (bouton « Copier le code »),
pour ne pas avoir à rouvrir le dépôt.

## Les photos : jamais de hotlinking

Règle non négociable posée par Stéphanie en cours de route : l'app ne doit **jamais**
charger une photo directement depuis le serveur du vendeur (hotlinking). L'affichage
passe par **wsrv.nl**, un proxy d'images gratuit adossé au CDN Cloudflare : le vendeur
n'est visité qu'une fois, le proxy garde sa propre copie en cache et la ressert ensuite à
tous les viewers, y compris après un vidage de cache local (vérifié par des appels
successifs : `cf-cache-status` passe à `HIT` dès le 2e appel).

La donnée stockée garde toujours l'**adresse réelle** de la photo, jamais celle du
proxy : c'est ce qu'on voit et corrige dans « Modifier la fiche ». Fonction
`urlAffichage()` dans `index.html`, appliquée aux deux endroits qui affichent une photo
(vignette, aperçu du panneau d'ajout).

**Un aller-retour à ne pas refaire :** quand le proxy échoue sur une image bloquée
(cas Sklum ci-dessous), un repli en hotlink direct a été ajouté puis retiré la même
session, sur demande explicite. Si une photo ne passe pas par le proxy, elle reste
manquante (« Photo indisponible ») plutôt que d'être chargée directement. Le seul vrai
correctif est de corriger l'URL stockée sur la fiche.

### Repli automatique : i0.wp.com quand wsrv.nl bloque

Confirmé le 18/08/2026 : **IKEA bloque wsrv.nl de façon systématique** (403 sur toutes
les images testées, pas une variante isolée). `repliImage()` retente alors une fois via
**i0.wp.com** (Jetpack Photon, WordPress/Automattic) avant d'afficher « Photo
indisponible ». Vérifié sans régression sur Sklum.

wsrv.nl reste le proxy **principal** partout : c'est un service pensé pour ce rôle
(proxy public), alors qu'i0.wp.com est un service interne à l'écosystème WordPress
détourné de son usage, moins sûr dans la durée si Automattic change sa politique. D'où
le choix d'un repli ciblé plutôt qu'un remplacement complet.

### Sites qui ont posé problème

- **Sklum bloquait le proxy sur sa variante `-large_default`** (403), la même photo
  passant très bien via une autre URL du JSON-LD. Le signet choisit déjà la bonne URL en
  priorité. **Plus de souci signalé depuis (18/08/2026)**, à garder en tête si ça
  revient.
- **decoclico a des erreurs dans son propre balisage produit** : le JSON-LD peut
  annoncer une adresse d'image qui n'existe pas sur leur serveur (ex. `140-x-200` au lieu
  de `140-x-20` dans le nom de fichier). **Plus de souci signalé depuis (18/08/2026)**.

Décision actuelle : corriger au cas par cas si ça revient (Stéphanie signale l'article,
correctif collé dans « Modifier la fiche »), pas de systématisation. **À reconsidérer si
les photos manquantes redeviennent fréquentes** — la solution complète serait de
télécharger et stocker une copie permanente de chaque photo au moment de l'ajout
(Storage + une petite fonction serveur), au lieu de dépendre de proxys tiers et de la
politique de chaque vendeur.

## Architecture

Un seul `index.html`, CSS et JS inlinés, zéro build, zéro npm. Stockage en REST direct
via `fetch`, pas de SDK. Déploiement GitHub Pages depuis `main`.

```
Page produit (Mac)     ──[signet]───────────┐
Page produit (iPhone)  ──[raccourci]────────┤
                                            ├─> app#add=<json> ─> catégorie ─> base
Coller une URL dans l'app ──────────────────┘   (repli microlink, jamais de hotlink)
```

| Fichier | Rôle |
|---|---|
| `index.html` | toute l'app |
| `signet.md` | installation du signet et du raccourci iOS (recette complète) |
| `manifest.json` + `icon.svg` | ajout à l'écran d'accueil |

### Backend : Firestore, pas Supabase — et c'est un choix contraint

Le plan d'origine était Supabase (cohérent avec les autres apps du workspace). **Bloqué
par la limite de 2 projets gratuits** sur le compte Supabase de Stéphanie, déjà atteinte
par ses deux apps Vinted (vérifié qu'une nouvelle organisation ne contourne pas la
limite : elle est globale au compte). Bascule vers Firebase/Firestore, qui n'a pas cette
limite sur son plan gratuit.

Trois différences absorbées par la couche de stockage (`charger`/`ajouter`/`modifier`/
`majCategorie`), invisibles pour le reste de l'app :
- Firestore type chaque valeur (`{stringValue}`, `{integerValue}`...) :
  `versChampsFirestore()` / `depuisChampsFirestore()` font l'aller-retour avec de
  simples objets JS.
- Pas de `WHERE` ni de `PATCH` en masse côté serveur : chargement de toute la
  collection puis filtre/tri en JS ; changement de catégorie en masse fait d'un appel
  par article concerné (`Promise.all`).
- **Une `PATCH` sans `updateMask.fieldPaths` remplace tout le document** au lieu de ne
  toucher que les champs donnés (piège vérifié par test unitaire avant mise en prod,
  jamais parti en production) : chaque modification liste explicitement ses champs via
  `urlDocFirestore()`.

## Fonctionnalités

- **Vue Catégories** (sections empilées, une par catégorie) ou **vue Tout** (grille
  unique), mémorisée d'une visite à l'autre.
- **1, 2 ou 3 colonnes de catégories**, avec des vignettes qui gardent la même taille
  partout dans un même mode (le nombre de colonnes de *produits* par catégorie est fixé
  explicitement pour ça, jamais déduit d'une largeur).
- **Gérer les catégories** (bouton dans l'en-tête) : fenêtre dédiée listant les
  catégories, une par ligne. Glisser-déposer par la poignée pour changer l'ordre
  (Pointer Events, même code souris et tactile), nom éditable directement dans la ligne,
  suppression avec confirmation. Supprimer une catégorie ne supprime aucun meuble, ils
  repassent dans « Sans catégorie ». Cet ordre est **local à chaque appareil**, pas
  partagé. Remplace l'ancien réglage inline sur les titres de section (pas assez
  intuitif : le retour de Stéphanie est que ce n'était pas clair que le nom cliqué se
  renommait).
- **Réorganiser les articles** (bouton dans l'en-tête, ex-« Gérer ») : ordre des articles
  à l'intérieur d'une catégorie, via des flèches ↑↓ et un champ `rang` par article. Celui-
  ci est **partagé** (Firestore), contrairement à l'ordre des catégories. Hors périmètre
  du passage au glisser-déposer (demande explicite : seules les catégories devaient
  changer).
- **Croix de suppression** sur chaque vignette, avec 5 secondes pour Annuler (toast avec
  bouton d'action).
- **Menu « ··· »** réduit à « Modifier la fiche » (nom, photo, prix, lien, catégorie
  tous éditables après coup).
- Prénom demandé une fois au premier lancement, associé automatiquement à chaque ajout.

## Installation

> Déjà fait pour ce projet (`FB_PROJECT`/`FB_API_KEY` renseignés dans `index.html`,
> Firestore en ligne). Section gardée comme documentation, utile en cas de redéploiement
> ou de nouveau projet Firebase.

### 1. Créer le projet Firebase

1. [console.firebase.google.com](https://console.firebase.google.com) → Ajouter un
   projet → nom `fav-deco`.
2. Menu de gauche (peut être rangé sous « Bases de données et stockage » selon la
   version de la console) → **Firestore** → Créer une base de données → **mode
   production** → région proche.
3. Onglet **Règles**, remplacer par :
   ```
   rules_version = '2';
   service cloud.firestore {
     match /databases/{database}/documents {
       match /favoris/{docId} {
         allow read: if true;
         allow create: if true;
         allow update: if true;
         allow delete: if false;
       }
     }
   }
   ```
   Publier. Pas de policy `delete` : personne ne peut vider la collection depuis
   l'extérieur, la suppression passe par le booléen `supprime`.
4. Paramètres du projet (roue crantée) → noter l'**ID du projet**. Section « Vos
   applications » → ajouter une app Web (`</>`) si aucune n'existe → copier la valeur
   `apiKey`.

### 2. Brancher l'app

Renseigner en haut du `<script>` de `index.html` :

```js
var FB_PROJECT = "fav-deco-xxxxx";   // l'ID du projet, pas forcément juste "fav-deco"
var FB_API_KEY = "la_cle_api_web";
```

Tant que ces deux valeurs sont vides, l'app tourne en **mode local** : elle fonctionne,
mais les données restent sur l'appareil et ne sont pas partagées.

### 3. Capture

Voir `signet.md`, ou le panneau **Commencer à utiliser l'app**.

## Sécurité, choix assumés

- La clé API est en clair dans un dépôt public. Sans risque particulier ici : côté
  Firestore, c'est **la règle de sécurité qui protège**, pas le secret de la clé (la clé
  API seule ne donne aucun accès si les règles ne l'autorisent pas).
- **Pas de règle `delete`** : aucune suppression définitive possible depuis l'extérieur.
- Garde-fou de chargement repris de `app Vinted recherche` : une réponse vide ne
  remplace jamais un affichage rempli.

## État actuel

**En production, partagé entre Stéphanie et Frédéric.** Firestore branché, les 36 fiches
initiales (créées en mode local avant le branchement) migrées avec succès, signet Mac et
raccourci iPhone installés et testés des deux côtés (raccourci transmis à Frédéric par
AirDrop). Testé de bout en bout : ajout, modification, suppression, annulation,
catégories (ordre, renommage, suppression), réorganisation des articles. Plus de souci
de photo manquante signalé (Sklum, decoclico, IKEA via le repli i0.wp.com).

Rien de bloquant à ce stade. Si besoin de continuer :
- Corriger au cas par cas si une photo redevient manquante (voir « Sites qui ont posé
  problème »).
- Reconsidérer la solution de stockage permanent des photos (voir « Repli automatique »)
  si les proxys tiers deviennent un point de friction récurrent.

## Bugs connus et pièges rencontrés

- **`columns` CSS + `loading="lazy"` ne vont pas ensemble.** La première version
  utilisait une grille en colonnes façon Pinterest : les images se chargeaient après le
  calcul des colonnes et n'étaient jamais peintes, alors que le DOM les disait chargées.
  Remplacé par une grille à tuiles de ratio fixe.
- **Centrage des photos très verticales (proches de 9:16).** `width/height:100% +
  object-fit` pouvait mal se comporter selon le navigateur. Remplacé par
  `max-width/max-height`, plus robuste, jamais de rognage.
- **Une fonction appelée depuis un attribut HTML inline (`onerror="...")` est invisible
  si elle est définie dans l'IIFE qui enferme le script.** Vécu deux fois (`repliImage`,
  puis vigilance appliquée pour `codeRaccourciIOS`) : la fonction existe, un appel direct
  en JS marche, mais l'attribut inline échoue en silence (`ReferenceError` jamais vue).
  Corrigé par export explicite sur `window`, uniquement pour les fonctions référencées
  depuis du HTML.
- **Apostrophes mal échappées dans une chaîne JS** (`\\'` au lieu de `\'`) cassent la
  syntaxe de tout le script, pas juste la ligne concernée. Attrapé par validation Node
  (`node --check`) avant mise en ligne, pas par simple relecture.
- **Le mode local ne filtrait pas les articles supprimés au rechargement** : `charger()`
  filtrait côté serveur pour Supabase (`supprime=eq.false`) mais pas en local, où
  `lireLocal()` renvoyait tout, y compris les articles supprimés. Un article supprimé
  réapparaissait donc après un rafraîchissement de page.
- **Firestore : une `PATCH` sans `updateMask.fieldPaths` remplace tout le document.**
  Voir section Architecture. Vérifié par test unitaire avant mise en prod.
- **Safari abîme un signet déposé par glisser-déposer** (encode le code, le rend
  syntaxiquement invalide). Le panneau Réglages propose donc un copier-coller du code
  plutôt qu'un glisser-déposer pour Safari.
- Le serveur de preview lancé par Claude Code n'a pas accès à `~/Documents` et renvoie
  404. Contournement : servir depuis Bash, puis y pointer le navigateur.
- **Basculer `FB_PROJECT`/`FB_API_KEY` de vide à renseigné change instantanément la
  source de données lue par l'app**, de `localStorage` vers Firestore (`partage` dans
  `index.html`) : plus jamais de lecture du stockage local une fois ces valeurs
  présentes. Les données ajoutées en mode local restent intactes sur l'appareil mais
  deviennent invisibles d'un coup, ce qui a fait croire à une base vidée. Vérifier
  `localStorage.getItem('favdeco:items')` sur l'appareil concerné avant de s'inquiéter ;
  la migration se fait ensuite par script (API REST Firestore, un `POST` par fiche).
- **Partage d'un raccourci iOS à un tiers : trois pièges différents.** L'erreur « iPhone
  non connecté à iCloud » lors d'un partage par lien iCloud vient en réalité du réglage
  iCloud propre à l'app **Raccourcis** (Réglages → iCloud → Raccourcis) ou d'un espace
  iCloud plein, pas d'une vraie déconnexion du compte. Un fichier `.shortcut` envoyé par
  **SMS** (pas iMessage) arrive souvent corrompu ou illisible : préférer l'**AirDrop**
  direct ou **Mail**. Et même reçu intact, le destinataire ne peut pas l'ajouter tant que
  **« Autoriser les raccourcis non fiables »** (Réglages → Raccourcis → Avancé) n'est pas
  activé : ce réglage n'apparaît lui-même qu'après avoir déjà utilisé au moins un
  raccourci de la galerie Apple, piège pour qui n'a jamais ouvert l'app.

## Prochaine étape

Aucune identifiée. Le projet est en usage normal ; revenir ici seulement si un nouveau
besoin ou un souci récurrent apparaît.
