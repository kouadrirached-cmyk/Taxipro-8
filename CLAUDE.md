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

## Mécanismes de synchronisation (fragiles — manipuler avec précaution)
- **Tombstones** : les suppressions sont marquées (tombstones) et non effacées, pour éviter qu'un appareil hors ligne ressuscite des données supprimées.
- **Verrou d'hydratation (hydration lock)** : empêche les données locales périmées d'écraser Firestore au chargement.
- **Écritures par lots atomiques** (Firestore batch writes) : toute écriture multi-documents doit rester atomique.
- **Écriture transactionnelle (v116) — NE PAS REVENIR À `batch.set` SEUL** : `set` remplace le document entier, donc un appareil en retard effaçait du cloud ce qu'il n'avait pas reçu (quarts disparus, chauffeur ajouté qui « ne paraît pas »). Fusionner avant d'écrire ne suffit pas : entre la lecture et l'écriture, l'autre téléphone écrit, et sa nouveauté est perdue — c'est mesuré, pas théorique. `_fb.save()` écrit donc dans une **transaction Firestore** (`runTransaction`), qui surveille les documents lus et rejoue tout si quelqu'un les modifie entre-temps. Toutes les lectures doivent précéder les écritures, et **le callback ne doit jamais toucher à l'état local** (il peut être rejoué) : la récupération locale se fait après, dans `_recuperer()`. Repli en `batch` si la transaction échoue, car une transaction exige le réseau et l'app doit marcher hors ligne.
- **`_cm` n'est tamponné que si la config CHANGE (v116)** : `lSave()` comparait rien et posait `Date.now()` à chaque passage, y compris pour les sauvegardes qui ne modifient rien (hydratation, rendus, battement d'annuaire). Un appareil périmé se déclarait ainsi le plus frais et écrasait pour de bon. L'empreinte `_cfgEmpreinte(C)` (config sans `_cm`) décide. Toute adoption d'une config venue d'ailleurs doit mettre à jour `window._cfgEmpreinteVue`.
- **Chauffeurs et véhicules : union par `id`** — ils ne sont JAMAIS supprimés, seulement désactivés (`active=false` via `togDrv`/`togVeh`). L'union à l'écriture est donc sûre et garantit qu'aucune écriture ne peut faire disparaître un chauffeur. Si un jour une suppression franche est ajoutée, il faudra des tombstones AVANT de garder cette union.
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

## Pièges connus
- **Cache PWA** : les mises à jour ne s'affichent pas sans invalidation du cache / bouton de forçage. Toujours vérifier le service worker après modification.
- **Erreurs de syntaxe JS = crash silencieux** : un seul fichier signifie qu'une erreur casse toute l'app. Toujours valider la syntaxe avant de pousser.
- **Tester la logique de sync** avant de pousser : simuler deux appareils si possible.

## Conventions
- **Langue de l'interface : français (Québec)**. Termes : « chauffeur », « course », « quart », « paie », « recettes ».
- Devise : dollars canadiens (CAD). **Deux formateurs, à ne pas confondre** : `fm()` écrit `$104.83` et sert aux écrans de l'app (ainsi depuis le début, inchangé pour ne pas bouleverser l'affichage) ; `fmQc()` écrit `104,83 $` à la québécoise et sert aux **documents qui sortent de l'app** — feuille de taxes, papiers du comptable — parce qu'ils peuvent être ouverts sur un ordinateur en anglais.
- Dates : format québécois (jour mois année).
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
