# Triage SCA : tp-aks-backend

Date : 2026-09-30. Portée : lecture seule, aucune dépendance modifiée, aucun déploiement, aucune ressource Azure touchée.

## Chiffres et méthode

- **27 identifiants uniques** (GHSA) pour **32 lignes** (paquet, version, identifiant). L'écart vient de `jackson-databind` (coordonnées `com.fasterxml.jackson.core` 2.x et `tools.jackson.core` 3.x) : 5 identifiants sont partagés mais les deux versions se corrigent séparément.
- Sévérités : **4 critiques**, 8 hautes, 20 moyennes.
- Source : `osv-scanner` 2.6.0 relancé en local sur `pom.xml` (même version et même commande que `sca.yml`). Le log CI ne contient que la redirection vers le Job Summary, pas le tableau.
- Chaîne de dépendances : `deps.dev` (arbres des starters). Noms de propriétés de version vérifiés dans `spring-boot-dependencies` 3.5.16.
- Spring Boot 3.5.16 est la dernière 3.5.x publiée sur Maven Central : **pas de bump du parent possible**, les correctifs passent par des surcharges de propriétés dans `<properties>`.
- **Limite** : pas de JDK ici, donc rien n'a été compilé. Le classement en lots repose sur la sémantique des versions et sur la lecture du code, pas sur un build. `./mvnw dependency:tree` à lancer avant d'appliquer.

## Lecture des colonnes d'atteignabilité

- **Chargé en runtime ?** : le composant est-il dans le jar déployé (ou seulement au build/test).
- **Chemin vulnérable exercé ?** : le code applicatif (ou sa configuration) déclenche-t-il la fonction touchée par la faille : oui, non, inconnu.
- Aucune faille n'a un déclencheur confirmé. Les composants les plus exposés sont Tomcat et Jackson (sur le chemin de chaque requête entrante).

## Lots

### Lot 1 : bumps sûrs, un commit (18 lignes, dont les 4 critiques)

Patchs sur la même ligne, composants sans couplage avec le code applicatif. Surcharges à ajouter dans `<properties>` de `pom.xml` :

| Propriété | Actuelle | Cible | Efface |
|---|---|---|---|
| `tomcat.version` | 10.1.55 | 10.1.59 | 3 critiques Tomcat |
| `netty.version` | 4.1.135.Final | 4.1.137.Final | 1 critique (netty-handler) + 13 autres |
| `postgresql.version` | 42.7.11 | 42.7.12 | 1 haute |

### Lot 2 : bumps qui demandent un test de build (14 lignes)

À valider par `./mvnw verify` (tests inclus) puis un démarrage local contre Redis/Postgres du `docker-compose.yml`.

| Changement | Actuelle | Cible | Pourquoi tester |
|---|---|---|---|
| `jackson-bom.version` | 2.21.4 | 2.21.6 | Patch, mais la sérialisation du cache Redis (`CacheConfig`) en dépend. Efface les 7 lignes `com.fasterxml.jackson` |
| `log4j2.version` | 2.24.3 | 2.25.5 | Mineur (pont `log4j-to-slf4j`) |
| `commons-lang3.version` | 3.17.0 | 3.18.0 | Mineur, origine à confirmer par `dependency:tree` (probablement `swagger-core-jakarta` via springdoc) |
| `tools.jackson.core:jackson-databind` (épinglage via `dependencyManagement`) | 3.1.4 | 3.1.6 | Jackson 3 arrive uniquement par `spring-cloud-azure` 7.4.0 ; aucune propriété Boot 3.5 ne le gère |

Point d'attention hors CVE : `spring-cloud-azure` 7.4.0 (mergé par Dependabot) cible Spring Boot 4 (`deps.dev` montre `spring-boot-starter-logging` 4.1.0 dans son arbre) alors que le parent est Boot 3.5.16. Le CI (build et scan) ne prouve pas que le démarrage réel fonctionne avec cette combinaison. À vérifier avant tout déploiement.

### Lot 3 : migrations majeures à écarter

**Aucune faille ne l'exige** : chacune des 32 lignes a un correctif sur la même version majeure. Rien à migrer pour solder le SCA. À écarter explicitement (cités dans `.worker-out/etat.md` du TP) : Spring Boot 4, springdoc 3, Temurin 25. Justification : chantiers de migration sans lien avec les 27 identifiants, risque de rupture élevé pour un TP de deux jours.

## Recommandation : preuve de remédiation réelle

Lecture retenue : **une faille par catégorie de sévérité présente dans le dépôt** (critique, haute), choisie parmi les lots 1 et 2. Le lot 3 est écarté par définition et ne fournit aucune preuve.

1. **Critique : `tomcat-embed-core` 10.1.55 vers 10.1.59, GHSA-gcx9-497g-6cp6 / CVE-2026-65182** (contournement de security-constraint).
   - Composant le plus exposé de l'app (serveur HTTP), patch sans rupture, une seule ligne `tomcat.version`.
   - Le même bump efface GHSA-9xv2-5v5q-p794 et GHSA-h3x4-894j-xpx5. Le lot 1 complet efface les 4 critiques.
2. **Haute : `jackson-databind` 2.21.4 vers 2.21.6, GHSA-q4xh-88c3-wmh7 / CVE-2026-68497** (nombre non borné à l'analyse de `Duration`/`XMLGregorianCalendar`).
   - Jackson analyse chaque corps de requête ; patch sans rupture ; une propriété `jackson-bom.version` efface les 7 lignes 2.x.

Honnêteté sur la portée : dans les deux cas le composant est **exposé** mais le **chemin vulnérable n'est pas exercé** par le code (voir tableau). La preuve démontrable est « composant exposé corrigé, scan OSV vert sur ces identifiants », pas « exploit bloqué ».

## Détail des findings

| Paquet | Version actuelle | Version corrigée (cette faille) | Cible du lot (toutes failles) | Sévérité (CVSS) | Identifiants | Correctif | Chargé en runtime ? | Chemin vulnérable exercé ? | Lot |
|---|---|---|---|---|---|---|---|---|---|
| `io.netty:netty-handler` | 4.1.135.Final | 4.1.137.Final | 4.1.137.Final | Critique (9.1) | GHSA-c4c3-7fpv-j4q5 / CVE-2026-75595 | patch | Oui, en client seulement (Lettuce, azure-core-http-netty) | Non: `SniHandler` est côté serveur, l'app n'ouvre aucun serveur Netty | 1 |
| `org.apache.tomcat.embed:tomcat-embed-core` | 10.1.55 | 10.1.59 | 10.1.59 | Critique (9.8) | GHSA-9xv2-5v5q-p794 / CVE-2026-65905 | patch | Oui, serveur HTTP embarqué (jar) | Non: DIGEST, FORM et security-constraints non utilisés (auth = `ApiKeyFilter` maison) | 1 |
| `org.apache.tomcat.embed:tomcat-embed-core` | 10.1.55 | 10.1.59 | 10.1.59 | Critique (9.1) | GHSA-gcx9-497g-6cp6 / CVE-2026-65182 | patch | Oui, serveur HTTP embarqué (jar) | Non: DIGEST, FORM et security-constraints non utilisés (auth = `ApiKeyFilter` maison) | 1 |
| `org.apache.tomcat.embed:tomcat-embed-core` | 10.1.55 | 10.1.59 | 10.1.59 | Critique (9.1) | GHSA-h3x4-894j-xpx5 / CVE-2026-68525 | patch | Oui, serveur HTTP embarqué (jar) | Non: DIGEST, FORM et security-constraints non utilisés (auth = `ApiKeyFilter` maison) | 1 |
| `com.fasterxml.jackson.core:jackson-databind` | 2.21.4 | 2.21.6 | 2.21.6 | Haute (7.5) | GHSA-q4xh-88c3-wmh7 / CVE-2026-68497 | patch | Oui, JSON des requêtes/réponses et cache Redis (jar) | Non: aucun DTO `Duration`/`XMLGregorianCalendar` (`Duration` sert au TTL du cache) | 2 |
| `io.netty:netty-codec` | 4.1.135.Final | 4.1.136.Final | 4.1.137.Final | Haute (8.7) | GHSA-558v-64gr-wgg4 / CVE-2026-59901 | patch | Oui, en client seulement (Lettuce, azure-core-http-netty) | Non: `Bzip2Decoder` non câblé par les clients Azure/Lettuce | 1 |
| `io.netty:netty-codec-http` | 4.1.135.Final | 4.1.136.Final | 4.1.137.Final | Haute (7.5) | GHSA-6jqx-86gh-f27w / CVE-2026-55831 | patch | Oui, en client seulement (Lettuce, azure-core-http-netty) | Improbable: client sortant, pairs de confiance (Azure Storage, Redis); décodeurs SPDY/CORS/multipart serveur non utilisés | 1 |
| `io.netty:netty-codec-http` | 4.1.135.Final | 4.1.136.Final | 4.1.137.Final | Haute (8.7) | GHSA-jppx-w49h-x2qq / CVE-2026-56745 | patch | Oui, en client seulement (Lettuce, azure-core-http-netty) | Improbable: client sortant, pairs de confiance (Azure Storage, Redis); décodeurs SPDY/CORS/multipart serveur non utilisés | 1 |
| `io.netty:netty-codec-http` | 4.1.135.Final | 4.1.136.Final | 4.1.137.Final | Haute (7.5) | GHSA-mvh2-crg5-v77c / CVE-2026-55833 | patch | Oui, en client seulement (Lettuce, azure-core-http-netty) | Improbable: client sortant, pairs de confiance (Azure Storage, Redis); décodeurs SPDY/CORS/multipart serveur non utilisés | 1 |
| `io.netty:netty-codec-http2` | 4.1.135.Final | 4.1.136.Final | 4.1.137.Final | Haute (7.5) | GHSA-93wv-jw9v-4972 / CVE-2026-56819 | patch | Oui, en client seulement (Lettuce, azure-core-http-netty) | Improbable: client sortant, pairs de confiance (Azure Storage, Redis); décodeurs SPDY/CORS/multipart serveur non utilisés | 1 |
| `org.postgresql:postgresql` | 42.7.11 | 42.7.12 | 42.7.12 | Haute (8.2) | GHSA-j92g-9f8w-j867 / CVE-2026-54291 | patch | Oui, driver JDBC (jar) | Inconnu: dépend de `channelBinding=require` dans l'URL JDBC, portée par le Secret (Key Vault), invisible dans le dépôt | 1 |
| `tools.jackson.core:jackson-databind` | 3.1.4 | 3.1.6 | 3.1.6 | Haute (7.5) | GHSA-q4xh-88c3-wmh7 / CVE-2026-68497 | patch | Sur le classpath (transitif spring-cloud-azure 7.4.0), usage à confirmer | Non: aucun DTO `Duration`/`XMLGregorianCalendar` (`Duration` sert au TTL du cache) (aucun import `tools.jackson` dans `src/`) | 2 |
| `com.fasterxml.jackson.core:jackson-databind` | 2.21.4 | 2.21.5 | 2.21.6 | Moyenne (6.5) | GHSA-5gvw-p9qm-jgwh / CVE-2026-59889 | patch | Oui, JSON des requêtes/réponses et cache Redis (jar) | Non: aucun `@JsonView`/`@JsonUnwrapped` dans `src/` | 2 |
| `com.fasterxml.jackson.core:jackson-databind` | 2.21.4 | 2.21.5 | 2.21.6 | Moyenne (5.3) | GHSA-5jmj-h7xm-6q6v / CVE-2026-54515 | patch | Oui, JSON des requêtes/réponses et cache Redis (jar) | Non: désérialisation insensible à la casse non activée (ni code ni `application.yml`) | 2 |
| `com.fasterxml.jackson.core:jackson-databind` | 2.21.4 | 2.21.6 | 2.21.6 | Moyenne (5.6) | GHSA-gx83-3vf8-gh7j / CVE-2026-83557 | patch | Oui, JSON des requêtes/réponses et cache Redis (jar) | Inconnu: dépend du default typing de `GenericJackson2JsonRedisSerializer` (`CacheConfig`), données lues dans le Redis de l'app | 2 |
| `com.fasterxml.jackson.core:jackson-databind` | 2.21.4 | 2.21.5 | 2.21.6 | Moyenne (6.5) | GHSA-mhm7-754m-9p8w | patch | Oui, JSON des requêtes/réponses et cache Redis (jar) | Non: aucun `@JsonView` dans `src/` | 2 |
| `com.fasterxml.jackson.core:jackson-databind` | 2.21.4 | 2.21.5 | 2.21.6 | Moyenne (5.3) | GHSA-vvgp-rfg2-7rr6 / CVE-2026-77310 | patch | Oui, JSON des requêtes/réponses et cache Redis (jar) | Non: aucun champ `java.net.URL`/`InetAddress` dans `src/` | 2 |
| `com.fasterxml.jackson.core:jackson-databind` | 2.21.4 | 2.21.6 | 2.21.6 | Moyenne (5.3) | GHSA-wjgm-6hv5-3cvf / CVE-2026-19032 | patch | Oui, JSON des requêtes/réponses et cache Redis (jar) | Non: aucun champ `java.nio.file.Path` dans `src/` | 2 |
| `io.netty:netty-codec-dns` | 4.1.135.Final | 4.1.136.Final | 4.1.137.Final | Moyenne (5.3) | GHSA-mfg7-5gfp-c4w3 / CVE-2026-73508 | patch | Oui, en client seulement (Lettuce, azure-core-http-netty) | Improbable: réponses DNS d'un résolveur de confiance | 1 |
| `io.netty:netty-codec-http` | 4.1.135.Final | 4.1.136.Final | 4.1.137.Final | Moyenne (6.3) | GHSA-4mp9-239f-g9hg / CVE-2026-59898 | patch | Oui, en client seulement (Lettuce, azure-core-http-netty) | Improbable: client sortant, pairs de confiance (Azure Storage, Redis); décodeurs SPDY/CORS/multipart serveur non utilisés | 1 |
| `io.netty:netty-codec-http` | 4.1.135.Final | 4.1.136.Final | 4.1.137.Final | Moyenne (6.5) | GHSA-6cqp-g7gg-8hr5 / CVE-2026-56746 | patch | Oui, en client seulement (Lettuce, azure-core-http-netty) | Improbable: client sortant, pairs de confiance (Azure Storage, Redis); décodeurs SPDY/CORS/multipart serveur non utilisés | 1 |
| `io.netty:netty-codec-http` | 4.1.135.Final | 4.1.137.Final | 4.1.137.Final | Moyenne (6.5) | GHSA-8c42-7qj2-3j46 / CVE-2026-59903 | patch | Oui, en client seulement (Lettuce, azure-core-http-netty) | Improbable: client sortant, pairs de confiance (Azure Storage, Redis); décodeurs SPDY/CORS/multipart serveur non utilisés | 1 |
| `io.netty:netty-codec-http` | 4.1.135.Final | 4.1.136.Final | 4.1.137.Final | Moyenne (5.7) | GHSA-gcjf-9mgh-3p7g / CVE-2026-59921 | patch | Oui, en client seulement (Lettuce, azure-core-http-netty) | Improbable: client sortant, pairs de confiance (Azure Storage, Redis); décodeurs SPDY/CORS/multipart serveur non utilisés | 1 |
| `io.netty:netty-codec-http` | 4.1.135.Final | 4.1.136.Final | 4.1.137.Final | Moyenne (6.9) | GHSA-q4f6-jm68-57ww / CVE-2026-59899 | patch | Oui, en client seulement (Lettuce, azure-core-http-netty) | Improbable: client sortant, pairs de confiance (Azure Storage, Redis); décodeurs SPDY/CORS/multipart serveur non utilisés | 1 |
| `io.netty:netty-codec-http2` | 4.1.135.Final | 4.1.136.Final | 4.1.137.Final | Moyenne (6.9) | GHSA-c69g-56f8-xwqj / CVE-2026-59900 | patch | Oui, en client seulement (Lettuce, azure-core-http-netty) | Improbable: client sortant, pairs de confiance (Azure Storage, Redis); décodeurs SPDY/CORS/multipart serveur non utilisés | 1 |
| `io.netty:netty-handler` | 4.1.135.Final | 4.1.137.Final | 4.1.137.Final | Moyenne (6.9) | GHSA-fccg-mwvh-qqg4 / CVE-2026-75596 | patch | Oui, en client seulement (Lettuce, azure-core-http-netty) | Non: même raison (handshake TLS serveur) | 1 |
| `org.apache.commons:commons-lang3` | 3.17.0 | 3.18.0 | 3.18.0 | Moyenne (6.5) | GHSA-j288-q9x7-2f5v / CVE-2025-48924 | mineur | Probable, via swagger-core-jakarta (springdoc), à confirmer | Non: le code applicatif n'appelle pas `commons-lang3` | 2 |
| `org.apache.logging.log4j:log4j-api` | 2.24.3 | 2.25.5 | 2.25.5 | Moyenne (6.3) | GHSA-qv9r-c865-cp47 / CVE-2026-49844 | mineur | Oui, pont log4j-to-slf4j (logs vers Logback) | Non: pas de Log4j Core (layouts absents), les logs passent par Logback | 2 |
| `tools.jackson.core:jackson-databind` | 3.1.4 | 3.1.5 | 3.1.6 | Moyenne (6.5) | GHSA-5gvw-p9qm-jgwh / CVE-2026-59889 | patch | Sur le classpath (transitif spring-cloud-azure 7.4.0), usage à confirmer | Non: aucun `@JsonView`/`@JsonUnwrapped` dans `src/` (aucun import `tools.jackson` dans `src/`) | 2 |
| `tools.jackson.core:jackson-databind` | 3.1.4 | 3.1.6 | 3.1.6 | Moyenne (5.6) | GHSA-gx83-3vf8-gh7j / CVE-2026-83557 | patch | Sur le classpath (transitif spring-cloud-azure 7.4.0), usage à confirmer | Inconnu: dépend du default typing de `GenericJackson2JsonRedisSerializer` (`CacheConfig`), données lues dans le Redis de l'app (aucun import `tools.jackson` dans `src/`) | 2 |
| `tools.jackson.core:jackson-databind` | 3.1.4 | 3.1.5 | 3.1.6 | Moyenne (5.3) | GHSA-vvgp-rfg2-7rr6 / CVE-2026-77310 | patch | Sur le classpath (transitif spring-cloud-azure 7.4.0), usage à confirmer | Non: aucun champ `java.net.URL`/`InetAddress` dans `src/` (aucun import `tools.jackson` dans `src/`) | 2 |
| `tools.jackson.core:jackson-databind` | 3.1.4 | 3.1.6 | 3.1.6 | Moyenne (5.3) | GHSA-wjgm-6hv5-3cvf / CVE-2026-19032 | patch | Sur le classpath (transitif spring-cloud-azure 7.4.0), usage à confirmer | Non: aucun champ `java.nio.file.Path` dans `src/` (aucun import `tools.jackson` dans `src/`) | 2 |
