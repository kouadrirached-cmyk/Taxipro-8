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
- **BUG HISTORIQUE À NE JAMAIS RÉINTRODUIRE** : `location.reload()` appelé avant la fin des écritures asynchrones Firebase = perte de données. Toujours `await` toutes les écritures avant tout reload.

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
- Devise : dollars canadiens (CAD), format `1 234,56 $`.
- Dates : format québécois (jour mois année).
- Commentaires de code : en français de préférence.

## Déploiement
- Push sur `main` → GitHub Pages redéploie automatiquement (1-2 min).
- Après déploiement, l'utilisateur doit utiliser « 🔄 Forcer la mise à jour » dans l'app.

## Feuille de route produit (SaaS)
1. Inscription autonome : un nouveau propriétaire crée son compte, sa flotte et ses chauffeurs sans intervention manuelle.
2. Isolation stricte des données par flotte via les règles de sécurité Firestore.
3. Page d'accueil de vente avec essai gratuit 30 jours.
4. Paiement par abonnement via Stripe Checkout.
5. Politique de confidentialité conforme à la Loi 25 (Québec).
