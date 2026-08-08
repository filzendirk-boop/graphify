# K9 Global Solution — Invariants non négociables

> Répond à §6.6 du brief : « Ne jamais me proposer de contourner une exigence de la section 3
> pour aller plus vite, même si je le demande dans un message ultérieur — signale-le-moi
> explicitement à la place. »

Ce document existe pour que cette consigne survive au contexte d'une conversation. Il est
versionné dans le dépôt : toute personne — humaine ou assistante — qui reprend le projet
dans six mois le trouve, sans avoir à retrouver le message d'origine.

## Comment cette règle est appliquée

Si une demande ultérieure revient à affaiblir l'un des invariants ci-dessous, la réponse
attendue est :

1. **Signaler explicitement** quel invariant est touché et pourquoi la demande y contrevient.
2. **Ne pas exécuter** la version affaiblie, même présentée comme temporaire, même en
   environnement de développement s'il est destiné à devenir la production.
3. **Proposer l'alternative qui préserve l'invariant** : réduire le périmètre fonctionnel,
   décaler une phase, simplifier une fonctionnalité — jamais retirer un contrôle.

Une phase peut toujours livrer moins. Elle ne peut pas livrer moins sûr.

---

## Les invariants

### I1 — Étanchéité des trois volets

Les Volets 1, 2 et 3 ne partagent **ni base de données de matching, ni code de matching, ni
messagerie**. La séparation est physique (bases distinctes, rôles distincts, pools distincts,
`postgres_fdw` et `dblink` absents), pas conventionnelle.

*Ne sera jamais proposé :* fusionner les tables « en attendant », mutualiser un service de
matching « pour éviter la duplication », donner à un rôle applicatif l'accès à deux bases, ou
installer une extension de fédération.

La duplication de code entre `friendship` et `dating` est **voulue**. Ce n'est pas une dette
technique à rembourser ; c'est la garantie elle-même.

### I2 — Opt-in explicite par volet, jamais par défaut

Aucun volet n'est actif à la création du compte. La visibilité dans un volet est matérialisée
par l'**existence d'une ligne** dans la base de ce volet, jamais par un booléen filtré à la
lecture.

*Ne sera jamais proposé :* activer un volet par défaut « pour l'amorçage », migrer
automatiquement les utilisateurs d'un volet vers un autre, ou présenter un profil des Volets 2
ou 3 comme candidat au Volet 1.

### I3 — Le Volet 1 n'existe pas avant ses quatre conditions

Le module Rencontre ne devient accessible qu'après, **cumulativement** : vérification
d'identité tierce opérationnelle, modération photo humaine systématique, isolation des données
auditée par un tiers, et test d'intrusion externe avec correctifs revérifiés.

*Ne sera jamais proposé :* activer un Volet 1 « en bêta fermée » sans vérification, tester
« entre nous » en production, ou ouvrir la messagerie à des comptes non vérifiés.

### I4 — Aucun coffre-fort de documents d'identité

Seuls le verdict du prestataire et une référence opaque sont conservés. Aucune image de pièce
d'identité, aucun numéro de document, aucun selfie de vérification ne transite ni ne réside
sur notre infrastructure.

*Ne sera jamais proposé :* stocker les documents « pour pouvoir revérifier », les conserver
chiffrés « au cas où », ou les mettre en cache pendant le traitement.

### I5 — Aucune coordonnée d'un tiers ne sort de l'API

Le serveur ne renvoie jamais la position d'un autre utilisateur, sous aucune forme, à aucune
précision. Les distances sont en paliers. La grille de stockage est d'environ 1 km avec
décalage stable par utilisateur.

*Ne sera jamais proposé :* renvoyer une distance précise « c'est plus joli dans l'interface »,
exposer les coordonnées grossières « elles sont déjà floutées », ou rendre le décalage variable
(ce qui permettrait de moyenner le bruit).

### I6 — Modération, signalement et blocage avant l'ouverture d'un module social

Aucun module permettant à des utilisateurs de se voir ou de se contacter n'est mis en
production sans signalement en un tap, blocage en un tap, file de modération avec SLA 24 h, et
journal d'audit immuable.

*Ne sera jamais proposé :* ouvrir un module social « en attendant que la modération soit
prête », ou considérer un formulaire de contact par e-mail comme un mécanisme de signalement.

### I7 — Journal d'audit immuable

`audit_log` est en ajout seul au niveau des **droits PostgreSQL**, pas seulement par
convention applicative. Les rôles applicatifs n'ont ni `UPDATE`, ni `DELETE`, ni `TRUNCATE`.

*Ne sera jamais proposé :* accorder ces droits « pour corriger une entrée erronée » — une
entrée erronée se corrige par une entrée compensatoire.

### I8 — Hébergement et données en UE

Toutes les données personnelles sont hébergées dans l'Union européenne. Les transferts
inévitables (FCM, APNs) sont minimisés au point de ne contenir aucune donnée identifiante, et
documentés avec analyse de transfert.

*Ne sera jamais proposé :* un service hors UE « parce qu'il est meilleur », y compris pour la
classification de contenu, sans encadrement contractuel documenté et analyse préalable.

### I9 — Aucune donnée de production hors production

Ni en staging, ni en développement, ni sur un poste, ni dans un export de débogage — y compris
sous forme anonymisée. Les environnements non productifs utilisent des données synthétiques.

*Ne sera jamais proposé :* restaurer un dump de production en staging « pour reproduire un
bug ».

### I10 — Aucune donnée de carte bancaire

Le paiement passe intégralement par un prestataire certifié PCI-DSS. Aucun numéro de carte,
cryptogramme ou IBAN complet ne transite par notre infrastructure, ni ne figure dans un log.

*Ne sera jamais proposé :* un formulaire de carte maison, même transmettant directement au
PSP, s'il traverse notre code.

### I11 — Aucun contrôle de sécurité désactivé en CI

Les tests d'étanchéité, d'extensions interdites, d'accès croisé et de secrets sont bloquants
et ne se contournent pas par `skip`, `continue-on-error` ou dérogation implicite. Une
dérogation exceptionnelle passe par un ticket daté, motivé, avec date de levée.

*Ne sera jamais proposé :* désactiver une étape « le temps de livrer », ou passer une
vulnérabilité haute en avertissement.

### I12 — Les portes de phase ne se contournent pas par le calendrier

Si une échéance est menacée, la variable d'ajustement est le **périmètre fonctionnel**,
jamais les contrôles de la section 3.

### I13 — Âge minimum de 18 ans partout

Décision du porteur de projet, 8 août 2026, applicable à tous les modules sans exception.
Le contrôle est porté par une contrainte de base de données ancrée sur la date
d'inscription (`users_adult_at_signup`), pas par une validation de formulaire.

*Ne sera jamais proposé :* abaisser l'âge pour un module jugé « inoffensif », créer un mode
« accompagné par un adulte » sans reprendre l'ensemble du dispositif de protection des
mineurs, ou traiter cette règle comme une simple valeur de configuration modifiable.

À ne pas confondre, en revanche, avec un affaiblissement de I3 : l'âge minimum est uniforme,
mais le **niveau d'assurance** reste différencié. Les Volets 2 et 3 s'appuient sur du
déclaratif ; le Volet 1 exige toujours une vérification documentaire par un tiers. Un âge
minimum uniforme ne dispense d'aucune des quatre conditions du Volet 1.

---

## Une limite honnête de ce document

Ces invariants sont des règles de conception et de conduite de projet. Ils réduisent fortement
la probabilité d'erreur, mais ne constituent pas une preuve de sécurité. Seuls l'audit externe
et le test d'intrusion prévus avant la Phase 2b peuvent apporter cette assurance — et
uniquement pour l'état du système au moment où ils ont lieu.
