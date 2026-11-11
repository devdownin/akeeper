# Audit applicatif aKeeper — optimisation et compléments fonctionnels

- **Date** : 11/11/2026
- **Périmètre** : le code de l'application, qui vit dans [`devdownin/akeeper-1`](https://github.com/devdownin/akeeper-1). Ce dépôt-ci ne contient que de la documentation.
- **Révision auditée** : `feat/ui-p2-polish` (`03c0bf4`), la plus avancée. Elle contient aussi P0, P1 et les PR #11 et #12. Les constats valent aussi pour `main` (`57dfef0`).
- **Méthode** : revue statique du code Java (`app`, `vpn`, `rules`, `engine`) et Rust (`akeeper-engine`, `akeeper-proto`, `akeeper-ffi`), de la CI et de la documentation (`CLAUDE.md`, `docs/`). Les tests n'ont pas été exécutés dans cet environnement.
- **Priorités** : selon la matrice de `AUDIT_UI_UX.md` §10 (P1 bloquant → P4 amélioration).

## Synthèse

L'architecture est saine. Le moteur Rust est borné, sans `unsafe` hors FFI, et fuzzé. Les bundles sont signés en Ed25519 avec anti-rollback, le stockage est chiffré et les tests JVM et Rust sont nombreux. Les principaux gains possibles :

1. **Sécurité de la pause** : elle peut durer bien plus que les 15 min affichées (P1).
2. **Batterie** : trois réveils CPU par seconde de façon permanente (P2).
3. **Latence** : appels Binder synchrones sur le thread réseau unique, pas de cache DNS ni de bascule entre résolveurs (P2).
4. **Journal** : la recherche et le filtre « Autorisés » ne portent que sur les 500 derniers événements (P2).

## 1. Anomalies et risques

| # | Prio | Constat | Emplacement | Correction proposée |
|---|---|---|---|---|
| 1 | 🚨 P1 | **La pause peut dépasser 15 min.** Le moteur mesure la durée avec `Instant`, soit `CLOCK_MONOTONIC`, qui s'arrête pendant la veille profonde. Le délai du service (`Handler.postDelayed`, `uptimeMillis`) s'arrête aussi. Si le téléphone dort, le blocage reste suspendu au-delà de l'heure « reprise à HH:mm » affichée. Côté UI, `paused()` vaut seulement `pausedUntil > 0`, sans comparaison à l'heure courante. | `akeeper-engine/src/lib.rs:312`, `ShieldVpnService.java:108`, `ShieldStatus.java:38` | Moteur : échéance sur `CLOCK_BOOTTIME` (`libc::clock_gettime`, `libc` est déjà une dépendance), revérifiée à chaque réveil de `poll`. Service : `AlarmManager.set(ELAPSED_REALTIME_WAKEUP)` inexact, qui ne demande aucune permission. UI : `paused()` doit aussi vérifier `pausedUntil > now`. Ajouter un test avec une horloge injectée. |
| 2 | ⚠️ P2 | **Recherche et filtre « Autorisés » incomplets.** La requête SQL est limitée à 500 lignes, puis le filtre `!blocked` et la recherche texte s'appliquent en Java. Un domaine plus ancien est introuvable, et l'onglet « Autorisés » est vide si les 500 derniers événements sont des blocages. | `ActivityLog.java:107-119`, `ActivityScreen.java:136` | Faire le filtrage en SQL : `blocked = 0` et `name LIKE ? ESCAPE '\'`. Ajouter un index `(blocked, ts)`. |
| 3 | ⚠️ P2 | **Réinstallation d'une app bloquée.** Les UID ne sont recalculés qu'au démarrage du service et quand les règles changent. Une app bloquée qui est désinstallée puis réinstallée (nouvel UID) échappe au pare-feu jusqu'au prochain redémarrage. | `ShieldVpnService.java:228` | Receiver dynamique `PACKAGE_ADDED` / `PACKAGE_REPLACED` (`RECEIVER_NOT_EXPORTED`) qui relance `AppRules.uids()`. |
| 4 | ⚠️ P2 | **Pas de bascule entre résolveurs DNS.** Les serveurs amont sont choisis à tour de rôle, sans nouvelle tentative. Si l'un d'eux est muet, une requête sur N attend 5 s puis échoue. | `dns.rs:18`, `dns.rs:152-170` | À l'expiration, renvoyer la requête au serveur suivant (1 essai), et suivre l'état de santé de chaque serveur : ignorer temporairement un serveur après K échecs. |
| 5 | 💡 P3 | **DNS/TCP impossible.** Une réponse UDP tronquée (TC) échoue, ce qui casse les réponses TXT ou DNSSEC volumineuses. | `lib.rs` (RST vers le résolveur) | Petit relais DNS/TCP vers l'amont, ou augmenter l'EDNS0 côté amont. |
| 6 | 💡 P3 | **Le cache des journaux reste actif après une erreur Keystore.** `ActivityRecorder` retente `ActivityLog.get()` chaque seconde, et le journal système est inondé de warnings. | `ActivityRecorder.java:54` | Attente progressive (1 s → 5 min), puis abandon jusqu'au prochain changement de préférence. |
| 7 | 💡 P3 | **Pas d'épinglage TLS du serveur de règles**, ce qui est documenté. L'intégrité reste assurée par Ed25519, mais la disponibilité dépend de la chaîne de confiance système. | `network_security_config.xml` | Ajouter un `<pin-set>` avec expiration dès que l'URL Pages est fixée. |
| 8 | 💡 P3 | **Branches divergentes.** `main` de `akeeper-1` date du 24/10. P0, P1, P2 et les PR #11 et #12 attendent dans des branches empilées. | dépôt | Fusionner dans l'ordre P0 → P1 → P2, puis supprimer les branches. Ce dépôt `akeeper` pourrait aussi pointer vers `akeeper-1` dans son README. |

## 2. Optimisations (performance et batterie)

| # | Prio | Constat | Emplacement | Optimisation |
|---|---|---|---|---|
| O1 | ⚠️ P2 | **Le moteur se réveille chaque seconde.** `EXPIRY_TICK` = 1 s dès qu'un flux existe, ce qui est presque toujours le cas : une connexion push reste ouverte en permanence. | `lib.rs:67`, `lib.rs:266` | Calculer la prochaine échéance réelle (minimum des `deadline` DNS en attente et des expirations NAT) au lieu d'un tic fixe. |
| O2 | ⚠️ P2 | **Le journal est vidé chaque seconde, même désactivé** : appel JNI et réveil du thread. | `ActivityRecorder.java:48` | Ne planifier la tâche que si `logEnabled`, et l'annuler ou la replanifier depuis l'observateur de préférences. Avec l'écran éteint, une période de 5 à 10 s suffit, car le tampon garde 4096 événements. |
| O3 | ⚠️ P2 | **L'écran Activité relit la base toutes les 2 s**, même sans nouvel événement. | `ActivityScreen.java:39` | Notification depuis `ActivityRecorder` après une insertion non vide (observateur comme `ShieldStatus`). |
| O4 | ⚠️ P2 | **Binder synchrone sur le thread réseau.** `owner_uid` est appelé dans la boucle `mio` pour chaque requête DNS quand le journal est actif (il l'est par défaut) et pour chaque nouveau flux TCP ou UDP. Tout le trafic attend ces appels. La spec §1 prévoyait un thread dédié. | `dns.rs:121-125`, `tcp_stack.rs:177,190`, `udp_nat.rs:76` | DNS : mettre l'UID en cache par port source pendant quelques secondes, ou le résoudre de façon différée au moment de `drain_events`. Flux : thread d'attribution dédié, avec le SYN ou le premier datagramme retenu jusqu'à la réponse. |
| O5 | ⚠️ P2 | **Pas de cache DNS** : chaque requête part vers l'amont. | `dns.rs` | Cache borné (environ 2 048 entrées, LRU) qui respecte le TTL, avec des bornes de 30 s à 1 h. Il doit être vidé à chaque `SetFilter` et ne servir que les réponses autorisées. Gains : latence, batterie, moins de requêtes visibles par le FAI. |
| O6 | 💡 P3 | **Recharge complète des règles à tout changement de préférence**, y compris l'interrupteur du journal, la durée de conservation ou le réseau local. Le moteur reconstruit alors son filtre et sa liste personnelle. | `ShieldVpnService.java:163-165` | Ne recharger que si `categoryMask` change. |
| O7 | 💡 P3 | **Une allocation par recherche dans les listes.** `Blocklist::blocks` alloue une `String` à chaque appel, et `Filter::verdict` interroge jusqu'à 6 listes. | `blocklist.rs:65`, `filter.rs:207` | Inverser le nom une seule fois et passer les préfixes à chaque liste, ou parcourir le FST nœud par nœud (`fst::raw`) en testant chaque frontière de label. |
| O8 | 💡 P3 | **R8 et la réduction des ressources ne sont pas activés en release** : aucun `isMinifyEnabled` ni `isShrinkResources` dans `app/` ou `build-logic/`, alors que des règles keep existent déjà (`consumer-rules.pro`). | `app/build.gradle.kts` | `isMinifyEnabled = true` et `isShrinkResources = true`. Vérifier ensuite le JNI (`NativeEngine.Host`) et Tink. |

## 3. Compléments fonctionnels proposés

Classés par valeur pour l'utilisateur et par effort.

| Prio | Fonction | Intérêt | Base existante |
|---|---|---|---|
| ⚠️ P2 | **Action « Pause » dans la notification et tuile des réglages rapides** (`TileService`) | Écart déjà documenté. C'est l'accès le plus direct à la protection. | `ShieldVpnService.pause/resume` |
| ⚠️ P2 | **DNS amont chiffré** (DoT ou DoH vers un résolveur choisi : Quad9, Cloudflare, personnalisé) | Aujourd'hui, les requêtes autorisées partent **en clair** vers le résolveur du FAI. C'est le principal manque en confidentialité. | `UpstreamDns`, sockets protégés, `rustls` côté moteur |
| 💡 P3 | **Pare-feu par app et par type de réseau** (bloquer en données mobiles seulement, ou en Wi-Fi seulement) | Fonction phare des pare-feu sans root ; économie de forfait. | `NetworkStateMonitor` (branche P2), `setBlockedUids` |
| 💡 P3 | **Durées de pause au choix** (5, 15 ou 60 min) | Souplesse sans risque, puisque le moteur plafonne à 1 h. | `MAX_PAUSE` |
| 💡 P3 | **Exceptions par app** (autoriser un domaine pour une seule app) | Réduit les faux positifs sans affaiblir le reste. | Clé `(uid, domaine)` dans `Filter` |
| 💡 P3 | **Export et import chiffrés** des listes personnelles et des règles par app (SAF, mot de passe) | La sauvegarde Android est volontairement désactivée : sans export, l'utilisateur perd sa configuration en changeant de téléphone. | `UserLists`, `AppRules`, Tink |
| 💡 P3 | **Niveau publicité « standard » ou « renforcé »** (HaGeZi LIGHT ou Pro) | Écart déjà noté dans R2. | Pipeline de bundles, catégories |
| 🎨 P4 | **Statistiques** : top des domaines et apps bloqués sur 24 h et 7 jours, graphique sur l'accueil | Montre la valeur de l'app. | `ActivityLog`, `SparklineView` (branche P1) |
| 🎨 P4 | **Filtre « Récentes »** dans Apps et **notification à la première connexion d'une nouvelle app** | Écart J3 ; donne la main sur une app tout juste installée. | Receiver de l'anomalie 3 |
| 🎨 P4 | **Relais ICMP (ping)** et **réassemblage des fragments IPv4** | Compatibilité réseau, limites J2 documentées. | `akeeper-proto::icmp` |

## 4. Qualité, tests et CI

- **Tests sur appareil** : aucun test instrumenté. Les runners `ubuntu-24.04` de GitHub ont KVM : un job émulateur (API 36) pourrait au moins vérifier `establish()`, le DNS, la pause et SQLCipher, qui ne sont aujourd'hui validés qu'à la main.
- **Mesures** : ajouter un Macrobenchmark (démarrage, défilement de A4) et un banc Rust `criterion` (`Filter::verdict`, `drain_tun`) pour chiffrer O1 à O7.
- **Sécurité de la chaîne** : `cargo-deny` et `verification-metadata.xml` arrivent avec la PR #12. Il faut les fusionner (point 8).
- **Fuzzing** : les 5 cibles tournent chaque nuit. Ajouter une cible `filter` (noms arbitraires contre `verdict`).

## 5. Plan d'action proposé

| Lot | Contenu | Effort estimé |
|---|---|---|
| **Sprint 1** | 1 (pause), 2 (journal SQL), 3 (réinstallation), O6, O8, fusion des branches (8) | ~3 j |
| **Sprint 2** | O1, O2, O3 (batterie), 4 (bascule DNS), O5 (cache DNS) | ~5 j |
| **Sprint 3** | O4 (attribution asynchrone), action Pause et tuile, durées de pause | ~5 j |
| **Ensuite** | DNS amont chiffré, pare-feu par réseau, export, exceptions par app, CI émulateur | à chiffrer |
