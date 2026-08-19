# Attraper un produit en un clic

Trois façons d'ajouter un meuble. La première est la bonne dans presque tous les cas.

---

## Pourquoi un signet plutôt qu'un simple copier-coller

Les boutiques se protègent des robots. Testé le 18/08/2026, en allant chercher la page
depuis un serveur :

| Boutique | Réponse au serveur |
|---|---|
| Sklum | 200, lisible |
| IKEA | 200, lisible |
| La Redoute | **403**, page anti-bot DataDome |
| Kave Home | **429**, bloqué |

Et le service d'enrichissement automatique (microlink) échoue sur **IKEA, La Redoute et
Kave Home**, avec le code `EPROXYNEEDED` : il faudrait son option proxy payante.

Autrement dit, coller un lien ne suffira pas sur 3 de tes 4 boutiques préférées.

**Le signet, lui, lit la page depuis ton propre navigateur.** Pour la boutique, c'est toi
qui regardes la page, pas un robot. Vérifié sur les quatre :

| Boutique | Ce que le signet récupère |
|---|---|
| IKEA | EKTORP Canapé 2 places, photo, **499 €** |
| La Redoute | Canapé-lit TUSKE, photo, **459 €** |
| Kave Home | Fauteuil Asen, photo, **339 €** |
| Sklum | Chaise Jolie, photo, **116,95 €** |

---

## 1. Sur Mac : le signet

### Dans Safari : copier-coller, surtout pas glisser

**Ne fais pas glisser le bouton dans Safari.** Safari enregistre alors une version
encodée du code, où les espaces deviennent `%20` et les apostrophes `%27`. Le script
devient une erreur de syntaxe et il ne se passe strictement rien au clic, sans le moindre
message. C'est le piège qui a coûté un aller-retour le 18/08/2026.

1. Ouvre fav-deco, clique sur **Commencer à utiliser l'app**.
2. Clique sur **Copier le code du signet**.
3. Menu **Signets**, puis **Modifier les signets**.
4. Clic droit dans le dossier **Favoris**, puis **Nouveau signet**.
5. Nomme-le `fav-deco` et **colle le code dans le champ Adresse**.

### Dans Chrome

Le glisser-déposer fonctionne : fais glisser le bouton « + fav-deco » depuis
**Commencer à utiliser l'app** directement dans la barre de favoris.

### À l'usage

Sur n'importe quelle page produit, clique sur `fav-deco` dans ta barre de favoris.
fav-deco s'ouvre avec le nom, la photo et le prix déjà remplis. Tu choisis la catégorie,
tu enregistres.

> Si la barre de favoris est cachée dans Safari : menu **Présentation**, puis
> **Afficher la barre des favoris**.

### Important : l'adresse est inscrite dans le signet

Le signet contient l'adresse de l'app au moment où tu l'as copié. **Tant que l'app n'est
pas déployée en ligne, le signet pointe vers une adresse locale temporaire et ne peut pas
fonctionner depuis une page de boutique en https.** Il faut donc déployer l'app d'abord,
puis reprendre le code du signet depuis **Commencer à utiliser l'app**. Même chose si tu
redéploies ailleurs plus tard.

---

## 2. Sur iPhone : le raccourci Partager

L'équivalent du signet, dans le menu Partager de Safari. Stéphanie l'a déjà créé sur son
iPhone et le transmet à Frédéric par AirDrop, pas besoin de le reconstruire à la main.

> La première fois, iOS demande d'autoriser l'exécution de JavaScript sur la page : il
> faut accepter, sinon le raccourci ne peut pas lire la page produit.

### Le créer à la main (si l'AirDrop n'est pas possible)

Vécu le 18/08/2026 : cette création manuelle est **fastidieuse**, l'interface de
Raccourcis a changé selon les versions d'iOS et les étapes ci-dessous peuvent ne pas
correspondre exactement à ce qui s'affiche. Préférer l'AirDrop dès que possible.

1. Ouvre l'app **Raccourcis**, appuie sur **+**.
2. Renomme le raccourci **Ajouter à la déco**.
3. Ouvre le panneau **Détails** (l'icône a varié selon les versions : un rond « i », ou
   une icône de réglages/curseurs). Vérifie que **Dans la feuille de partage** est
   activé.
4. Le réglage « Types d'entrée acceptés » n'existe plus séparément sur les versions
   récentes d'iOS. Pas bloquant : l'action ajoutée à l'étape suivante ne fonctionne que
   dans Safari, ce qui suffit à limiter le raccourci à ce contexte.
5. Ajoute l'action **Exécuter JavaScript sur la page web**.
6. Remplace tout le contenu de l'action par le code ci-dessous, en remplaçant
   `ADRESSE_DE_LAPP` par l'adresse réelle de fav-deco.
7. Ajoute enfin l'action **Show Web View** (cherche « web view » ou « vue web » dans le panneau
   d'actions, le libellé exact varie selon la version d'iOS), branchée sur le résultat du
   JavaScript. **Pas « Ouvrir les URL »** : cette action ouvre un nouvel onglet Safari à chaque
   fois, qui reste ensuite ouvert, ce qui devient vite gênant quand on ajoute plusieurs articles
   à la suite. Show Web View affiche la page dans une fenêtre d'aperçu qui se referme d'elle-même.

```javascript
var d = document, h = location.href;
function m(){for(var i=0;i<arguments.length;i++){var e=d.querySelector('meta[property="'+arguments[i]+'"],meta[name="'+arguments[i]+'"]');if(e&&e.content)return e.content}return''}
var o = {url:h, titre:'', image_url:'', prix:null, boutique:location.hostname.replace(/^www\./,'')};
var b = d.querySelectorAll('script[type="application/ld+json"]');
for (var i=0; i<b.length && !o.titre; i++){
  try{
    var j = JSON.parse(b[i].textContent), p = Array.isArray(j) ? j.slice() : [j];
    while (p.length){
      var n = p.shift();
      if (!n || typeof n != 'object') continue;
      if (n['@graph']) { p = p.concat(n['@graph']); continue; }
      var t = n['@type'];
      if (!(t == 'Product' || (Array.isArray(t) && t.indexOf('Product') > -1))) continue;
      o.titre = n.name || '';
      var im = n.image; if (Array.isArray(im)) im = im[0];
      if (im && typeof im == 'object') im = im.url || im.contentUrl;
      if (im) o.image_url = im;
      var f = n.offers; if (Array.isArray(f)) f = f[0];
      if (f && f.price != null) o.prix = parseFloat(String(f.price).replace(',', '.'));
      break;
    }
  } catch(e){}
}
if (!o.titre) o.titre = m('og:title','twitter:title') || d.title || '';
if (!o.image_url) o.image_url = m('og:image','og:image:secure_url','twitter:image');
if (o.prix == null){ var q = m('product:price:amount','og:price:amount');
  if (q){ var v = parseFloat(String(q).replace(/[^\d.,]/g,'').replace(',','.')); if (!isNaN(v)) o.prix = v; } }
if (!o.image_url){ var bi = null, ar = 0;
  [].forEach.call(d.images, function(x){ var a = x.naturalWidth * x.naturalHeight;
    if (a > ar && x.naturalWidth > 220){ ar = a; bi = x; } });
  if (bi) o.image_url = bi.currentSrc || bi.src; }
if (o.image_url){ try { o.image_url = new URL(o.image_url, h).href; } catch(e){} }
o.titre = (o.titre || '').replace(/\s+/g,' ').trim().slice(0,180);

completion("ADRESSE_DE_LAPP#add=" + encodeURIComponent(JSON.stringify(o)));
```

**À l'usage :** sur la page du produit dans Safari, bouton Partager, puis
**Ajouter à la déco**. fav-deco s'ouvre pré-rempli dans une fenêtre d'aperçu qui se referme
d'elle-même une fois l'ajout fait, sans laisser d'onglet ouvert.

> La première fois, iOS demande l'autorisation d'exécuter du JavaScript sur la page.
> C'est normal, il faut accepter, sinon le raccourci ne peut pas lire la page.

---

## 3. Partout : coller le lien

Le bouton **Ajouter** de l'app accepte un lien collé. L'app tente de retrouver la photo
toute seule.

Ça marche sur beaucoup de sites, mais **pas sur IKEA, La Redoute ni Kave Home**, pour la
raison expliquée plus haut. Dans ce cas l'app te le dit et te laisse remplir le nom et la
photo à la main. Pour récupérer l'adresse d'une photo sur Mac : clic droit sur l'image,
**Copier l'adresse de l'image**, puis colle dans le champ Photo.

Une fiche sans photo reste une fiche valable : le lien est gardé, tu peux compléter plus
tard.
