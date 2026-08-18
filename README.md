# fav-deco — les meubles repérés, rangés par catégorie

> Une liste partagée entre Stéphanie et Frédéric, pour remplacer la note Apple où les
> liens de meubles s'empilaient sans classement.

**Statut :** en développement, v1 fonctionnelle en local. Créé le 2026-08-18.
**Type :** web app perso, partagée à deux.
**Notion :** aucun, projet volontairement hors Notion.

---

## Objectif

Repérer des meubles sur Kave Home, La Redoute, Sklum, IKEA et autres, et les retrouver
dans une grille de photos filtrable par catégorie, sur Mac comme sur iPhone, à deux.

Volontairement **hors périmètre en v1** : votes de chacun, budget et total, statuts
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

Vérifié en conditions réelles, les quatre boutiques passent par cette voie : IKEA 499 €,
La Redoute 459 €, Kave Home 339 €, Sklum 116,95 €, avec titre et photo à chaque fois.

## Architecture

Un seul `index.html`, CSS et JS inlinés, zéro build, zéro npm. Supabase en REST direct
via `fetch`, pas le SDK. Déploiement GitHub Pages depuis `main`.

```
Page produit (Mac)     ──[signet]───────────┐
Page produit (iPhone)  ──[raccourci]────────┤
                                            ├─> app#add=<json> ─> catégorie ─> Supabase
Coller une URL dans l'app ──────────────────┘   (repli microlink)
```

Le signet et le raccourci **ne contiennent pas la clé Supabase**. Ils extraient les
données et ouvrent l'app avec un `#add=`. L'app est le seul endroit qui parle à Supabase.

Extraction, dans l'ordre : JSON-LD `Product` (le plus fiable en e-commerce), puis
OpenGraph, puis la plus grande image de la page. Cette dernière étape n'existe que dans le
signet, un serveur ne peut pas la faire.

| Fichier | Rôle |
|---|---|
| `index.html` | toute l'app |
| `signet.md` | installation du signet et du raccourci iOS |
| `manifest.json` + `icon.svg` | ajout à l'écran d'accueil |

## Installation

### 1. Supabase

Créer un **projet Supabase dédié**. Ne pas réutiliser ceux de Suivi ventes Vinted ou
Vinted recherche : la clé anon vaut pour tout le projet, pas pour une table, donc donner
cette clé à Frédéric lui ouvrirait aussi les tables Vinted.

Coller ce SQL dans l'éditeur SQL de Supabase :

```sql
create table favoris (
  id         bigint generated always as identity primary key,
  url        text not null,
  titre      text,
  image_url  text,
  prix       numeric,
  boutique   text,
  categorie  text,
  ajoute_par text,
  supprime   boolean not null default false,
  cree_le    timestamptz not null default now()
);

alter table favoris enable row level security;

create policy lecture on favoris for select to anon using (true);
create policy ajout   on favoris for insert to anon with check (true);
create policy edition on favoris for update to anon using (true);
-- Volontairement AUCUNE policy delete : personne ne peut vider la table.
-- La suppression passe par le booléen "supprime".
```

### 2. Brancher l'app

Renseigner en haut du `<script>` de `index.html` :

```js
var SB_URL = "https://xxxxxxxx.supabase.co/rest/v1";
var SB_KEY = "la_cle_anon";
```

Tant que ces deux valeurs sont vides, l'app tourne en **mode local** : elle fonctionne,
mais les données restent sur l'appareil et ne sont pas partagées. Utile pour essayer.

### 3. Capture

Voir `signet.md`.

## Sécurité, choix assumés

- La clé anon est en clair dans un dépôt public. C'est le même compromis que
  `Suivi ventes Vinted`, mais ici les données sont des liens de meubles, l'enjeu est
  faible.
- **Amélioration par rapport aux apps précédentes :** pas de policy DELETE, donc aucune
  suppression définitive possible depuis l'extérieur.
- Garde-fou de chargement repris de `app Vinted recherche` : une réponse vide ne remplace
  jamais un affichage rempli.

## État actuel

Vérifié en local le 18/08/2026 : grille, filtres, menu de tuile, changement de catégorie,
suppression douce, panneau d'ajout, repli photo cassée, extraction sur les 4 boutiques,
trajet signet vers app.

Reste à faire :
1. Créer le projet Supabase et brancher les clés.
2. Installer le signet (Mac) et le raccourci (les deux iPhone).
3. `git init`, dépôt GitHub `fav-deco`, Pages depuis `main`.
4. Partager le lien à Frédéric.

## Bugs connus et pièges rencontrés

- **`columns` CSS + `loading="lazy"` ne vont pas ensemble.** La première version utilisait
  une grille en colonnes façon Pinterest : les images se chargeaient après le calcul des
  colonnes et n'étaient jamais peintes, alors que le DOM les disait chargées
  (`naturalWidth: 1400`). Remplacé par une grille à tuiles de ratio fixe, qui réserve la
  place à l'avance et rétablit au passage la lecture de gauche à droite.
- Le serveur de preview lancé par Claude Code n'a pas accès à `~/Documents` et renvoie
  404. Contournement : servir depuis Bash, puis y pointer le navigateur.

## Prochaine étape

Créer le projet Supabase et coller les clés.
