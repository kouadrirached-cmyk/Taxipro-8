# TaxiPro — Contexte du projet

## Vue d'ensemble
TaxiPro est une application de gestion de flotte de taxis pour petits propriétaires-opérateurs au Québec. Développée par Amin, propriétaire de 3 Toyota Prius (véhicules 707, 718, 732) chez Taxis Unis de Chicoutimi (9420-1001 Québec Inc.).

**Objectif actuel : transformer TaxiPro d'un outil interne en produit SaaS vendable à d'autres propriétaires de flottes (~35 $/mois par flotte).**

## Architecture technique
- **Un seul fichier** : toute l'application vit dans `index.html` (HTML + CSS + JS inline). NE PAS diviser en plusieurs fichiers sans demande explicite.
- **PWA** hébergée sur GitHub Pages : https://kouadrirached-cmyk.github.io/Taxipro-8
- **Synchronisation** : Firebase Firestore (multi-appareils, temps réel).
- **Pas de backend serveur** : tout est côté client + Firestore.
- Le propriétaire travaille **exclusivement sur Android mobile, sans PC**. L'interface doit être mobile-first à 100 %.

## Logique métier critique (NE JAMAIS casser)
1. **Paie 60/40** : le chauffeur reçoit 40 % des recettes, le propriétaire 60 %. Exception : quand le propriétaire (Amin) conduit lui-même, il reçoit 100 %.
2. **Surplus visible par le propriétaire seulement** : les chauffeurs ne voient jamais les données financières globales ni le surplus.
3. **Séparation des permissions chauffeur / propriétaire** : rôles distincts, le chauffeur ne voit que ses propres courses et sa paie.
4. **Mots de passe** : hachés en SHA-256, jamais en clair.
5. **Un quart ne doit JAMAIS valoir zéro parce que le nom a changé (v117)** : `calcSalary` et `partPatronQuart` rendaient 0 dès que `getDrv(s.driver)` ne trouvait personne — nom corrigé, chauffeur retiré, orthographe différente. Le salaire du chauffeur ET la part du patron disparaissaient en silence alors que l'argent avait été gagné et versé. `pctQuart(s)` prend le taux de l'équipe si le chauffeur existe, sinon le `drvPct` **enregistré sur le quart** (écrit à la saisie pour ça). Ne jamais revenir à un retour 0 silencieux.

### Les paies versées (`C.payes`)
- Clé : `"AAAA-MM-JJ|Nom du chauffeur"`, où la date est le **lundi** de la semaine, en heure LOCALE (`_wkKeyDe`). Ne pas fabriquer cette clé avec `toISOString()` : au Québec les deux coïncident, mais pas dans un fuseau à l'est de Greenwich — ce qui compte pour la revente du produit.
- La clé contient le NOM : renommer un chauffeur doit passer par `renommerChauffeur()`, qui déplace aussi les clés `C.payes` (et les suffixes `_deductCash` / `_essCash`).
- Cocher une semaine « payée » crée de **vraies sorties de trésorerie** (`payeKey` sur la transaction). Un rattrapage en masse ne doit donc JAMAIS rejouer ces paiements : `payerToutJusqua()` ne fait que solder le passif (`horsApp:true`), sans toucher à la trésorerie, et consigne un lot annulable dans `C.payeLots`.
- **Le rattrapage ne touche JAMAIS la semaine en cours (v120)** — une semaine qui n'est pas finie ne peut pas avoir « déjà été versée hors app ». La v117 pré-remplissait la date avec la semaine la PLUS RÉCENTE : un seul appui marquait payée la paye pas encore versée, le bouton « 💵 Payer » disparaissait et le patron ne pouvait plus payer ses chauffeurs. Le défaut par défaut est désormais `derniereSemaineTerminee()`, le champ porte un `max`, et `payerToutJusqua()` filtre `d.wk < semaine en cours` même si la date est forcée.
- **Un solde de rattrapage s'affiche « 🧹 Soldé … — déjà versé hors app »**, jamais « ✅ Payé le » : sinon il se confond avec un vrai versement. Et quand plus rien n'est dû mais qu'un lot existe, le titre de la carte dit « soldé en bloc » au lieu d'un « à jour » rassurant.
- **Bannière de déblocage en haut de la page Paye (v122)** : dès qu'un lot de rattrapage existe, `soldeEnBlocBanniereHTML()` affiche la date, le montant, « aucun argent n'est sorti de ta trésorerie » et un bouton « ↩️ Remettre ces semaines à payer ». L'annulation était enfouie dans une carte repliée : le patron voyait « Soldé » au lieu de « Payer », croyait l'app cassée, et restait bloqué.
- **LE PATRON PAIE QUAND IL VEUT (v122)** — ne jamais retirer le bouton « 💵 Payer » d'une semaine. L'app affichait « Paye lundi prochain » SANS bouton pendant la semaine en cours : un chauffeur qui finit le mercredi ne pouvait pas être payé, et le patron se retrouvait en retard sans recours. `isPaid` et `allPaid` ne dépendent plus de `isCurWk`.
- **Payer en milieu de semaine impose le complément** : `payerComplement(wkKey,drvName)` compare ce qui a été versé (`C.payes[k].net`) avec ce qui est dû maintenant, affiche « ⚠️ X gagnés depuis le paiement » et verse la seule différence, avec sa propre sortie de trésorerie et une trace dans `complements[]`. Sans ça, les quarts roulés après le paiement seraient avalés par le « payé ».

## Mécanismes de synchronisation (fragiles — manipuler avec précaution)
- **Tombstones** : les suppressions sont marquées (tombstones) et non effacées, pour éviter qu'un appareil hors ligne ressuscite des données supprimées.
- **Verrou d'hydratation (hydration lock)** : empêche les données locales périmées d'écraser Firestore au chargement.
- **Écritures par lots atomiques** (Firestore batch writes) : toute écriture multi-documents doit rester atomique.
- **Écriture transactionnelle (v116) — NE PAS REVENIR À `batch.set` SEUL** : `set` remplace le document entier, donc un appareil en retard effaçait du cloud ce qu'il n'avait pas reçu (quarts disparus, chauffeur ajouté qui « ne paraît pas »). Fusionner avant d'écrire ne suffit pas : entre la lecture et l'écriture, l'autre téléphone écrit, et sa nouveauté est perdue — c'est mesuré, pas théorique. `_fb.save()` écrit donc dans une **transaction Firestore** (`runTransaction`), qui surveille les documents lus et rejoue tout si quelqu'un les modifie entre-temps. Toutes les lectures doivent précéder les écritures, et **le callback ne doit jamais toucher à l'état local** (il peut être rejoué) : la récupération locale se fait après, dans `_recuperer()`. Repli en `batch` si la transaction échoue, car une transaction exige le réseau et l'app doit marcher hors ligne.
- **`_cm` n'est tamponné que si la config CHANGE (v116)** : `lSave()` comparait rien et posait `Date.now()` à chaque passage, y compris pour les sauvegardes qui ne modifient rien (hydratation, rendus, battement d'annuaire). Un appareil périmé se déclarait ainsi le plus frais et écrasait pour de bon. L'empreinte `_cfgEmpreinte(C)` (config sans `_cm`) décide. Toute adoption d'une config venue d'ailleurs doit mettre à jour `window._cfgEmpreinteVue`.
- **Chauffeurs et véhicules : union par `id`** — ils ne sont JAMAIS supprimés, seulement désactivés (`active=false` via `togDrv`/`togVeh`). L'union à l'écriture est donc sûre et garantit qu'aucune écriture ne peut faire disparaître un chauffeur. Si un jour une suppression franche est ajoutée, il faudra des tombstones AVANT de garder cette union.
- **`REGISTRES_QUARTS` : union obligatoire (v118)** — `C.justifs` (l'explication que le chauffeur écrit sur son manque), `C.ecartsRegles` et `C.ecartsDebits` sont classés par identifiant de quart et voyagent dans le document de config, faute d'un document à eux. Sans union, l'appareil qui écrit la config effaçait ceux qu'il n'avait pas encore reçus : la justification d'un chauffeur n'atteignait jamais le patron. On n'y retire JAMAIS une entrée (on ajoute ou on bascule), donc `_unionRegistres()` est sûre. **Si une suppression franche est ajoutée à l'un d'eux, le sortir de `REGISTRES_QUARTS` ou lui donner des tombstones.**
- **Config du cloud plus récente ≠ ne rien écrire (v118)** : quand `dc._cm > C._cm`, on ne remplace pas les réglages du cloud par les nôtres — mais on repart de la config du cloud (horodatage compris) pour y poser nos ajouts aux registres. Avant, l'écriture était abandonnée en entier et ces ajouts étaient perdus en silence : c'est ce qui arrivait au téléphone d'un chauffeur dont l'horloge retardait, ou dès que le patron venait de changer un réglage.
- **Le doc `deleted` (tombstones) reste en dernier-écrivain-gagne** : surtout ne pas en faire l'union, car « ♻️ Remettre un quart perdu » retire volontairement un tombstone — une union le remettrait et re-supprimerait le quart.
- **BUG HISTORIQUE À NE JAMAIS RÉINTRODUIRE** : `location.reload()` appelé avant la fin des écritures asynchrones Firebase = perte de données. Toujours `await` toutes les écritures avant tout reload.
- **Resynchronisation au retour au premier plan (v113)** : iOS suspend la page en arrière-plan et coupe le canal temps réel ; Safari la restaure ensuite depuis son cache SANS rien réexécuter, laissant les `onSnapshot` accrochés à un canal mort. `txpResync()` rebranche les écoutes ET relit les documents à chaque `visibilitychange`, `focus`, `pageshow` et `online`, plus un veilleur si plus rien n'arrive pendant 2 minutes. NE PAS retirer ces écouteurs : sans eux, les iPhone cessent silencieusement de synchroniser. Transport réglé en `experimentalAutoDetectLongPolling`.
- **Sécurité Firestore (v65)** : chaque appareil s'authentifie anonymement (Firebase Auth) et s'inscrit dans le doc `members` de sa flotte via `window._fbEnsure()`. Les règles (`firestore.rules`) n'autorisent lecture/écriture qu'aux membres. TOUJOURS `await window._fbEnsure()` avant toute opération Firestore directe. À la création d'une flotte, le doc `members` doit être écrit AVANT le batch des documents de données.

## Fonctionnalités existantes (v63 et plus — le code fait foi, pas cette liste)
- Paie automatique 60/40 avec gestion propriétaire-conducteur
- Analyse financière par véhicule : coût/km, revenu/km, rentabilité
- Calendrier de planification des véhicules (quarts jour / soir)
- Tableau de bord des contributions des chauffeurs avec médailles
- Ventilation des profits quotidiens sur 7 jours
- Architecture multi-tenant en mode démo
- Bouton « 🔄 Forcer la mise à jour » pour contourner le cache PWA

## Diagnostic de synchronisation (v121)
- **Réglages → « 🩺 Où sont les quarts de mes chauffeurs »** : lit le document `shifts` du cloud **en direct** (`{source:'server'}`) et le compare à `S` pour une semaine choisie, chauffeur par chauffeur. Il tranche entre les trois causes de « mon chauffeur a travaillé mais je ne le vois pas » : pas descendu ici (bouton `recupererQuartsDuCloud()`), jamais monté, ou chauffeur sur une autre flotte. Ne jamais remplacer cette lecture serveur par un `get()` ordinaire : le cache local renverrait l'état de l'appareil et le diagnostic s'auto-confirmerait.
- `recupererQuartsDuCloud()` passe par `txpMerge` : une suppression volontaire (tombstone) ne peut pas ressusciter.
- **Le chauffeur voit son code de flotte** au bas de son tableau de bord, avec le nombre de quarts sur son téléphone et s'ils sont partis. Il n'a pas accès aux Réglages : sans cette ligne, il n'avait aucun moyen de donner ce code à son patron. Ne rien ajouter d'autre là : pas de montants globaux, pas de données des autres chauffeurs.

## Quand des quarts n'arrivent pas (v123)
- **`lSaveLocal()` ne doit JAMAIS avaler un échec.** Il avait un `catch(e){}` muet : mémoire du téléphone saturée (deux ans de quarts + HUIT sauvegardes hebdomadaires contenant chacune une copie complète de C+S+EX), l'enregistrement des quarts échouait sans un mot, et au rechargement l'app relisait l'ancienne liste. Des quarts bien reçus disparaissaient — « il en a rentré 5, j'en vois 3 ». Désormais : on sacrifie les `txpro_bkp_*` les plus anciennes une par une en réessayant, et si ça ne passe toujours pas on pose `window._stockagePlein=true`, affiché en rouge chez le patron ET chez le chauffeur. Mesuré sur la v122 : écriture refusée → rien de libéré, aucun signal, l'app continue.
- **Écriture additive des quarts et des dépenses** : `set(ref,{data:…},{merge:true})` avec un marqueur de suppression explicite pour les tombstones. Un appareil ne peut plus qu'AJOUTER ses quarts ; il lui est structurellement impossible d'effacer ceux qu'il n'a pas reçus, même si la transaction a cédé sous la contention et qu'on est retombé sur le lot. Ne pas revenir à un `set` complet. Réserve connue : le `merge` fusionne champ par champ à l'intérieur d'un quart, donc un champ RETIRÉ d'un quart ne se propage pas — les quarts gardent une forme fixe, c'est sans conséquence aujourd'hui.
- **`envoiStatut()` dit la vérité sur l'envoi.** `_fbReady` ne signifie que « le SDK a démarré » : la ligne du chauffeur annonçait « envoyés » alors que Firestore refusait ses écritures (appareil non membre, flotte verrouillée, identité anonyme perdue après un vidage de cache). `_fb.save()` retient maintenant `window._fbDernierEchec={code,ts}`, et le statut affiche « ⛔ Tes quarts ne partent PAS » avec la raison et la marche à suivre. Priorité : mémoire pleine > refus > attente > hors ligne > ok.
- Le diagnostic du patron affiche en plus : **appareils autorisés**, **flotte verrouillée O/N**, **taille du document des quarts** (limite Firestore ~1 Mo). Une flotte verrouillée refuse tout nouveau téléphone — et « 🔄 Forcer la mise à jour » peut suffire à faire perdre son autorisation à un appareil.

## Pièges connus
- **Cache PWA** : les mises à jour ne s'affichent pas sans invalidation du cache / bouton de forçage. Toujours vérifier le service worker après modification.
- **Erreurs de syntaxe JS = crash silencieux** : un seul fichier signifie qu'une erreur casse toute l'app. Toujours valider la syntaxe avant de pousser.
- **UNE DONNÉE ABÎMÉE NE DOIT PAS VIDER UN ÉCRAN (v120)** : une exception pendant le rendu laisse la page **blanche**, sans message. C'est arrivé pour de vrai — un quart dont `date` valait `null` faisait planter `parseLocalDate` → `salairesDus()` → `renderPaye()`, et le patron ne voyait plus aucune paye ni aucun bouton « Payer ». Trois garde-fous : `parseLocalDate` rend une date invalide au lieu de lever (`dateValide(ds)` teste le format), les agrégations de paie ignorent les quarts sans date valide, et `goPage()` enveloppe le rendu dans un `try/catch` qui affiche « ⚠️ Cet écran n'a pas pu s'afficher » avec le renvoi vers « 🔍 Vérifier les quarts ». **Ce filet est un dernier recours, pas une excuse** : si un test voit le message de panne au lieu des données, c'est un échec.
- **Tester la logique de sync** avant de pousser : simuler deux appareils si possible.

## Conventions
- **Langue de l'interface : français (Québec)**. Termes : « chauffeur », « course », « quart », « paie », « recettes ».
- Devise : dollars canadiens (CAD). **Deux formateurs, à ne pas confondre** : `fm()` écrit `$104.83` et sert aux écrans de l'app (ainsi depuis le début, inchangé pour ne pas bouleverser l'affichage) ; `fmQc()` écrit `104,83 $` à la québécoise et sert aux **documents qui sortent de l'app** — feuille de taxes, papiers du comptable — parce qu'ils peuvent être ouverts sur un ordinateur en anglais.
- Dates : format québécois (jour mois année).
- **TOUTE date `AAAA-MM-JJ` passe par `ymdLocal()` (v119) — JAMAIS `toISOString()`.** `toISOString()` rend l'heure UTC : au Québec, à partir de 20 h (19 h l'hiver), il renvoie **demain**. Mesuré : la vue « Jour » affichait « Recette totale 0,00 $ · 0 quarts » tous les soirs à partir de 20 h — exactement quand les chauffeurs de soir rentrent leur quart — et le formulaire se pré-remplissait à la date du lendemain, si bien que les quarts de soir étaient **enregistrés au mauvais jour**. `today()`, `inDay()` et toutes les clés de semaine passent désormais par `ymdLocal()` (vérifié : au Québec les clés de semaine produites sont identiques aux anciennes sur 3 ans, changements d'heure compris — aucune clé `C.payes` n'est orpheline). Réparation des quarts déjà mal datés : `quartsDateDecalee()` / `corrigerDatesDecalees()` dans Réglages → « 🔍 Vérifier les quarts ». La signature retenue est stricte — date POSTÉRIEURE au moment de la saisie ET saisie faite après 19 h — pour ne jamais toucher à une saisie d'avance volontaire.
- Commentaires de code : en français de préférence.

## Déploiement
- Push sur `main` → GitHub Pages redéploie automatiquement (1-2 min).
- Après déploiement, l'utilisateur doit utiliser « 🔄 Forcer la mise à jour » dans l'app.

## Feuille de route produit (SaaS)
1. ✅ Inscription autonome (v64) : un nouveau propriétaire crée son compte, sa flotte et ses chauffeurs sans intervention manuelle.
2. ✅ Isolation stricte des données par flotte (v65) : Auth anonyme + doc `members` par flotte + règles `firestore.rules` (à coller dans la console Firebase). Bouton « Verrouiller la flotte » dans Réglages.
3. ✅ Annuaire des flottes (v104) : collection `annuaire`, une fiche par flotte déposée par la flotte elle-même à chaque ouverture. Console « Qui utilise l'application » dans Réglages, flotte principale seulement : compte les propriétaires, suspend (réversible) ou supprime une flotte. Les gardiens sont listés dans `annuaire/_admins` — **la place se prend une seule fois, à revendiquer dès le déploiement**.
4. Page d'accueil de vente avec essai gratuit 30 jours (champs `createdAt`/`trialEndsAt` déjà dans la config de flotte depuis v64).
5. Paiement par abonnement via Stripe Checkout.
6. Politique de confidentialité conforme à la Loi 25 (Québec) — **rendue nécessaire par la mesure d'usage (v107)** : il faut y déclarer Google Analytics et le droit de refus.

## Photos et pièces jointes (v112)
- **NE JAMAIS mettre d'image dans `C`, `S` ou `EX`.** Ces trois objets partent chacun en UN document Firestore, plafonné à 1 Mo : deux photos suffisent à bloquer la synchronisation de toute la flotte.
- Les photos de factures vivent dans **IndexedDB** (base `txpro_factures`), compressées à 1400 px / JPEG 0,68 (~150 Ko). Seuls les renseignements légers (date, fournisseur, montant, lien vers la dépense) vont dans `C.factures`.
- Conséquence assumée : une photo reste sur l'appareil qui l'a prise. L'export « Dossier pour le comptable » produit un seul fichier HTML contenant le tableau et toutes les images.
- Une facture ne crée jamais d'argent toute seule : elle documente une dépense existante ou propose de la créer (`factureVersDepense`). Sinon un même achat serait compté deux fois.

## Mesure d'usage (v107)
- Google Analytics 4 via `gtag.js`, éteint tant que `GA_MESURE_ID` est vide (réglage d'essai par appareil dans `localStorage['txpro_ga_id']`).
- **Liste blanche stricte** : `GA_EVENEMENTS` et `GA_PARAMS` dans `index.html`. Tout nom d'événement ou paramètre absent de ces listes est jeté, et **aucune valeur numérique ne sort jamais**. Ne jamais élargir ces listes à un nom, un courriel ou un montant.
- Coupable par flotte : bouton dans Réglages → `C.mesureUsage=false` + `localStorage['txpro_ga_off']` (vaut aussi hors ligne).
