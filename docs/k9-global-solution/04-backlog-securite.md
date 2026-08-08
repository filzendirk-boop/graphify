# K9 Global Solution — Backlog de sécurité et conformité

> Répond à §6.4 du brief : la checklist de la section 3 sous forme de tickets actionnables.
> Format prêt à importer comme issues GitHub. Chaque ticket porte un critère d'acceptation
> **vérifiable** — « la sécurité est renforcée » n'en est pas un.

**Légende priorité :** P0 = bloquant pour la phase indiquée · P1 = à faire dans la phase ·
P2 = souhaitable · **[GATE]** = porte de validation, rien ne passe en production sans.

---

## Épopée SEC-A — Fondations d'infrastructure (Phase 0)

| ID | Titre | Prio | Critère d'acceptation |
|---|---|---|---|
| A01 | Créer le dépôt K9 avec branches protégées | P0 | `main` protégée : revue obligatoire d'un tiers, CI verte requise, push direct interdit y compris pour les administrateurs, commits signés |
| A02 | Provisionner l'infrastructure en Terraform, région UE | P0 | `terraform plan` reproductible, état distant chiffré et verrouillé, aucune ressource créée à la main ; les trois environnements sortent du même code |
| A03 | Créer les **trois bases séparées** avec rôles distincts | P0 **[GATE]** | `k9_core`, `k9_friendship`, `k9_dating` existent ; `app_core` ne peut pas se connecter à `k9_friendship` ni `k9_dating` — prouvé par un test qui tente la connexion et vérifie l'échec |
| A04 | Instance PostgreSQL dédiée pour `k9_dating` | P0 | Instance nº2 distincte, groupe de sécurité réseau distinct, sauvegardes séparées |
| A05 | Interdire `postgres_fdw` et `dblink` sur les trois bases | P0 **[GATE]** | Test d'intégration : `CREATE EXTENSION postgres_fdw` échoue avec les rôles applicatifs sur chaque base ; le test tourne à chaque CI |
| A06 | Gestionnaire de secrets et amorçage | P0 | Aucun secret dans le dépôt ; `gitleaks` en pré-commit et en CI ; rotation des identifiants de base automatisée et testée |
| A07 | Trois environnements isolés | P0 | dev/staging/production dans des projets réseau séparés ; aucun identifiant partagé ; production inaccessible depuis staging |
| A08 | Générateur de données synthétiques | P0 | Une commande peuple staging avec des utilisateurs, chiens, lieux et annonces plausibles ; **aucune donnée de production n'est jamais copiée**, y compris anonymisée |
| A09 | TLS 1.3 partout, y compris en interne | P0 | Test SSL Labs note A ; HSTS avec préchargement ; application→base et application→Redis en TLS vérifié |
| A10 | Chiffrement des sauvegardes + test de restauration | P0 | Restauration complète réussie sur environnement jetable, chronométrée, procédure écrite ; à rejouer chaque trimestre |
| A11 | Clés KMS distinctes par base | P1 | `k-core`, `k-friend`, `k-dating` ; une compromission de la sauvegarde `k9_core` ne déchiffre aucun message de volet — démontré |
| A12 | Journalisation centralisée en UE avec purge de PII | P1 | Aucun e-mail, jeton, coordonnée GPS ni corps de message dans les logs ; filtre de rédaction testé unitairement ; rétention 30 jours pour l'applicatif |

---

## Épopée SEC-B — Identité, authentification, comptes (Phase 0)

| ID | Titre | Prio | Critère d'acceptation |
|---|---|---|---|
| B01 | Déployer Keycloak en UE, durcir la configuration | P0 | Console d'administration non exposée publiquement ; MFA obligatoire pour les administrateurs ; comptes par défaut supprimés |
| B02 | Jetons d'accès courts et refresh rotatifs | P0 | Accès 10 min ; refresh 30 j lié à l'appareil ; **la réutilisation d'un refresh invalide la famille entière** et lève une alerte — test automatisé |
| B03 | Revendications minimales dans le JWT | P0 | Un JWT décodé ne contient ni e-mail, ni nom, ni date de naissance, ni position — test de contrat |
| B04 | Vérification d'e-mail obligatoire avant tout usage social | P0 | Un compte non vérifié ne peut ni publier, ni contacter, ni apparaître dans une découverte |
| B05 | MFA optionnelle utilisateur, obligatoire modération/support/admin | P0 | Un compte modérateur sans MFA ne peut pas se connecter au back-office |
| B06 | Gestion et révocation des sessions par l'utilisateur | P1 | Liste des appareils connectés dans l'app ; révocation effective en moins de 60 s (durée du jeton d'accès) |
| B07 | Limitation progressive des tentatives d'authentification | P1 | Délai croissant par compte et par IP ; jamais de blocage définitif exploitable en déni de service ciblé |
| B08 | Suppression du compte **depuis l'application** | P0 | Exigence Apple et Google ; déclenche la saga d'effacement de RGPD-04 |
| B09 | Verrouillage biométrique optionnel de l'app | P2 | Protège carnet de santé et messages en cas de vol du téléphone (menace M7) |

---

## Épopée SEC-C — Étanchéité des volets (Phases 0, 2a, 2b) — **cœur du dispositif**

| ID | Titre | Prio | Critère d'acceptation |
|---|---|---|---|
| C01 | Test d'architecture des frontières inter-modules | P0 **[GATE]** | `dependency-cruiser` casse le build si `friendship` importe `dating` ou l'inverse, directement ou transitivement ; test volontairement mis en échec une fois pour prouver qu'il détecte |
| C02 | Pools de connexion isolés par module | P0 **[GATE]** | Le module `dating` est le seul porteur des identifiants `app_dating` ; un autre module tentant d'y accéder échoue au démarrage |
| C03 | Règles Semgrep maison anti-fuite inter-volets | P0 | Règles écrites et actives : import croisé, renvoi de coordonnées d'un tiers, SQL concaténé, journalisation de champ sensible |
| C04 | Projection de profil par outbox | P0 | `ProfileUpdated` propage vers les bases de volet ; **aucune connexion inter-bases** ; la projection ne contient ni e-mail, ni date de naissance, ni santé — test de schéma |
| C05 | Opt-in par volet matérialisé par l'existence d'une ligne | P0 **[GATE]** | Désactiver le Volet 2 supprime la ligne `friendship_profiles` ; un compte sans opt-in n'est **retourné par aucune** requête de découverte — test avec compte témoin |
| C06 | Alerte sur connexion anormale au pool `app_dating` | P1 | Toute connexion depuis un service autre que `dating` déclenche une alerte d'astreinte |
| C07 | Duplication assumée et documentée du code de matching | P0 | ADR écrite expliquant pourquoi `friendship` et `dating` ne partagent aucun code de matching ; checklist de revue rappelant d'appliquer les correctifs des deux côtés |
| C08 | Test d'étanchéité de bout en bout | P0 **[GATE]** | Scénario automatisé : un compte opt-in Volet 2 uniquement n'apparaît jamais dans une découverte Volet 1, et réciproquement ; exécuté à chaque CI |

---

## Épopée SEC-D — Protection des mineurs et Volet 1 (Phase 2b, sauf D15/D16 en Phase 0)

> Contexte : l'âge minimum est fixé à **18 ans partout** (D01, clos). D15 et D16 relèvent
> donc de l'inscription générale et sont dus dès la Phase 0 ; le reste de l'épopée reste
> attaché à l'activation du Volet 1.

| ID | Titre | Prio | Critère d'acceptation |
|---|---|---|---|
| D01 | Trancher l'âge minimum général (question B2) | — | ✅ **CLOS le 8 août 2026 : 18 ans partout, sans exception.** Décision tracée en `00-perimetre-et-ambiguites.md` §B2 et en invariant I13 |
| ~~D02~~ | ~~Consentement parental différencié par pays~~ | — | ❌ **SANS OBJET** suite à D01 : plus aucun mineur, donc aucun consentement parental à recueillir. Quatre régimes nationaux retirés du périmètre |
| ~~D03~~ | ~~Cloisonnement mineurs/majeurs Volets 2 et 3~~ | — | ❌ **SANS OBJET** suite à D01 : la population est entièrement majeure |
| D15 | Contrôle de majorité à l'inscription | P0 | Date de naissance obligatoire, refus si < 18 ans à la date du jour ; contrainte en base et non seulement en formulaire ; le refus n'indique pas quel critère a échoué (évite d'apprendre à réessayer) |
| D16 | Empêcher la réinscription immédiate après refus pour minorité | P1 | Un refus pour âge ne doit pas se contourner en resoumettant une autre date trente secondes plus tard ; verrou par attestation d'appareil, sans conserver la date refusée |
| D04 | Intégrer le prestataire de vérification d'identité | P0 **[GATE]** | itsme et/ou Veriff en production ; **aucune image de document n'est stockée chez nous** — vérifié par revue du code et du bucket |
| D05 | Verrou d'intégrité `dating_profiles` ↔ `identity_checks` | P0 **[GATE]** | Le déclencheur PostgreSQL rejette l'insertion d'un profil sans vérification approuvée et majeure ; test d'intégration tentant le contournement |
| D06 | Modération photo humaine systématique avant publication Volet 1 | P0 **[GATE]** | Aucun `dating_profiles.is_active = true` sans `photos_reviewed_at` — contrainte `CHECK` en base |
| D07 | Classification d'âge apparent sur les photos de profil | P0 | Score calculé ; en dessous du seuil, mise en file humaine obligatoire ; taux de faux négatifs mesuré sur un jeu d'évaluation |
| D08 | Interdire toute messagerie non vérifiée dans le Volet 1 | P0 **[GATE]** | Un compte sans `identity_check` approuvé ne peut émettre aucun message — testé au niveau API et au niveau base |
| D09 | Signalement `minor_suspected` en priorité maximale | P0 | Suspension conservatoire immédiate du contenu, examen humain < 1 h, procédure écrite |
| D10 | Détection de motifs langagiers évoquant la minorité | P1 | Règles multilingues FR/DE/EN/LU ; déclenche une revue, jamais une sanction automatique |
| D11 | Procédure de traitement du contenu pédopornographique | P0 **[GATE]** | Procédure écrite, validée juridiquement, testée à blanc : conservation sous scellés, blocage, signalement aux autorités, **jamais de suppression avant instruction** |
| D12 | Protection et rotation des modérateurs | P1 | Plafond d'exposition quotidien, floutage par défaut, accompagnement psychologique contractualisé |
| D13 | **Pentest externe du Volet 1** | P0 **[GATE]** | Rapport d'un prestataire indépendant ; toutes les vulnérabilités critiques et hautes corrigées et **reteste** avant activation |
| D14 | Audit externe de l'isolation des données | P0 **[GATE]** | Un tiers confirme par écrit qu'aucune requête croisée n'est possible entre les trois bases |

---

## Épopée SEC-E — Modération et DSA (Phases 1 à 2b)

| ID | Titre | Prio | Critère d'acceptation |
|---|---|---|---|
| E01 | Signalement en un tap sur tout contenu de tiers | P0 | Accessible depuis chaque écran affichant du contenu d'autrui ; jamais plus d'un tap depuis le contenu |
| E02 | Blocage en un tap, immédiat, silencieux, bilatéral | P0 | La personne bloquée n'est pas notifiée et disparaît des deux côtés ; l'interface précise clairement la portée (par volet) |
| E03 | SLA de traitement 24 h avec priorisation | P0 | Champ `sla_due_at` alimenté, tableau de bord de suivi, alerte sur dépassement |
| E04 | Journal d'audit immuable, par volet | P0 **[GATE]** | `REVOKE UPDATE, DELETE` effectif sur `audit_log` ; chaînage par hachage vérifiable ; test tentant une modification et constatant l'échec |
| E05 | **DSA** — mécanisme de notification et d'action | P0 | Formulaire électronique, accusé de réception automatique, suivi de l'état par le signalant |
| E06 | **DSA** — exposé des motifs à chaque décision | P0 | Toute décision de modération génère un message contenant motif, base légale ou contractuelle, caractère automatisé ou non, et voies de recours |
| E07 | **DSA** — traitement interne des réclamations | P0 | Recours possible pendant 6 mois, gratuit, examiné par une personne (pas uniquement par un automate) |
| E08 | **DSA** — point de contact et mentions légales | P0 | Publiés dans l'app et sur le site ; Impressum conforme au droit allemand |
| E09 | **DSA** — rapport de transparence annuel | P1 | Données de modération agrégées exportables depuis `moderation_decisions` |
| E10 | Back-office de modération sans accès base | P0 | Les modérateurs n'ont aucun identifiant de base de données ; chaque consultation de contenu est journalisée avec l'identité du modérateur |
| E11 | Détection de coordonnées bancaires et sortie de plateforme | P1 | IBAN, numéros de carte, identifiants de messagerie tierce détectés dans les messages des Volets 1 et 2 (menace M3) |
| E12 | Pipeline média complet | P0 **[GATE]** | Antivirus, **suppression EXIF**, réencodage, classification ; contrainte `media_published_requires_strip` active ; aucun média publiable sans passage par la quarantaine |

---

## Épopée SEC-F — CI/CD et chaîne de construction (Phase 0)

| ID | Titre | Prio | Critère d'acceptation |
|---|---|---|---|
| F01 | Pipeline complet lint → build → déploiement | P0 | Étapes conformes à `01-architecture.md` §8 ; approbation manuelle obligatoire pour la production |
| F02 | SAST bloquant | P0 | Semgrep + CodeQL ; criticité haute casse le build ; dérogation uniquement par ticket tracé et daté |
| F03 | Analyse des dépendances et des images | P0 | Dependabot actif, Trivy sur l'image ; critique/haute exploitable bloquante |
| F04 | Règles Semgrep spécifiques au projet | P0 | Voir C03 ; couverture testée par des cas positifs et négatifs |
| F05 | Scan des secrets | P0 | gitleaks en pré-commit et en CI ; procédure de rotation déclenchée sur détection, pas seulement suppression du commit |
| F06 | Scan d'infrastructure as code | P1 | Checkov/tfsec sur Terraform, criticité haute bloquante |
| F07 | DAST sur staging | P1 | OWASP ZAP en mode base après chaque déploiement staging ; rapport archivé |
| F08 | Analyse des artefacts mobiles | P1 | MobSF sur APK et IPA ; absence de secret embarqué vérifiée |
| F09 | Images signées et provenance | P2 | Signature cosign, SBOM généré et conservé |
| F10 | Test automatisé d'accès croisé (BOLA/IDOR) | P0 | Un compte A tente d'accéder aux ressources d'un compte B sur chaque point d'entrée ; toute réussite casse le build |

---

## Épopée SEC-G — RGPD opérationnel (Phases 0 et 1)

| ID | Titre | Prio | Critère d'acceptation |
|---|---|---|---|
| G01 | Registre des traitements | P0 **[GATE]** | Document tenu à jour, une entrée par traitement, base légale et rétention explicites |
| G02 | AIPD / DPIA | P0 **[GATE]** avant Phase 2a | Analyse d'impact complète, mesures d'atténuation tracées ; à refaire avant la Phase 2b |
| G03 | Consentements granulaires et révocables | P0 | Quatre consentements distincts minimum ; refuser l'un ne dégrade pas les autres fonctions ; table en ajout seul avec preuve |
| G04 | Saga d'effacement multi-bases | P0 **[GATE]** | Accusé par base, entrée d'audit finale, alerte sur saga incomplète, confirmation à l'utilisateur sous 30 jours |
| G05 | Export de portabilité | P1 | Archive JSON + GPX, livrée par URL présignée courte, **sans données personnelles de tiers** |
| G06 | DPA signés avec tous les sous-traitants | P0 **[GATE]** | Aucun sous-traitant en production sans contrat signé ; liste tenue avec localisation des données |
| G07 | Minimisation des notifications push | P0 | Charge utile sans nom, sans extrait de message, sans position ; TIA documentée pour le transfert FCM/APNs |
| G08 | Confidentialité de la géolocalisation | P0 **[GATE]** | Distances en paliers uniquement ; grille ~1 km ; décalage **stable** par utilisateur ; **aucune coordonnée de tiers en réponse d'API** — test automatisé tentant la triangulation |
| G09 | Politique de confidentialité et CGU multilingues | P0 | FR/DE/EN/LU ; versionnées ; acceptation tracée par version ; nuances par juridiction validées par un conseil |
| G10 | Politique de rétention automatisée | P1 | Tâches de purge conformes au tableau `02-modele-de-donnees.md` §7 ; rôle `app_retention` distinct ; opérations journalisées |
| G11 | Manifeste de confidentialité et fiche Data Safety | P0 | Cohérents avec G01, sinon retrait du store |
| G12 | Désigner un référent conformité (ou DPO) | P1 | Nom, contact publié, avant l'activation du Volet 1 |

---

## Épopée SEC-H — Exploitation et réponse à incident (Phase 1)

| ID | Titre | Prio | Critère d'acceptation |
|---|---|---|---|
| H01 | Plan de réponse à incident écrit | P0 **[GATE]** | Rôles, escalade, classification en 4 niveaux, canaux ; testé à blanc au moins une fois |
| H02 | Modèle de notification CNPD sous 72 h | P0 | Formulaire pré-rempli, chemin de décision documenté, responsable nommé |
| H03 | Registre des violations | P0 | Tenu même pour les incidents non notifiables (art. 33.5) |
| H04 | Alertes de sécurité métier | P0 | Les dix signaux de `03-plan-securite-conformite.md` §7 instrumentés, avec seuils calibrés |
| H05 | Accès production restreint et tracé | P0 | MFA, justification écrite, session enregistrée, revue mensuelle ; **deux personnes requises pour `k9_dating`** |
| H06 | Exercice de crise semestriel | P1 | Au moins une restauration complète et une simulation de fuite Volet 1 par an |
| H07 | `security.txt` et divulgation coordonnée | P1 | Publié, adresse surveillée, engagement de non-poursuite pour les chercheurs de bonne foi |
| H08 | Revue OWASP Top 10 + Mobile Top 10 avant chaque mise en production | P0 | Checklist signée dans la PR de version ; refus de fusion sans signature |

---

## Récapitulatif des portes de validation

Aucune des phases suivantes ne démarre sans que les portes de la précédente soient franchies
et **documentées par écrit**.

| Porte | Tickets requis | Condition |
|---|---|---|
| **Sortie de Phase 0** | A01–A05, A09, A10, B01–B05, B08, C01–C05, C08, **D15**, F01–F05, F10, G01, G03, G06, G09, H01 | Le socle et l'étanchéité sont vérifiés automatiquement, et aucun compte de mineur ne peut être créé |
| **Sortie de Phase 1** | E01–E06, E10, E12, G04, G08, G11, H02–H05, H08 | Modération, DSA, effacement et géo-confidentialité opérationnels |
| **Entrée en Phase 2a** | G02 (AIPD), C06, C07 | Analyse d'impact validée avant tout matching |
| **Entrée en Phase 2b** | D04–D14 et D15–D16 en totalité, G02 refaite, H06 | **Vérification d'identité, isolation auditée par un tiers et pentest externe — cumulativement.** D01 est clos, D02 et D03 sont sans objet |

Ces portes correspondent à ce que la section 3 du brief pose comme non-négociable. Elles ne
se contournent pas par décision de calendrier ; le seul chemin est de réduire le périmètre
fonctionnel de la phase, jamais ses contrôles.
