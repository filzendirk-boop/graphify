# K9 Global Solution — Confirmation de périmètre et points à trancher

> **Statut : Phase 0 — documents de conception. Aucun code métier produit.**
> Conformément à la section 1 du brief, ce dossier contient l'architecture, le plan de
> sécurité et le plan de livraison. Le code de production attend une validation explicite.

## Index du dossier

| Fichier | Contenu | Répond à |
|---|---|---|
| `00-perimetre-et-ambiguites.md` | Ce document : compréhension du périmètre, ambiguïtés bloquantes et non-bloquantes | §6.1 |
| `01-architecture.md` | Stack, diagrammes, découpage par domaine, isolation des trois volets | §6.2 |
| `02-modele-de-donnees.md` | Schéma complet, DDL, index géospatiaux, rétention | §6.3 |
| `03-plan-securite-conformite.md` | Plan de sécurité et de conformité RGPD/DSA | §3 |
| `04-backlog-securite.md` | Checklist de sécurité sous forme de tickets actionnables | §6.4 |
| `05-plan-de-livraison.md` | Phases, critères de sortie, portes de validation | §5 |
| `06-invariants-non-negociables.md` | Règles que je ne contournerai pas, y compris sur demande | §6.6 |

---

## 1. Compréhension du périmètre

**Produit.** Une application mobile Flutter (iOS + Android) tout-en-un pour propriétaires
de chiens en Grande Région (LU/BE/FR/DE), multilingue FR/DE/EN/LU, regroupant onze
modules fonctionnels autour d'un socle commun (compte, profil chien, géolocalisation,
messagerie, notifications, paiement).

**Le point structurant du produit n'est pas fonctionnel, il est architectural.** Les trois
volets de mise en relation partagent un profil utilisateur et un profil chien, mais doivent
être **étanches** entre eux : ni code de matching mutualisé, ni tables de matching
mutualisées, ni messagerie mutualisée, ni possibilité de requête croisée côté backend.
C'est cette contrainte qui pilote la conception de la base de données et du découpage
des services, pas l'inverse. Tout le document `01-architecture.md` en découle.

**Ce que j'ai retenu comme non-négociable** (détaillé dans `06-invariants-non-negociables.md`) :

1. Le Volet 1 (Rencontre) ne s'active qu'après vérification d'identité/âge par prestataire
   tiers, modération photo automatique + humaine, isolation totale des données, et audit
   externe. Ces quatre conditions sont cumulatives, pas alternatives.
2. Un compte présent uniquement dans les Volets 2/3 n'apparaît jamais comme suggestion
   dans le Volet 1, et réciproquement. L'opt-in est explicite, par volet, jamais par défaut.
3. Hébergement UE, pas de transfert hors UE sans encadrement contractuel documenté.
4. Aucune donnée de carte bancaire ne transite par notre infrastructure.
5. Je ne proposerai jamais de contourner la section 3 pour gagner du temps ; si une demande
   ultérieure va dans ce sens, je le signalerai explicitement au lieu de l'exécuter.

**Interprétation que je fais de « isolation stricte ».** Le brief demande qu'« aucune requête
croisée ne soit possible côté backend ». Une simple séparation par tables ou par schémas
dans une même base ne satisfait pas cette exigence : il suffit qu'un rôle applicatif
dispose des deux droits pour qu'une jointure redevienne possible, volontairement ou par
accident. Je propose donc une séparation par **bases de données distinctes, avec rôles,
pools de connexion et clés de chiffrement distincts, et sans extension de fédération
(`postgres_fdw`, `dblink`) installée** — la jointure devient alors impossible au niveau du
moteur, pas seulement interdite par convention. Le Volet 1 va plus loin : instance
managée séparée, sauvegardes séparées, accès humain séparé. Détail en `01-architecture.md` §4.

---

## 2. Ambiguïtés à trancher — bloquantes

Ces trois points changent la nature du travail. Je n'attends pas de réponse pour avoir
produit les documents de conception, mais il en faut une avant d'écrire du code.

**État au 8 août 2026 : B2 est tranché (18 ans partout). B1 et B3 restent ouverts.**

### B1 — Le dépôt de travail actuel n'est pas celui du projet

Le dépôt dans lequel je travaille (`filzendirk-boop/graphify`) contient **Graphify**, une
application web de visualisation de données en React/Redux/D3 issue d'un cursus Fullstack
Academy (dernier commit fonctionnel : dépendances de 2018, Travis CI, Sequelize 4, React 16).
Il n'a aucun rapport avec K9 Global Solution.

Je n'ai pas créé de squelette de code applicatif ici, parce qu'installer un backend NestJS
et une app Flutter dans ce dépôt produirait un mélange durablement confus, et parce que la
Phase 0 du plan de sécurité (secrets, environnements, CI/CD, branches protégées) doit être
posée sur un dépôt neuf pour être auditable. Les documents de conception sont en revanche
livrés ici, faute d'autre emplacement disponible dans cette session.

**Décision attendue :** créer un dépôt neuf `k9-global-solution` (recommandé — mono-repo
`apps/mobile` + `apps/api` + `infra`), ou confirmer que K9 doit vivre dans ce dépôt-ci.
Je n'ai accès qu'à `filzendirk-boop/graphify` dans cette session ; l'ajout d'un autre dépôt
doit être autorisé côté GitHub.

### B2 — Âge minimum général de l'application — ✅ **TRANCHÉ : 18 ans partout**

> **Décision du porteur de projet, 8 août 2026 : 18 ans pour l'ensemble de l'application,
> sans exception.** Ce point n'est plus ouvert ; il est repris comme invariant I13 dans
> `06-invariants-non-negociables.md`.

Rappel du raisonnement, conservé pour la traçabilité vis-à-vis d'un auditeur ou de la CNPD.
Les quatre juridictions cibles ne fixent pas le même âge de consentement numérique
(art. 8 RGPD, marge nationale 13–16 ans) : Luxembourg 16 ans, Allemagne 16 ans,
France 15 ans, Belgique 13 ans. Un âge minimum inférieur à 18 ans aurait imposé un
mécanisme de consentement parental différencié par pays, une modération renforcée des
conversations impliquant un mineur, et un cloisonnement mineurs/majeurs dans les Volets 2
et 3.

**Ce que la décision retire du périmètre :** les quatre régimes de consentement parental
(ticket D02, désormais sans objet), le cloisonnement mineurs/majeurs des Volets 2 et 3
(ticket D03, sans objet), et le risque de contact adulte/mineur dans les volets sociaux —
qui était le risque réputationnel majeur du produit.

**Ce que la décision ne retire pas, et c'est le point à ne pas confondre :** l'âge minimum
est désormais le même partout, mais le **niveau d'assurance sur cet âge reste différencié**.
Volets 2 et 3 : 18 ans déclaratif. Volet 1 : 18 ans **vérifié par prestataire tiers**,
condition bloquante inchangée. Un âge minimum uniforme ne dispense en rien de la
vérification d'identité du module Rencontre — la section 3.1 du brief reste applicable
intégralement.

**Coût accepté :** perte du segment 16–18 ans pour les balades. Marginal sur un marché où
le titulaire du chien est très majoritairement majeur, et réversible : ce segment s'ouvrira
plus tard, si souhaité, par un sous-profil « accompagné » — bien plus facile à ajouter qu'à
retirer.

### B3 — Modèle de monétisation et statut vis-à-vis des paiements

Le brief impose un PSP certifié PCI-DSS (§3.4) mais ne dit pas **ce qui est vendu**. Or les
implications réglementaires diffèrent radicalement :

- **Abonnement premium** (fonctionnalités de l'app) → Apple/Google imposent leurs achats
  in-app avec commission 15–30 % ; Stripe/Viva ne peuvent pas être utilisés pour du contenu
  numérique consommé dans l'app. Ajoute l'obligation allemande de bouton de résiliation
  (§ 312k BGB) et le droit de rétractation européen.
- **Réservation d'hébergement / location de véhicule avec encaissement pour un tiers**
  → nous devenons place de marché : encaissement pour compte de tiers, ce qui suppose soit
  un PSP avec offre marketplace (Stripe Connect, Adyen for Platforms), soit un statut
  d'agent de paiement. Ajoute la directive DAC7 (déclaration des revenus des vendeurs) et
  les obligations d'information du règlement P2B.
- **Mise en relation sans encaissement** (l'app renvoie vers le site du loueur) → aucune de
  ces obligations. C'est de très loin le plus simple pour un MVP.

**Ma recommandation :** Phases 1 à 2a **gratuites et sans aucun flux de paiement**, réservation
en Phase 3 d'abord en simple mise en relation sans encaissement, et abonnement premium
seulement quand il y a une base d'utilisateurs à convertir. Cela retire entièrement le
paiement du chemin critique du MVP. Viva.com reste pertinent pour les ventes de kits de
premiers secours en circuit B2C classique, hors app.

---

## 3. Ambiguïtés non bloquantes — décisions à prendre en cours de route

| # | Sujet | Pourquoi ça compte | Ma recommandation par défaut |
|---|---|---|---|
| N1 | Taille et compétences de l'équipe | Détermine monolithe modulaire vs microservices, Kubernetes vs conteneurs managés | Supposé 1–3 développeurs : monolithe modulaire NestJS + conteneurs managés, **pas** de Kubernetes |
| N2 | Budget vérification d'identité | 1–3 € par vérification, non récupérable si le Volet 1 est gratuit | Volet 1 payant ou plafonné en volume ; à arbitrer avant Phase 2b |
| N3 | Origine des données de lieux | OpenStreetMap est sous licence ODbL : partage à l'identique de la base dérivée et attribution obligatoires. Mélanger OSM et contributions utilisateurs dans une même table peut contaminer l'ensemble | Séparer strictement `place_source = osm` et `place_source = community`, attribuer OSM, ne jamais fusionner les enregistrements |
| N4 | Contenu des premiers secours canins | Un guide de soins d'urgence engage une responsabilité ; selon la formulation, cela peut relever de l'exercice de la médecine vétérinaire | Contenu relu et signé par un vétérinaire, avertissement non contournable, jamais de posologie médicamenteuse, toujours « contactez un vétérinaire » en action primaire |
| N5 | Source des vétérinaires d'urgence | Pas de jeu de données européen unique ; les ordres nationaux (LU, BE, FR, DE) ont des annuaires aux conditions d'usage variables | Démarrer sur un référentiel constitué manuellement pour la Grande Région, vérifié par téléphone, plutôt qu'un scraping juridiquement fragile |
| N6 | Contenu d'éducation/dressage | Vidéos et fiches : production interne, licence, ou partenariat éducateur | À arbitrer en Phase 3 ; c'est un projet éditorial, pas technique |
| N7 | Nom et marque | « K9 Global Solution » doit être vérifié à l'EUIPO ; les noms des apps citées en §2.1 (Woog, DogMap, Dogo…) ne doivent apparaître ni dans le produit, ni dans le code, ni dans les métadonnées des stores | Recherche d'antériorité avant tout dépôt sur les stores |
| N8 | Front web | Change le choix de langage backend et le partage de types | Aucun front web au MVP ; NestJS garde l'option ouverte |
| N9 | Tracking GPS en arrière-plan | Localisation continue : justification obligatoire à la revue App Store, impact batterie, volume de données | Enregistrement de session explicitement démarré par l'utilisateur, jamais de suivi passif |
| N10 | Autorité de contrôle | Établissement au Luxembourg → CNPD autorité chef de file, guichet unique pour BE/FR/DE | Confirmer que l'entité juridique porteuse est bien la S.A.R.L.-S luxembourgeoise |

---

## 4. Un point que le brief ne mentionne pas et qui devrait y figurer : le DSA

L'application héberge du contenu généré par les utilisateurs (avis sur les lieux, photos,
profils, messages) et met en relation des personnes. À ce titre elle est un **service
d'hébergement** au sens du règlement européen 2022/2065 (DSA), applicable depuis
février 2024, indépendamment de sa taille. Les obligations qui s'appliquent dès le premier
utilisateur :

- **Mécanisme de notification et d'action** accessible, électronique, avec accusé de réception.
- **Exposé des motifs** (« statement of reasons ») transmis à l'utilisateur pour toute
  décision de modération : suppression de contenu, suspension, bannissement.
- **Système interne de traitement des réclamations** gratuit, permettant de contester une
  décision de modération pendant six mois.
- **Point de contact unique** publié, et représentant légal si l'établissement n'est pas dans l'UE.
- **Rapport de transparence** annuel sur la modération.
- **Interdiction des interfaces trompeuses** (dark patterns) — pertinent pour les futurs
  abonnements et pour l'opt-in par volet, qui doit rester réellement neutre.

Ce n'est pas une couche à ajouter après : l'exposé des motifs et le traitement des
réclamations supposent que chaque décision de modération soit tracée avec sa raison, sa
base et son auteur dès la première ligne de code de modération. C'est pris en compte dans
le modèle de données (`02-modele-de-donnees.md`, tables `moderation_*`) et dans le backlog
(épopée SEC-E, tickets DSA-01 à DSA-05).

---

## 5. Ce que je propose comme prochaine étape

1. Vous tranchez B1 (dépôt) et B3 (monétisation). ~~B2 (âge minimum)~~ : tranché, 18 ans partout.
2. Vous relisez `01-architecture.md` §4 (isolation des volets) — c'est la décision la plus
   coûteuse à revenir dessus plus tard, et celle sur laquelle j'aimerais un accord explicite.
3. Sur validation, j'initialise le dépôt Phase 0 : structure de dossiers, CI/CD, gestion des
   secrets, environnements, et le socle `User` + `DogProfile` — **sans une ligne de code de
   Volet 1**, et sans code de matching partagé entre volets.
