# Projet Mathemia — Livrets d'accueil

App web de livrets d'accueil numériques pour locations courte durée (Primo Conciergerie).
Dirigeant / dev : Mathieu Morel. Document de référence du projet : `Livrets_accueil_recap_complet.docx`.

## Stack
- **Front** : un seul fichier `index.html` (~3260 lignes, ~405 Ko), JavaScript vanilla, design noir/or premium.
- **Libs CDN** : axios, qrcodejs, @vimeo/player, jspdf, **SortableJS** (drag-drop multi-listes).
- **Backend** : Supabase — `https://iduognfftslrcresthuu.supabase.co` (bucket images `livret-images`, file_size_limit 5 MB).
- **Hébergement** : Vercel — `livrets-accueil.vercel.app`. Routing `/livret/{slug}` côté JS.
- **Repo GitHub** : `ConciergerieNevers/-livrets-accueil` (privé, connecté à Vercel). Ce dossier local n'est PAS un repo git ; Mathieu pousse via l'interface web GitHub (copier-coller).

## Architecture du HTML
4 écrans dans le même fichier, affichés/cachés en JS :
- `#login-screen` — connexion admin (Supabase Auth).
- `#admin-screen` — tableau de bord (stats, QR, **PDF refondu**, dupliquer, supprimer).
- `#editor-screen` — éditeur multi-onglets.
- `#voyageur-screen` — vue publique `/livret/{slug}` : panneaux home, infos, guide, adresses, services, contact.

## Tables Supabase (état actuel après toutes les évolutions)

### `livrets`
Colonnes principales : nom, slug, bienvenue, vimeo_url, wifi_nom, wifi_mdp, checkin, checkout, parking, regles[], poubelles, hote, adresse, latitude, longitude, contact_nom, contact_tel, contact_email, image_fond, lien_avis_google, lien_reservation_directe, lien_autres_logements, cta1_texte, cta2_texte, cta3_texte, procedure_arrivee (texte legacy).
**Colonnes ajoutées récentes (toutes IF NOT EXISTS)** :
- `boutons_video jsonb default '[]'` — chapitres-boutons sous la vidéo home
- `procedure_etapes jsonb default '[]'` — étapes carrousel d'arrivée (photo+texte)
- `logo_url text` — logo personnalisable dans l'avatar Contact

### `adresses` (FK livret_id)
Colonnes principales : nom, categorie, description, url, ordre.
**Colonnes ajoutées** :
- `type text default 'recommandation'` — valeurs : `recommandation` ou `lien_utile`
- `image_url text` — photo perso uploadée
- `icone text` — emoji libre OU clé `ADRESSE_ICONS` OU clé `ILLUSTRATIONS` (rétrocompat)

### `equipements` (FK livret_id)
icone, nom, description, image_url, video_url, ordre. Drag-droppable, vidéo Vimeo intégrée dans la carte voyageur si video_url est Vimeo.

### `services` (FK livret_id)
titre, description, image_url, prix, lien_paiement, ordre. Drag-droppable. Photo carrée via cropper.

### `consultations` · `avis_negatifs`
Tracking vues + formulaire privé (avis 1-3 étoiles).

## SQL ajouté à la base (cumulatif — tous rejouables IF NOT EXISTS)
```sql
ALTER TABLE public.livrets ADD COLUMN IF NOT EXISTS boutons_video jsonb DEFAULT '[]'::jsonb;
ALTER TABLE public.adresses
  ADD COLUMN IF NOT EXISTS type text DEFAULT 'recommandation',
  ADD COLUMN IF NOT EXISTS image_url text,
  ADD COLUMN IF NOT EXISTS icone text;
ALTER TABLE public.livrets ADD COLUMN IF NOT EXISTS procedure_etapes jsonb DEFAULT '[]'::jsonb;
ALTER TABLE public.livrets ADD COLUMN IF NOT EXISTS logo_url text;
UPDATE storage.buckets SET file_size_limit = 5242880 WHERE id = 'livret-images';
```

## Onglets de l'éditeur

| ID | Onglet | Contenu |
|---|---|---|
| `#tg` | **Général** | nom, slug, bienvenue, image de fond |
| `#tv` | **Vidéo** | lien Vimeo + procédure d'arrivée TEXTE (legacy) + **section boutons sous vidéo** (chapitres timecodes) + **section étapes d'arrivée avec photos** (carrousel) |
| `#ti` | **Infos** | adresse, hôte, GPS, wifi, check-in/out, parking, règles, poubelles |
| `#teq` | **Guide** | équipements drag-drop ⋮⋮ + bouton "📥 Importer les chapitres Vimeo" + picker visuel 53 illustrations |
| `#ta` | **Adresses** | 2 sections : 📍 Recommandations + 🌐 Liens utiles. Drag-drop, photo/icône, picker dédié ADRESSE_ICONS, ✨ Auto par catégorie |
| `#tsv` | **Services +** | services payants Stripe, photo carrée (cropper), drag-drop |
| `#tc` | **Contact** | **logo personnalisable** (avatar rond) + nom/tel/email + 3 CTAs |

## Fonctionnalités clés
- **Multilingue** (10 langues via Google Translate).
- **QR codes** livret + wifi auto-connexion.
- **PDF refondu** bilingue FR/EN (jspdf) : grand bandeau hôte, contact, check-in/out, procédure, wifi+QR, règles, poubelles, parking, urgences.
- **Avis intelligents** : 4-5★ → redirection auto Google ; 1-3★ → formulaire privé `avis_negatifs`.
- **Boutons sous vidéo** : chapitres avec timecode (seek dans la vidéo) OU URL externe. Import auto depuis chapitres Vimeo (`getChapters()`).
- **Guide équipements** : drag-drop, vidéo Vimeo intégrée inline au timecode, picker illustrations.
- **Adresses 2 sections + style inversé** : Recommandations sombre + Liens utiles or massif. Flèche → cliquable bottom-right.
- **Banque icônes adresses dédiée** : 37 icônes + auto-détection par catégorie.
- **Procédure d'arrivée carrousel** : étapes photo 1:1 + texte + compteur "N/total" en gros doré top-left. Fallback texte legacy.
- **Logo Contact personnalisable** : upload dans onglet Contact, remplace l'emoji 🏠.
- **Services +** : photo carrée 1:1 cropper, drag-drop, vignette voyageur 108×108.

## Bibliothèques visuelles

### ILLUSTRATIONS (banque Guide — 53)
Icônes line-art 24×24 stroke or. Clés : `sofa, bed, suite, kitchen, bath, shower, toilet, tv, multimedia, speaker, washer, dishwasher, laundry, coffee, flame, fireplace, snow, oven, microwave, fridge, wifi, trash, vacuum, key, keybox, electricmeter, car, evcharge, leaf, patio, pergola, balcony, shutter, pool, spa, sauna, gym, gameroom, bbq, bike, hello, closing, lock, lamp, music, film, desk, chicken, message, garden, star, door, access`.

### ILLUSTRATIONS_RICH (scènes complètes 640×360)
Scènes éditoriales (defs gradients, particules, mobilier en line-art or). Style validé sur "Entrée" puis décliné aux 53 clés.

### CHAPTER_THEMES + CHAPTER_DESCRIPTIONS + VIS_LABELS
Regex mots-clés → clé d'illustration + emoji fallback + dégradé. Map des descriptions courtes (pré-remplies à l'import). Map des libellés français pour le picker.

### ADRESSE_ICONS (37) + ADRESSE_LABELS + ADRESSE_KEYWORDS
Bank SÉPARÉE de la Guide, focus lieux : restaurant, pizza, café, asiatique, vin, bar, boulangerie, glacier, fromagerie, etc. + libellés FR + regex pour la baguette magique ✨ Auto.

## Globals JS importants
`equip`, `adr`, `serv`, `btnv`, `procEtapes` (arrays) · `imgFond`, `logoUrl` (strings) · `voyVimeoPlayer` (instance Vimeo.Player du voyageur) · `currentLivretData` (objet livret du voyageur).

## Helpers JS clés
- `normalizeVimeoUrl(url)`, `vimeoEmbedUrlWithTime(url)`, `isVimeoUrl()`, `isVimeoCdnThumb()`
- `parseTimecode("1:30")` / `formatTimecode(90)`
- `vimeoSeekTo("1:30")`, `playEqVideo(posterEl, embedUrl)`
- `findChapterTheme(title)`, `findAdrIcon(text)`, `autoDetectAdrIcon(idx)`
- `getThemeForChapter(e, index)`, `generateThemeSvg(theme)`
- `pickCols(row, cols)` — normalisation clés pour batch INSERT (fix PGRST102)
- `procSlide(direction)`, `setupProcCarouselScroll()` — carrousel étapes
- `showLogoPreview(url)`, `resetLogoPreview()`, `removeLogo()` — logo Contact

## Schémas COLS_xxx (à respecter pour batch INSERT PostgREST)
```js
COLS_ADRESSES = ['livret_id','nom','categorie','description','url','ordre','type','image_url','icone'];
COLS_EQUIPEMENTS = ['livret_id','icone','nom','description','image_url','video_url','ordre'];
COLS_SERVICES = ['livret_id','titre','description','image_url','prix','lien_paiement','ordre'];
```
⚠️ Si on ajoute une colonne à une de ces tables → mettre à jour ces arrays sinon la donnée ne sera pas sauvegardée.

## Leçons de debug critiques

### GRANT vs RLS (`42501`)
Erreur "permission denied" → vérifier les GRANT AVANT les policies RLS. Postgres a 2 niveaux. `information_schema.role_table_grants`. La fonction `logAndShowErr` affiche l'erreur brute dans le bandeau rouge.

### PGRST102 "All object keys must match"
Toutes les lignes d'un batch INSERT doivent avoir le même set de clés. Solution : `pickCols(row, COLS_XXX)` avant l'insert.

### PGRST204 "column not found in schema cache"
La colonne n'existe pas (encore). Toujours fournir le SQL `ALTER TABLE ... ADD COLUMN IF NOT EXISTS` AVANT le push GitHub.

### PGRST303 JWT expired
Demander reconnexion.

### Storage 400 "object exceeded maximum allowed size"
Le bucket `file_size_limit` est trop bas. Fix :
```sql
UPDATE storage.buckets SET file_size_limit=5242880 WHERE id='livret-images';
```

### macOS pbcopy encodage
Sans forçage UTF-8 les accents cassent en `L'exp√©rience`. Toujours :
```bash
LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8 cat index.html | LANG=en_US.UTF-8 pbcopy
```
Vérifier : `LANG=en_US.UTF-8 pbpaste | grep -c "L'expérience"` doit retourner `1`.

### Détection auto par mot-clé : attention aux substrings
- `tri` matchait dans "élec**TRI**que" → utiliser `\btri\b`.
- `électrique` matchait dans `evcharge` ET dans "Compteur électrique" → restreindre les regex (utiliser `borne (de )?recharge|wallbox|tesla` au lieu de `électrique` seul).
- Ordre critique dans `CHAPTER_THEMES` : entrées spécifiques (electricmeter) AVANT entrées génériques (evcharge).

## État du livret réel "Le Cosy Green" (id `fcb129ee-a789-4f0a-bc2f-cbcb2312d8e1`)

- **Hôte** : Antoine · **Adresse** : 34 côte de conflans, 58180 Marzy · **GPS** : 46.97370, 3.07242
- **Contact** : Camille & Sophie · +33744318931 · nevers@primoconciergerie.fr
- **Wifi** : SFR_E04F / ksmf34kxiswzpqlsjps1 · **Check-in** 16h / **Check-out** 11h
- **Vidéo Vimeo** : `https://player.vimeo.com/video/1150383065?h=33843a3c0a`
- **12 boutons-vidéo chapitres** : Début, Accès, Entrée, Cuisine, Pièce de vie, Télévision, Chambres, Salle de bains, Autres services, Chauffage, Compteur électrique, Nos services +
- **10 équipements Guide** (avec timecodes Vimeo) : Accès, Entrée, Cuisine, Pièce de vie, Télévision, Chambres, Salle de bains, Chauffage, Compteur électrique, Nos services +
- **12 recommandations** : Le Bengy, Café Vélo, O Wok 58, L'Envie, La Simplicité, La Fontaine Cavalier, La Gabare, Couleur Café, Une Note de Vin, Les Nougatines de Nevers, Faïencerie Georges, Parc floral d'Apremont
- **8 liens utiles** : Office de Tourisme, À voir/à faire, Les musées, Le patrimoine, Visites, Se restaurer, La Loire, Taxis

Le bouton **"✨ Charger l'exemple Cosy Green"** dans l'éditeur reflète cet état réel (mis à jour). Sert de modèle pour démarrer un nouveau livret.

## En cours
- Prochain livret à insérer : **"La Chapelle" (Nevers)** — hôtes Camille et Gwendoline · Tel +33744318931 · nevers@primoconciergerie.fr · Wifi `LA CHAPELLE`/`!lachapelle!` · Parking cour intérieure portail+digicode · Check-in 16h / Check-out 11h · Règles non-fumeur/animaux/soirées · Poubelles mercredi & vendredi. Bonnes adresses : Chez Couron, Café Vélo, Le Saint Sébastien, Le Bengy, L'Envie, Les Nougatines de Nevers, La Simplicité, La Fontaine Cavalier, La Gabare. À compléter manuellement : vidéo Vimeo, photos, adresse exacte, GPS.

## Workflow pour livrer une modif
1. Modifier `index.html` localement (via Edit tool dans Claude Code).
2. Si la modif touche une colonne Supabase → donner le SQL `ALTER TABLE ... IF NOT EXISTS` à exécuter AVANT le push.
3. Si on touche au schéma adresses/equipements/services → mettre à jour aussi les `COLS_XXX` arrays.
4. Copier dans le presse-papier en forçant UTF-8 :
   ```bash
   LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8 cat index.html | LANG=en_US.UTF-8 pbcopy
   ```
5. Vérifier : `LANG=en_US.UTF-8 pbpaste | grep -c "L'expérience"` → `1`.
6. Mathieu va sur https://github.com/ConciergerieNevers/-livrets-accueil/blob/main/index.html → crayon ✏️ → Cmd+A → Cmd+V → "Commit changes".
7. Vercel redéploie automatiquement en 1-2 min. Mathieu fait Cmd+Shift+R pour vider le cache.

## Préférences de collaboration
- Français.
- Explications simples (Mathieu n'est pas développeur de formation).
- Toujours rappeler ce qu'est un commit / le presse-papier / un cache navigateur quand on touche à ces concepts.
- Diff minimal : ne pas refactorer ce qui marche, garder le fichier unique tant que possible.
- Quand on ajoute une colonne Supabase : toujours donner le SQL `IF NOT EXISTS` rejouable et prévenir que sans ça l'app va planter avec `PGRST204`.
- Quand on ajoute une catégorie à la banque Guide : ajouter à `ILLUSTRATIONS` + `ILLUSTRATIONS_RICH` + `VIS_LABELS` + `CHAPTER_THEMES` (regex) + `CHAPTER_DESCRIPTIONS`.
- Quand on ajoute une catégorie à la banque Adresses : ajouter à `ADRESSE_ICONS` + `ADRESSE_LABELS` + éventuellement `ADRESSE_KEYWORDS`.

---

## Intégration SuperHote V2 (lien voyageur personnalisé)
- **Principe** : chaque voyageur reçoit un lien unique (inséré dans le mail automatique SuperHote). Le livret s'ouvre avec son prénom/nom + dates + bannière. Pas de webhook SuperHote → API interrogée à la volée.
- **Format du lien** : `https://livrets-accueil.vercel.app/?r=<booking_id>&k=<nom>&d=<date_arrivée>&f=<url_facture>&g=<url_guest_page>&ci=<heure_checkin>&co=<heure_checkout>` (variables SuperHote dans le template mail).
- **Edge Function `guest-lookup`** (déployée, **verify_jwt ON**, clé anon) :
  - Voyageur `?r&k&d` → réservation par **recherche dichotomique** sur `/reservations` (ids décroissants), cache dans `sh_reservations`, vérifie **nom OU date** (anti-curieux).
  - Admin `?rentals=1` → liste des logements (paginé) pour le menu de l'éditeur (`requireUser`).
  - Secret : `SUPERHOTE_TOKEN` (jamais dans l'app).
- **App (`index.html`)** : `lbres(bookingId,lastName,checkinDate,invoiceUrl,guestPageUrl,checkinTime,checkoutTime)` au `DOMContentLoaded` si `?r=` → appelle `SH_FN`, trouve le livret via `livrets?superhote_rental_id=eq.{rental_id}`, rend via `rv()`, puis `showGuestBanner`/`showResa`/`showFacture`.
- **Éléments voyageur** :
  - Bannière `#voy-guest` "👋 Bienvenue [nom]" + compte à rebours "J-X avant votre arrivée" (disparaît le jour J). ⚠️ **z-index** : `position:relative;z-index:2` sinon cachée derrière `.vh-bg`.
  - Bloc "Votre réservation" (`#voy-resa-zone`) avant le wifi : dates, heures, nuits, voyageurs, "Modifier mes horaires" → guest page.
  - Titre doré "Votre arrivée en toute simplicité".
  - Bouton facture (`#voy-facture-zone`, param `f`).
  - **Wifi déplacé** de l'accueil → onglet Infos.
- **Éditeur** : `<select id="f-superhote-rental">` + `loadSuperhoteRentals(selectedId)` ; `saveLivret` envoie `superhote_rental_id`.
- **SQL** : `livrets.superhote_rental_id bigint` + table `sh_reservations` (cache).

## Catalogue de services centralisé ("Mes services")
- Créer un service UNE fois, cocher quels logements le vendent (plus de doublons).
- **Table `catalogue_services`** : `id, titre, description, prix (text), image_url, lien_paiement, livret_ids uuid[], categorie, demande_date bool, jours_dispo int[] (0=dim..6=sam), delai_jours int, heure_limite int, ordre, actif, created_at`. RLS : SELECT public, ALL authenticated ; GIN index sur `livret_ids`. (Voir `catalogue-migration.sql`.)
- **Migration** : anciens services par livret regroupés (`INSERT … GROUP BY titre, array_agg(DISTINCT livret_id)`).
- **Back-office (`#catalogue-screen`, `openCatalogue`)** : liste compacte par catégorie, recherche, drag-drop (SortableJS `catReorder`). Édition inline (`editCatRow`/`catEditHtml`) : titre, catégorie, prix, description, photo, case **"Demander date+heure"** (`catf-demdate`), **jours dispo** (`catf-jours`), **délai** (`catf-delai`), **heure limite** (`catf-hlimite`), cases logements (`toggleCatLivret`).
- **Onglet Services éditeur** : cases à cocher du catalogue (`loadEditorServices`/`toggleEditorService`, PATCH `livret_ids`).
- **Voyageur** : `rv()` n'affiche QUE le catalogue (`catalogue_services?livret_ids=cs.{id}&actif=eq.true`).

## Panier + paiement Stripe combiné
- **Panier** : tableau `cart` basé sur `lineId` (service daté = nouvelle ligne par ajout). `addToCart/changeQty/removeCartLine/addAnotherDay/clearVoyCart`, persistance `localStorage` (`saveCart`). ⚠️ `renderCart` re-synchronise depuis `voyServices` (réglages live).
- **Calendrier custom** (le natif ne grise pas / ne se ferme pas) : `toggleCal/calNav/pickCalDay/calGrid`. `bookableDay(c,dt)` : deadline = `dt - delai_jours` à `heure_limite`. Dispo+futur+à temps = cliquable ; passé = gris ; non-dispo OU trop tard = rouge barré.
- **Sélecteur d'heure custom** "🕐 Heure souhaitée" : `toggleHeure/pickHeure/heureGrid` (7h-20h).
- **Pas de TVA ajoutée.**
- **`create-checkout`** (déployée, ⚠️ **clé ANON** — service_role = "permission denied" ici) : lit `services` + `catalogue_services`, construit la Checkout Session (price_data inline, `billing_address_collection=required`, `phone_number_collection`, `invoice_creation`), génère `crypto.randomUUID()` + INSERT `commandes` (`en_attente`, pas de RETURNING car RLS anon). Renvoie `{ok,url,order_id}`. (Voir `edge-function-create-checkout.ts`.)
- **`stripe-webhook`** (déployée, ⚠️ **verify_jwt OFF**) : `constructEventAsync` ; sur `checkout.session.completed` → UPDATE `commandes` statut=`paye` + `guest_nom/email/tel` via SERVICE_KEY. (Voir `edge-function-stripe-webhook.ts`.)
- **Table `commandes`** : `id, livret_id, livret_nom, guest_nom, guest_email, guest_tel, items jsonb, demande, total_cents, statut, stripe_session, created_at`. RLS : anon INSERT, authenticated SELECT/UPDATE.
- **Back-office "Mes demandes"** (`#demandes-screen`, `openDemandes`/`renderDemandes`) : liste les commandes (récent→ancien) avec contact, items, demande, total, badge statut.
- **Compte Stripe = SARL MATHEMIA (live)**. Secrets Supabase : `STRIPE_SECRET_KEY` + `STRIPE_WEBHOOK_SECRET`. Changer de compte = remplacer ces secrets, aucun code à toucher.

## Leçons clés
- **service_role → "permission denied for table services"** ici : utiliser la clé **anon** dans les Edge Functions qui lisent `services`/`catalogue_services`.
- **RLS anon sans SELECT** : `insert().select()` renvoie vide → `crypto.randomUUID()` côté fonction.
- **Bannière cachée** = z-index derrière `.vh-bg` → `position:relative;z-index:2`.
- **Vérif syntaxe JS** avant livraison : `jsc` (JavaScriptCore).
- **Calendrier natif** ne grise pas / ne se ferme pas → widget custom.
- Secrets (Stripe, SuperHote, whsec) collés **UNIQUEMENT** dans Supabase/Stripe, **jamais dans le chat**.

---

# === MISE À JOUR MAJEURE (mai 2026) — Outil de conciergerie "Be Nomad" ===

## ⚠️ Marque
- **Le nom n'est plus "Primo" → c'est "Be Nomad".** Ne plus afficher "Primo" dans les nouvelles UI (dossier/repo gardent leur nom).
- Toujours **1 fichier `index.html`** (~4460 lignes) + Edge Functions + Vercel.

## Edge Functions (5, déployées via dashboard Supabase)
1. **guest-lookup** (verify_jwt ON, anon) : voyageur `?r&k&d` (+ renvoie `invoices` = factures services du booking_id) · admin `?rentals=1` · admin **`?sync=<page>`** (synchro réservations → upsert `sh_reservations`, lots de 5, `cleanDate` + fallback ligne-par-ligne, renvoie `synced/failed/err/last_page/next/done/sample_keys`).
2. **create-checkout** (ANON) : panier → Checkout + commande `en_attente` ; reçoit/stocke `booking_id`, `demande`.
3. **stripe-webhook** (verify_jwt OFF) : `checkout.session.completed` → `paye` + contact + `invoice_url` (`stripe.invoices.retrieve`) + `payment_intent` ; `charge.refunded` → `rembourse`/`rembourse_partiel`. ⚠️ ajouter l'événement `charge.refunded` à l'endpoint.
4. **refund-item** (NOUVELLE, verify_jwt ON, admin) : remboursement partiel d'un service (`stripe.refunds.create`), item `statut:'refuse'`, `refunded_cents`.
5. **admin-checkout** (NOUVELLE, verify_jwt ON, admin) : commande sur-mesure prix libre → Checkout + commande, stocke `checkout_url`.

## SQL ajouté (IF NOT EXISTS)
- `commandes` : `booking_id`, `invoice_url`, `payment_intent`, `refunded_cents int default 0`, `traitee boolean default false` + policy DELETE authenticated.
- `livrets` : `actif boolean default true`, `categorie text`, `ordre int default 0`.
- `sh_reservations` : policy SELECT authenticated **+ `grant all ... to service_role`**.
- Table **`parametres`** (`cle` PK, `valeur`) — CGV (clé `cgv`), RLS select public / ALL authenticated.

## Front
- **Achat** : panier → "Suivant" → page paiement plein écran (`#voy-payscreen`) + demande particulière + **case CGV obligatoire** + Payer → **page "Paiement confirmé !"** (`showPaymentSuccess`).
- **Facture services sur le livret** (`showServiceInvoices` ← `g.invoices`).
- **Splash chargement voyageur** (`startIntro/finishIntro`) : ✦ Be Nomad ✦ + "Bienvenue [Prénom]" cursive **Dancing Script** dorée qui **se trace de gauche à droite** (effet écriture, `@keyframes introWrite` clip-path ~1,6 s) + lueur + spinner, **3 s** (`finishIntro`, constante `3000`).
- **"Mes demandes" = dashboard** : KPIs + graphe CA mois/semaine + top services + par logement + onglets ; fiche détail (✏️ Modifier item, 🚫 Refuser & rembourser, 🔗 lien paiement, **✅ Marquer traitée** `markTraitee`/`traitee`, supprimer) ; **pastille** = nb à traiter ; **➕ commande sur-mesure** (`openNewOrder`).
- **Planning** (`planningHtml`) Mois/Semaine/Jour, Par personne / Par heure.
- **Réservations** (`#reservations-screen`) : cache `sh_reservations` (filtre `checkout>=2026-05-01` fixe), recherche + filtres + regroupé par logement ; 👁 Voir / 🔗 Copier / ➕ Commande ; 🔄 Synchroniser + auto-sync à l'ouverture.
- **Accueil admin** : ⚙️ vue Cartes/Liste (localStorage) + ⇅ Réorganiser (drag-drop entre catégories `lvDropEnd`) + 🏷 Catégories + statut Actif/Pause (PATCH `actif`, pause = indisponible voyageur) + recherche logements ; KPIs = Livrets / Vues 30j / Upsell (mois).
- **CGV** : éditeur dans "Mes services" (`saveCGV`, table `parametres`) + lien voyageur (`showCGV`) + case obligatoire paiement.

## Stripe (SARL MATHEMIA, LIVE)
- E-mail facture auto = activer **"Paiements réussis"** dans E-mails clients ; mode **Live** obligatoire ; **délai de finalisation facture = immédiat (1 s)**.

## Leçons clés (NOUVELLES)
- **`sh_reservations` : `grant all ... to service_role` OBLIGATOIRE** sinon les fonctions n'écrivent rien (échec silencieux). Diag : `select count(*)` + insert manuel + remonter l'erreur d'upsert.
- **Upsert par lot atomique** : une ligne invalide casse tout le paquet → `cleanDate` (null si non ISO) + fallback ligne-par-ligne.
- **API SuperHote `/reservations`** = seulement id/rental_id/status/platform/checkin/checkout/nights/adults/children/guests/first_name/last_name/total_price/currency_id/external_id/dates → **pas** de guest-page/facture/heures → lien voyageur complet impossible à reconstruire (uniquement via messages SuperHote). **Pas d'API messagerie SuperHote.**
- Réservations triées par date de création (id ↓) → synchro complète = paginer tout (~384 pages).

## Reste à faire / décisions
- **Page d'accueil admin "exceptionnelle" Be Nomad** : pas encore construite.
- Vérifier déploiements en attente (SQL `traitee`, `livrets actif/categorie/ordre`, table `parametres`, GRANT sh_reservations, fonctions refund-item/admin-checkout, événement Stripe charge.refunded).
- **Accès multi-utilisateurs : ABANDONNÉ** (pas d'usine à gaz).

---

# === MODULE STOCKS DE CONSOMMABLES (septembre 2026) ===

Écran `#stock-screen` (`openStock`), bouton **📦 Stocks** dans la nav admin, pastille = nb de produits sous le seuil.
Conso déduite **automatiquement** des réservations SuperHote : **1 séjour terminé = 1 rotation**.

## Tables
- **`consommables`** : nom, categorie, unite, cout_unitaire, fournisseur, stock_actuel, seuil_alerte, conditionnement, actif, ordre. (26 lignes — Café en grain, Chocolat en poudre, Lait et Miel supprimés volontairement par Mathieu le 03/09/2026 : non utilisés)
- **`conso_formats`** : paquets d'achat — consommable_id, nom ("Boîte de 54 dosettes"), qte (54), cout_lot (8,45), fournisseur, defaut. Un produit peut avoir plusieurs formats. (28 lignes)
- **`conso_regles`** : consommable_id, `base` (**personne** = ×voyageurs 1 fois · **personne_nuit** = ×voyageurs×nuits · **rotation** = 1 fois par séjour · **nuit** = ×nuits · **sdb** = ×salles de bain), quantite, `updated_at` (trigger `touch_conso_regle` — une modif vaut à partir de sa date, jamais rétroactivement), `condition_type` (machine_cafe | lave_vaisselle | lave_linge | petit_dej), condition_valeur, **`livret_id`** (NULL = règle générale ; renseigné = règle propre à un logement, qui **remplace** la règle générale du même produit pour ce logement).
- **`stock_mouvements`** : consommable_id, `type` (sortie | entree | ajustement), quantite, livret_id/nom, booking_id, nb_personnes, nb_nuits, cout, note, format_id, format_nom, nb_lots, date_mouvement.
  ⚠️ Index unique `uniq_sortie_booking (booking_id, consommable_id) WHERE type='sortie'` → **empêche tout double décompte**.
- **`livrets`** nouvelles colonnes : `conso_geree bool`, `machine_cafe text`, `nb_sdb int`, `a_lave_vaisselle bool`, `a_lave_linge bool`, `petit_dej_offert bool`.
- **`parametres`** clé `conso_start` : date de départ du suivi (défaut = 1er du mois courant). Sans ça le 1er calcul avalerait ~900 rotations d'historique.

## 4 onglets
1. **📦 État** — alertes en tête, puis par catégorie : stock, conso/mois, **autonomie en jours** (rouge ≤7j, orange ≤21j, vert) + **date de rupture prévisionnelle** d'après les réservations déjà prises, seuil, P.U. Sélecteur « Moyenne calculée sur 6 / 12 derniers mois » (localStorage `stk_ref_mois`). Champ « Suivi depuis le » + bouton « 📈 Charger l'historique des coûts » (`loadHistoCouts` → `syncConso({histoOnly:true})`, n'écrit PAS le stock).
2. **📥 Réapprovisionner** — 1 carte par produit, 2 saisies : ① « Il en reste (compté) » → **écart d'inventaire** enregistré en `type:'ajustement'` (= perte/vol) ; ② paquet + nb de paquets → unités calculées. Aperçu live (`majReaLigne`). Le prix du lot **recale `cout_unitaire`** automatiquement. `journalHtml()` en bas.
3. **📊 Statistiques** — rotations, voyageurs, nuitées, logements, coût, €/rotation, €/voyageur (dernier mois) + tableau mois par mois + quantités consommées avec **ratio par voyageur** + total **manquant vs surplus** en €.
4. **⚙️ Logements & règles** — par logement : stock géré, machine à café, nb SdB, LV, LL, petit-déj. Puis toutes les règles éditables inline.

## Fonctions clés
`openStock` · `loadStockData` (charge consommables + règles + mouvements + livrets + **formats**) · `setStockTab` · `renderStock` · `consoParJour` / `refConso` / `consoDunSejour` / `consoSurPeriode` / `dateRupture` / `setRefMois` · `stockKPIs` · `stockEtatHtml` / `stockReapproHtml` / `stockStatsHtml` / `stockCoutsHtml` / `stockConfHtml` · `majReaLigne` · `doReappro` / `doReapproAll` · `journalHtml` · `syncConso(opts)` · `getConsoStart` / `setConsoStart` · `editConso` / `addFormat` / `delFormat` · `addRegle` / `saveRegle` / `delRegle` · `saveConsoConf` · `statsMois`

## Paramétrage réel (validé par Mathieu)
- **Hors stock (8)** : Villa Bali, La Chapelle, Montagnon, Vivier, Saint-Cyr, Le Nacha, Loft Manhattan, Villa du Circuit. **23 gérés.**
- **Machines** : Senseo 15 · Nespresso 4 · Tassimo 2 · Dolce Gusto 2 · aucune 5 (dont Bois Bézard).
- **Petit-déj offert** : LE NOEUD VERT + Studio Majorelle uniquement — 6 produits conditionnés `petit_dej`, tous en base **rotation** (1 fois par séjour, PAS par jour) : 2 croissants, 2 pains au chocolat, 1 jus, 1 confiture, 2 Nutella, 5 fruits.
- **Domaine du Bois Bézard** : plusieurs machines → `machine_cafe = NULL` + 3 **règles dédiées** (30 caps Dolce Gusto, 18 caps Nespresso, **2 bouteilles de vin** / rotation).
- **Ratios** : eau 1/pers · **thé 1/pers** · sucre 4/pers · capsules 2/pers · PQ 0,333/pers/nuit · Sopalin 0,0154/pers/nuit · éponge, sacs noir+tri 1/rotation · pastille LV 1/rotation si LV · pastille LL 1/rotation si LL · sac SdB 1/SdB · gel douche 0,048 L/SdB.
- **Gel douche** : compté en **litres** (recharges 2 L), pas en flacons. ⚠️ coût 9,25 €/L dérivé du flacon 400 ml — **prix de la recharge à corriger** (décision Mathieu : plus tard).
- **Seuils d'alerte** = 2 semaines de conso réelle (calculée sur juin-août 2026).
- Conso ≈ **1 497 €/mois** sur 23 logements. Stock de départ saisi le 03/09/2026 = 1 326 €.
- Seuils recalés par SQL à chaque changement de périmètre (requête de recalage dans l'historique de session).

## ⚠️ Calcul de l'autonomie — NE PAS revenir en arrière
L'autonomie **ne se calcule PAS** sur l'historique de `stock_mouvements` (trop court au démarrage → chiffres absurdes). Elle se calcule sur les **vraies réservations SuperHote** (`window.stockResas`, 400 jours chargés) auxquelles on applique les **règles actuelles** : `consoDunSejour(livret, pers, nuits)` → `consoSurPeriode(d1,d2)` → `refConso()` (cache `_refCache`, invalidé à chaque modif de règle). `dateRupture(cid)` déroule les réservations **futures** jour par jour jusqu'à épuisement → date réelle, pas une extrapolation.

## Écarts de prix relevés dans le tableau de Mathieu (non tranchés)
Son onglet « coût moyen » contredit ses lignes d'achat : Sopalin 0,40 vs **2,34 €** · pastille LV 0,12 vs **0,456 €** · PQ 0,47 vs **0,33 €**. Se corrigera tout seul au 1er réappro via `cout_lot`.
