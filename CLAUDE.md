# TaxiPro — Contexte du projet

> ## ⏪ LE RETOUR EN ARRIÈRE EST TERMINÉ — ce qu'il a prouvé
> **`index.html` est en v127** (`2026-10-29-v127-DIAGNOSTICS-ET-FLOTTE-SUSPENDUE`) : la v126 plus
> le bandeau d'alerte « flotte suspendue ». Les v116 à v126 sont donc toutes de retour.
>
> Déroulé : à la demande d'Amin, `index.html` a été ramené à la v116 (6 oct) le 28 octobre, puis
> à la v115 (27 sept) le même soir, pour éliminer le code comme cause. **Le problème a persisté
> aux deux reculs.** Le 29 octobre, Amin a exigé que ce soit réglé ; la v126 a été remise avec
> ses diagnostics, puisque la v115 ne permettait plus de rien observer.
>
> **Ce que le test a établi, et qu'il ne faut pas refaire :** le code n'est pas la cause. Six
> scénarios mesurés en émulateur n'ont jamais réussi à perdre un quart (détail plus bas). Quand
> Amin dit « avant ça marchait », ne pas repartir des mises à jour : la bascule du 20 octobre
> n'a correspondu à **aucun déploiement** (la v116 tournait sans changement du 6 au 21).
>
> Raison : l'app a fonctionné quatre mois sans incident, puis des quarts de chauffeurs ont cessé
> d'arriver chez le patron. Mesure certaine : le cloud ne contient qu'UN des six quarts d'un
> chauffeur pour la semaine du 19 au 25, alors que son téléphone les affiche tous avec « Sync ✓ ».
> Ses écritures ne montent pas. Le but du retour en arrière est d'éliminer le code comme cause.
>
> **Résultat acquis : le problème persiste sur la v116.** Les v117 à v126 sont donc écartées.
> Tentative de prouver que la v116 est coupable (un appareil à liste incomplète qui écraserait
> les autres) : **NON reproduite** en émulateur. Ne pas l'affirmer sans preuve.
>
> ### Ce qui a été mesuré sur la v115 le 28 octobre — le code est écarté
> Amin a proposé sa propre piste : « parfois vert, parfois nuageux, parfois rouge », donc le
> réseau qui va et vient. Sondes en émulateur sur la v115 **en service** :
>
> | Scénario | Le quart monte-t-il ? |
> |---|---|
> | Coupure, le réseau revient, app ouverte | ✅ oui |
> | Coupure, il ferme l'app, le réseau revient plus tard | ✅ oui |
> | Idem avec `enablePersistence` en échec (iPhone à mémoire saturée) | ✅ oui |
> | Envoi accroché qui n'aboutit ni n'échoue | ❌ **non** |
>
> **Aucune perte par le code n'a pu être reproduite**, ni sur la v115 ni sur la v116 : écoute
> temps réel morte, téléphone hors réseau au moment d'écrire, écriture tronquée mise en file qui
> part plus tard, persistance en panne, app fermée. GARDE 3 relit le cloud (ou son cache, qui
> garde les quarts des autres) et réécrit l'union à chaque fois. **Ne pas accuser le code sans
> une nouvelle mesure.**
>
> **UNE VRAIE FAIBLESSE TROUVÉE, à corriger quelle que soit la cause** : `setSyncStatus('saving')`
> est posé AVANT `await batch.commit()`, et `window._fbPending` ne passe à `true` que dans le
> `catch`. Un envoi qui reste accroché sans lever d'erreur laisse donc le badge nuageux, le
> drapeau à `false`, et **la relance de 30 s ne part jamais** — vérifié sur 40 s. Il n'y a ni
> délai d'attente ni signal. Même trou dans GARDE 1 (`shiftsCount===0 && prevShiftsCount>5`) et
> GARDE 2 (`!hasDrivers`) : elles sortent par `return` sans marquer `_fbPending`, donc ce qui est
> bloqué là ne sera jamais réessayé.
>
> **Piste écartée, à ne pas relancer** : le plafond de 20 accès de documents des règles Firestore
> (`isMember()` + `estSuspendue()` = 4 accès × 5 documents du lot). Testé avec les vraies
> `firestore.rules` et la fiche d'annuaire présente : **le lot passe**. Réserve : l'émulateur
> peut être plus permissif que la production sur ce plafond.
>
> **Conséquence** : la cause est hors du code, côté Firebase (refus d'écriture, appareil qui perd
> son autorisation). Or **la v115 est aveugle** — `window._fbDernierEchec`, `envoiStatut()`,
> `renvoyerMesQuarts()` et `baliseEtat()` datent des v123 à v125 et ont été retirés par le retour
> en arrière. Remonter à la v126 a été **recommandé à Amin** pour capter le code d'erreur du
> serveur sur le téléphone du chauffeur ; en attente de sa décision, ne pas basculer sans elle.
>
> ### « Le problème commence le 20 » — Amin, 28 octobre
> Sa mesure : avant le 20 tout passait, après le 20 plus rien. **Rien n'a été déployé le 20** :
> la v116 tournait sans changement du 6 au 21 octobre (v117 = `4a63df5`, 21 oct 14 h 33). La
> bascule n'est donc pas une mise à jour.
>
> Son seul geste de configuration, dans ses mots : « parfois je monte pour un chauffeur le
> pourcentage à 41, juste ça ». **Testé, deux appareils, v115 : le 41 % tient et les 6 quarts du
> chauffeur montent tous.** Piste écartée, ne pas la relancer sans nouvelle mesure.
>
> **CE QUI EST MESURÉ, ET QUI COLLE À LA DATE** : la v115 date les quarts avec
> `today()` = `toISOString()` (ligne 1968), posé dans le formulaire ligne 2768. En octobre au
> Québec (UTC−4), **les sept quarts de soir sur sept sont enregistrés au lendemain** ; un quart de
> jour saisi à 14 h est correct. Un quart roulé **le lundi 19 au soir est inscrit au 20** — d'où
> « ça commence le 20 » du point de vue du patron. Et le quart du **dimanche 25 au soir atterrit
> dans la semaine du 26**. Si un chauffeur rentre toute sa semaine d'un coup un dimanche soir,
> TOUS ses quarts quittent la semaine.
>
> **Donc « Lamine 1 / 1 » ne prouve pas une perte** : le diagnostic n'a interrogé que la semaine
> du 19 au 25. Les cinq autres quarts peuvent être dans le cloud, rangés sous la semaine du 26.
> Amin a été invité à regarder les semaines du 12 et du 26 dans son app. **Ne pas conclure à une
> perte de quarts avant cette vérification.**
>
> C'est la **v119** (25 oct) qui a corrigé ce défaut de date, et le retour en arrière l'a remis en
> service : depuis le 28 octobre au soir, les quarts de soir sont de nouveau mal datés. Le
> diagnostic des cinq semaines voisines (`out.semaines`, v126) et la réparation
> `corrigerDatesDecalees()` (v119) sont eux aussi hors service — ce sont précisément les outils
> qui trancheraient. Argument supplémentaire pour remonter à la v126.
>
> ### 🎯 CAUSE LA PLUS PROBABLE : la flotte est SUSPENDUE dans l'annuaire
> Fait décisif donné par Amin le 28 octobre : **« même pour cette semaine, aucune paye rentrée,
> aucune notification »**. La semaine du 26 est vide aussi — donc les quarts ne sont pas mal
> rangés, **rien n'arrive du tout**, et l'absence de notification confirme qu'aucun quart neuf
> n'atteint son téléphone.
>
> `firestore.rules` : `allow read, write: if estFlotte() && isDataDoc() && isMember() &&
> !estSuspendue();` — une flotte suspendue perd lecture ET écriture. **MAIS la règle suivante**,
> `allow read, write: if estFlotte() && isDataDoc() && isAdmin();`, rend son accès complet à un
> **gardien de l'annuaire**. Amin a été invité à revendiquer cette place, et la console « Qui
> utilise l'application » (v104) pose le bouton `⏸️ Suspendre` (`basculerSuspension`,
> `flotteSuspendre` à la ligne 8057) sur **chaque** carte de flotte, la sienne comprise, juste à
> côté de `🗑️ Supprimer`.
>
> **Sonde `sonde-flotte-suspendue.js` : TOUS les symptômes reproduits d'un coup.**
>
> | Symptôme rapporté | Mesure |
> |---|---|
> | Les quarts des chauffeurs n'arrivent plus | ⛔ `permission-denied` sur leur téléphone |
> | Le patron travaille normalement | ✅ il lit **et** écrit sans rien remarquer |
> | « Flotte non verrouillée » au diagnostic | ✅ `locked` et `suspendu` sont deux drapeaux DIFFÉRENTS |
> | Aucune notification | ✅ rien n'arrive, `notifEntrants` n'a rien à annoncer |
> | Reculer le code deux fois n'a rien changé | ✅ c'est de la donnée cloud, insensible au code |
> | Bascule un 20, sans déploiement | ✅ un seul appui, à n'importe quel moment |
> | Le chauffeur voyait « ☁️ Sync ✓ » | ✅ badge vert mesuré PENDANT le refus |
>
> **La réactivation fait repartir les envois immédiatement** : les 3 quarts en attente sont montés
> seuls. Les quarts ne sont donc pas perdus, ils attendent sur les téléphones — il suffit que
> chaque chauffeur rouvre l'app (le renvoi du démarrage, `Object.keys(S).length >
> Object.keys(remoteS).length`, s'en charge).
>
> Réserve : la console Firebase d'Amin étant fermée, cet état n'a pas pu être lu directement. Il
> lui a été demandé de regarder l'étiquette de sa propre carte dans ⚙️ Réglages → « 📇 Qui utilise
> l'application » (« ⏸️ Suspendue » en rouge, puis « ▶️ Réactiver »). **Ne pas l'affirmer comme
> établi avant sa réponse.**
>
> **À CORRIGER QUELLE QUE SOIT L'ISSUE** — ce piège doit devenir impossible :
> 1. `estSuspendue()` doit apparaître dans le diagnostic du patron, à côté de « flotte
>    verrouillée » : les deux drapeaux sont distincts et seul `locked` était affiché.
> 2. `basculerSuspension()` doit **refuser ou avertir fortement** quand la cible est la flotte de
>    l'appareil lui-même — un gardien ne voit aucun effet sur son propre écran, donc rien ne le
>    prévient qu'il vient de couper ses chauffeurs.
> 3. Le chauffeur refusé doit le SAVOIR : c'est ce que fait `envoiStatut()` (v123) avec
>    `window._fbDernierEchec`, retiré par le retour en arrière. Argument de plus pour la v126.
>
> ### 🎯 LA CAUSE : `invalid-argument` — un document dépasse 1 Mo
> **Mesuré sur le téléphone d'Amin le 29 octobre**, dès la v127 déployée : son bandeau affiche
> « ⚠️ Cet appareil n'arrive pas à écrire dans le cloud — Code du serveur : **`invalid-argument`** ».
> Son téléphone de PATRON est touché lui aussi : **personne n'écrit**.
>
> `invalid-argument` ne vient JAMAIS des permissions — ni flotte verrouillée, ni suspension, ni
> appareil non autorisé. Firestore rejette le **contenu**. Deux causes réelles : un document qui
> dépasse **1 Mio**, ou une valeur interdite (`undefined`, `NaN`, fonction, imbrication > 20).
>
> **Et les 5 documents partent dans UN SEUL lot atomique** : un seul document fautif bloque tout,
> **les quarts compris**. C'est ce qui fait qu'un appareil cesse d'envoyer en ayant l'air normal.
>
> **Reproduit en émulateur** (`test-invalid-argument.js`) : une config gonflée à 1 774 Ko par
> `C.treso` fait répondre au serveur exactement `invalid-argument`. Le mécanisme est établi.
>
> Pourquoi ça colle à tout ce qu'Amin décrit : la config grossit tous les jours (`C.treso` =
> toute la trésorerie, `C.payes` = chaque semaine × chaque chauffeur, plus `messages`, `justifs`,
> `cleans`, `schedules`, `giftCards`, `factures`…), donc elle franchit le mur **un jour précis** —
> le 20 — sans qu'aucun déploiement n'ait eu lieu ; **tous** les appareils sont bloqués d'un coup
> puisqu'ils écrivent tous la même config ; les lectures continuent de marcher ; et **reculer le
> code n'y change rien**, ce qui explique l'échec des deux retours en arrière.
>
> **POURQUOI ON NE L'A PAS VU PENDANT SIX JOURS** : le diagnostic v123 n'affichait que la taille
> du document des **quarts** (306 Ko, rassurant). La **config** n'a jamais été mesurée. Toute
> mesure de taille doit désormais porter sur les CINQ documents.
>
> Outils ajoutés (v128) : `diagnosticEcriture()` mesure les 5 documents et les plus gros postes de
> `C` ; `valeursRefusees()` situe une valeur interdite par son chemin exact ;
> `verdictEcriture()` écrit la cause en une phrase dans le bandeau ;
> `montrerDiagnosticEcriture()` donne le rapport complet, copié dans le presse-papier.
>
> **MESURE SUR SON TÉLÉPHONE (29 oct, v128) : LA TAILLE N'EST PAS LA CAUSE.**
> shifts 307 Ko · config 103 Ko · audit 84 Ko · deleted 3 Ko · expenses 2 Ko — **aucun ne
> s'approche des 1024 Ko**, et le détecteur local n'a trouvé aucune valeur interdite. L'hypothèse
> « document trop gros » est donc **écartée pour sa flotte** (le mécanisme reste réel et reproduit,
> mais ce n'est pas ce qui le bloque). Ne pas y revenir sans nouvelle mesure.
>
> **L'ERREUR À NE PLUS REFAIRE** : `_fbDernierEchec` ne gardait que `e.code`. Or Firestore écrit
> la raison **en toutes lettres** dans `e.message` — « Nested arrays are not supported (found in
> document taxipro/config) », « Unsupported field value: undefined », « exceeds the maximum
> allowed size ». Jeter ce message a coûté une soirée. **Toujours conserver `e.message`.**
>
> Ajouts v129 : `_fbDernierEchec.msg` (message entier, affiché dans le bandeau et dans le
> rapport) ; `sonderEcriture()` écrit les **5 documents un par un** et laisse le serveur désigner
> le fautif avec son message — sans danger, ce sont les écritures que l'app fait déjà, celles des
> quarts et des dépenses étant additives ; `valeursRefusees()` détecte en plus les **tableaux
> imbriqués** (cause classique d'`invalid-argument` qu'aucune mesure de taille ne voit) et les
> **noms de champs réservés** `__x__`, et scrute désormais les cinq charges, audit et tombstones
> compris. Corrigé au passage : `navigator.clipboard.writeText` rend une promesse, dont le rejet
> n'était pas attrapé.
>
> ## 🎯 LA CAUSE, DITE PAR LE SERVEUR (29 octobre, v129)
> ```
> Code du serveur    : invalid-argument
> Message du serveur : too many index entries for entity /taxipro/shifts
> ```
> **Firestore limite un document à 40 000 ENTRÉES D'INDEX.** C'est une limite **distincte de la
> taille** et elle ne se voit pas en kilo-octets : le document d'Amin ne pesait que **307 Ko**.
> Firestore indexe **chaque champ de chaque quart** — ~25 champs × 2 index = **~50 entrées par
> quart** — donc le mur tombe vers **800 quarts**. Quatre mois de travail, puis un jour précis.
>
> Voilà pourquoi rien d'autre ne collait : taille normale, aucune valeur interdite, aucune
> permission en cause, aucun déploiement ce jour-là, et deux retours en arrière sans effet.
>
> ### La correction (v130) : les quarts sont rangés EN TEXTE
> `_champsAdditifs()` écrit `window.txpQuartEnTexte(quart)` — le JSON du quart — au lieu de
> l'objet. **Une valeur texte ne compte que pour une entrée au lieu de vingt-cinq.**
> Mesuré : **12 quarts = 600 entrées avant, 24 après** (facteur 25). Pour 800 quarts : ~40 000 →
> ~1 600, avec de la marge jusqu'à ~20 000 quarts.
>
> **Ce qui ne change pas** : c'est toujours UN champ par identifiant de quart, donc l'écriture
> additive (`{merge:true}`) et les marqueurs de suppression `FieldValue.delete()` fonctionnent
> exactement pareil. Aucune protection n'est perdue.
>
> **La lecture accepte les deux formes.** `window.txpDecQ()` décode : texte → `JSON.parse`,
> objet → tel quel. **Les 12 points de lecture** du document passent par lui (transaction,
> `load()`, les deux `onSnapshot`, et les 4 lectures serveur des diagnostics). **Si un nouveau
> point de lecture est ajouté, il DOIT passer par `txpDecQ`.** La migration se fait toute seule :
> la première sauvegarde réussie réécrit tous les quarts en texte.
>
> **Surveillance ajoutée** : `estimerEntreesIndex()` compte les champs élémentaires (2 entrées
> chacun) et le rapport affiche, pour chaque document, « poids · entrées d'index ». Le verdict
> alerte dès 80 % des 40 000. **Toute mesure de document doit désormais regarder les DEUX
> limites.** À surveiller ensuite : `C.treso` et `C.payes` dans la config, qui grossissent sans
> fin et comptent, eux, en objets.
>
> 18 contrôles sur la forme texte (`test-quarts-en-texte.js`), dont la synchronisation entre deux
> appareils, une suppression, la relecture d'un document à l'ancienne forme et sa réécriture.
>
> ### POURQUOI LA MIGRATION NE SE FAIT PAS TOUTE SEULE (v131)
> Après la v130, Amin a répondu « rien monté ». **L'écriture normale fusionne** (`{merge:true}`) :
> les quarts que l'appareil n'a pas en mémoire **restent dans le cloud sous leur ancienne forme**,
> à ~50 entrées d'index chacun. Tant qu'il en reste assez, le document dépasse toujours les
> 40 000 entrées et toute écriture est refusée. **Le serpent se mord la queue : il faut écrire
> pour réparer, et on ne peut pas écrire.**
>
> `reparerDocumentQuarts()` casse le cercle : lecture **serveur** du document, réunion avec le
> local par `txpMerge` (tombstones respectés), puis **réécriture COMPLÈTE en texte avec un `set`
> sans merge**. Le remplacement total est volontaire et c'est le seul moyen de faire disparaître
> les anciens champs. Il est sûr ici parce que la lecture serveur vient d'avoir lieu et que,
> le document étant bloqué, aucun autre appareil ne peut écrire. **Ne pas généraliser ce `set`
> complet au chemin d'écriture normal** — c'est exactement le bug que la v123 a corrigé.
>
> Le bouton est dans le bandeau d'alerte (patron seulement) ET dans ⚙️ Réglages → 🧰 Outils
> professionnels, à côté de « 🔬 Tester les 5 documents », pour rester accessible une fois le
> bandeau disparu.
>
> 20 contrôles (`test-reparer-quarts.js`) sur le cas réel — cloud à l'ancienne forme dont
> l'appareil n'a qu'une partie, écoute coupée : 1 000 entrées → 44, les 19 quarts du cloud et les
> 3 de l'appareil réunis, le quart supprimé non ressuscité, et l'envoi normal qui repart.
>
> ### « 197 appareils connectés pour 10 chauffeurs » — Amin, 29 octobre
> Ce ne sont pas d'autres propriétaires : `members.uids` accumule une entrée par **identité
> anonyme Firebase**, et chaque « 🔄 Forcer la mise à jour » ou vidage de cache en crée une
> nouvelle. Dix chauffeurs sur quatre mois suffisent. Le dire avant d'agir : Amin a demandé de
> « fermer les sessions des autres propriétaires », ce qui part d'une lecture fausse du compteur.
>
> **NE PAS PURGER `members.uids` à l'aveugle.** Retirer un uid coupe immédiatement l'appareil
> correspondant — exactement le symptôme qu'on cherche à guérir. Tant que la flotte n'est pas
> verrouillée, un appareil purgé se réinscrit seul à la réouverture, donc la purge ne gagne rien ;
> si elle est verrouillée, la purge est **définitive**. Une purge n'est justifiable qu'après avoir
> établi la cause, et seulement sur les uid absents de `members.etats` depuis des semaines
> (la balise v125 donne la « dernière nouvelle » de chaque appareil).
>
> **Les sections ci-dessous (v116 à v126) sont de nouveau à jour** : tout ce qu'elles décrivent
> est dans `index.html`. Repères git : v115 = `3598c7d`, v116 = `51784ef`, v126 = `f6763ef`.
>
> Vérifié à chaque bascule, aller comme retour : le calcul de la paie est **identique** (taximètre
> net 329,75 · salaire 162,00 · part patron 197,85 · essence avancée 40,00 sur le quart de
> référence). Aucune donnée Firestore n'a été touchée par ces allers-retours.
>
> **269 contrôles verts sur la v127** : noyau 28, bandeau de suspension 31, renvoi et balises 22,
> diagnostic de sync 41, semaines voisines 16, manquants automatiques 15, « payer quand je veux »
> 30, paye robuste 29, rattrapage 20, mémoire pleine 19, plusieurs chauffeurs 18.
>
> **Pour payer tant que la cause n'est pas réglée** : les quarts manquants ne sont pas dans le
> cloud, donc aucune récupération ne peut les ramener — les chiffres n'existent que sur les
> téléphones des chauffeurs. Une page de calcul autonome a été remise à Amin le 28 octobre
> (<https://claude.ai/artifact/14nNzEXG1oGSdf623dsWxz>, privée) : il y recopie le compteur, les
> courses et les pourboires affichés par chaque chauffeur, et elle applique les mêmes formules
> que l'app (redevance CTQ retirée d'abord, puis le pourcentage, pourboires par-dessus, essence
> avancée remboursée à part). Vérifiée sur le quart de référence. Elle ne touche ni à `C`, `S`,
> `EX` ni à Firestore : la semaine devra être cochée « payée » dans l'app une fois les quarts
> remontés, sinon la trésorerie restera fausse.

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

## L'app signale elle-même les quarts manquants (v124)
- `verifierManquantsEnFond()` part à l'ouverture de la page **Paye** (patron seulement, au plus une fois par minute) : lecture serveur du document `shifts`, comparaison avec `S`, tombstones exclus. S'il manque quelque chose, `quartsManquantsBanniereHTML()` l'affiche EN HAUT de la page avec le détail par chauffeur et un bouton « ⬇️ Récupérer ces N quarts maintenant ».
- **Pourquoi là et pas dans les Réglages** : le patron ne doit pas avoir à lancer un diagnostic pour découvrir qu'il lui manque des quarts. Mesuré chez lui : un chauffeur avait 6 quarts et 1 991,05 $ au compteur, le patron n'en voyait qu'un à 75,20 $ / 7 courses.
- **Récupérer ne sert à rien si ça ne tient pas** : après `lSave()`, si `_stockagePlein` est vrai, la récupération le DIT et renvoie vers l'archivage au lieu d'annoncer un succès qui disparaîtra au prochain rechargement.

## Localiser le téléphone qui n'envoie pas (v125)
- **Mesuré** : diagnostic du patron, semaine du 19-25 → Lamine **1 / 1**. Le cloud ne contenait qu'UN de ses quarts alors que son téléphone en affichait six avec « Sync ✓ ». Le blocage était à la MONTÉE, pas à la descente. Toujours lire la colonne « ici / dans le cloud » avant de conclure.
- **`renvoyerMesQuarts()`** (bouton sur le tableau de bord du chauffeur) : force une écriture complète, puis RELIT le cloud pour compter ce qui y est vraiment. Jamais d'annonce de succès sans vérification — c'est précisément ce mensonge qui a coûté une semaine. En cas d'échec, il affiche le code du serveur et « montre ce message à ton patron ».
- **`baliseEtat()`** : chaque appareil dépose sa fiche (nom, version, nb de quarts, envoi en attente, mémoire pleine, dernière erreur, horodatage) **dans `members.etats[uid]`**, écriture additive. Le patron la voit dans son diagnostic sous « Téléphones de l'équipe », avec « dernière nouvelle ». Un appareil qui n'arrive plus à écrire ne dépose rien : **son silence est le signal**.
- **POURQUOI DANS `members` ET PAS DANS UN DOCUMENT À PART** : `firestore.rules` n'autorise que `config`, `shifts`, `expenses`, `audit`, `deleted` (`isDataDoc()`) plus `members`. Un nouveau document serait refusé, et le patron ne peut pas republier les règles tant que la console Firebase lui est fermée (vérification en deux étapes jamais terminée). **Ne jamais ajouter un document Firestore sans vérifier `isDataDoc()` d'abord.**

## « Pourquoi ça a arrêté d'un seul coup ? » (v126)
- L'app a tourné quatre mois sans incident. Ce qui a changé la semaine du 19-25, ce sont **les mises à jour v116→v119**. Quand le patron dit « avant ça marchait », partir de là et non d'une cause extérieure.
- **Un quart « manquant » peut être simplement MAL RANGÉ.** La v119 a corrigé la date proposée par le formulaire ; un téléphone pas encore à jour inscrit toujours le LENDEMAIN passé 20 h, et les quarts de dimanche soir atterrissent dans la semaine suivante. Le diagnostic affiche donc les **cinq semaines autour** de celle demandée (`out.semaines`), avec « ici / dans le cloud » : un écart qui se reporte sur la semaine voisine n'est pas une perte. Corriger ensuite via `corrigerDatesDecalees()`.
- **`forceMiseAJour()` attend la fin des envois** (jusqu'à 8 s) et demande confirmation s'il reste quelque chose en attente. Il rechargeait 300 ms après l'appui, ce qui violait la règle « jamais de `location.reload()` avant la fin des écritures Firebase » inscrite plus haut dans ce fichier — et ce bouton est pressé par les chauffeurs après chaque mise à jour.

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
