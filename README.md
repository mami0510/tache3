<!--
Licensed to the Apache Software Foundation (ASF) under one or more
contributor license agreements.  See the NOTICE file distributed with
this work for additional information regarding copyright ownership.
The ASF licenses this file to You under the Apache License, Version 2.0
(the "License"); you may not use this file except in compliance with
the License.  You may obtain a copy of the License at

     http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
-->

# Tâche 3

## Semgrep pour la détection des vulnérabilités dans GitHub Actions

### Objectif

L'objectif de la modification des workflows Github Actions est de faire échouer le build si une nouvelle vulnérabilité est introduite dans le code, meme si le code on possède déjà on ne veut pas en ajouter.

En considère une vulnérabilité tout finding qui n'existait pas dans le code de base (baseline) et qui est détectée par Semgrep.

### Conception

#### Comparaison avec la baseline

Pour concevoir cette solution, il va falloir qu'on puisse comparer les résultats de Semgrep avec le pack `p/java` (adapter à notre projet) du code actuel sur le code de base (baseline).

On voulait d'abord générer un rapport JSON  pour chaque build et les comparer, mais il se trouve que ce n'est pas une solution efficace, car il ne suffit pas de compter le nombre de findings, mais plutot garder leurs informations/identifiants pour pouvoir comparer et plus encore. Heureusment, Semgrep inclut une fonctionnalité qui permet de faire la comparaison directement entre un commit et une baseline.

#### Choix de la baseline

Pour choisir la baseline, on a choisi d'utiliser un commit spécifique avant toute modification de workflow `--baseline-commit SHA` qui passe les workflows et qui contient 22 findings.

On a préféré ceci par rapport à l'option d'utiliser le dernier commit puisqu'on peut le contourner en soumettant un commit deux fois, il ne passera pas la premiere fois mais mais la deuxieme fois oui, car il sera la baseline de lui meme. On evite cela en specifiant un commit précis.

#### Code de sortie

Pour la sortie `semgrep scan` retourne 0 meme si des vulnérabilités sont détectées, donc on utilise `--error` pour retourner un code d'erreur 1 si il y'en a des nouvelles et à partir de la, le build échouera.

Ce qui nous donne la ligne de commande suivante pour le scan Semgrep:

```bash
 semgrep scan --config p/java --metrics off --baseline-commit ${{env.SEMGREP_DEPART}} --error
 ```

#### Installation et connexion

Avant de pouvoir l'utiliser, il faut d'abord l'installer. Pour cela on ajoute une etape dans le workflow pour installer python ver. 3.13.5 (celle que j'avais dans mon ordi) et ensuite on installe Semgrep avec pip  on specifie la version 1.185.0 pour eviter tout probleme du a un changement de version, enfin on termine par se connecter à Semgrep avec notre token pour avoir acces au pack `p/java` au complet (241 vs 60 regles).

Ce qui nous donne cette partie du workflow:

```yaml
 - uses: actions/setup-python@v7 
 ##ajouter une version python a notre workflow
        with:
          python-version: "3.13.5"
      - name: Install Semgrep
        run: python -m pip install semgrep==1.185.0  
      - name: Login Semgrep 
      ##se connecter pour avoir tout le package java
        run: semgrep login
      - name: Run Semgrep with java pack 
      ## lancer et comparer avec la baseline fail si trouve une difference
        run: semgrep scan --config p/java --metrics off --baseline-commit ${{env.SEMGREP_DEPART}} --error
```

#### Variables d'environnement

On doit aussi ajouter 2 variable avec `env`: `SEMGREP_APP_TOKEN` qui est notre token pour ce connecter et `SEMGREP_DEPART` qui contient notre commit baseline

```yaml
    env:
      SEMGREP_APP_TOKEN: ${{secrets.SEMGREP_APP_TOKEN}} 
      SEMGREP_DEPART: 18e58e86b910e4d878730af329a559b60308b99d
```

#### Historique du dépot

Autre chose importante, il faut que le workflow puisse accéder à l'historique complet du repo pour pouvoir comparer avec le commit baseline, donc on ajoute `fetch-depth: 0` dans l'étape checkout.

### Implementation

Il y'a 5 fichier workflow au total (jdk17, jdk17 Windows, jdk17 Windows multi-locale, jdk21 et jdk25) qu'on va considerer. Ils vont contenir tous le meme changement vu plus tot. La seule différence va etre avec le workflows `main-jdk17-windows-build.yml` qui utilise un répertoire different et dans lequel on doit executer Semgrep: `working-directory:'tika build dir'`

La position de l'etape Semgrep est importante, puisqu'on veut que le build échoue si on detecte de nouvelles vulnérabilités, donc on doit le mettre avant l'etape de build (Build with Maven) mais pour plus de clarite, on le mets avant le Set up JDK.

### Validation

Pour valider que notre solution fonctionne, on a fait 6 scenarios de validation. Voici les 5 premiers dans le tableau ci-dessous:

| # | Scénario | Résultat attendu | Résultat obtenu | Run |
| --- | --- | --- | --- | --- |
| 1 | Commit qui ne modifie que les fichier de workflows (pas java) | checks ✅,  Semgrep n'analyse rien  | checks ✅, Semgrep a affiché « Nothing to scan »  | [Github commit](https://github.com/mami0510/tache3/commit/7fd9bc7199d040887bd38b7ebaf1774b37413818)|
| 2 | Commit qui ajoute une ligne de commentaire dans ExceptionUtils.java, fichier qui a déjà un finding dans la baseline | checks ✅, 0 nouveau finding malgré la modification du fichier | checks ✅, 0 finding | [Github commit](https://github.com/mami0510/tache3/commit/135d70d5198f00f4d7ad00befc1f144748ccabd3) |
| 3 | Commit qui ajoute un nouveau fichier Java avec un `e.printStackTrace()` dans un `catch` (règle `active-debug-code-printstacktrace`, présente dans la baseline pour d'autres fichiers) | checks ❌, 1 nouveau finding dans les logs, build echoue | checks ❌, Les cinq workflows de build échouent à l'étape Semgrep, qui détecte un nouveau finding | [Github commit](https://github.com/mami0510/tache3/commit/526d11498dba2fa5fca28c4c04c0cb428cd950b5) |
| 4 | Commit supplémentaire par-dessus le commit du scénario 3, sans corriger le finding | checks ❌, le finding reste absent de la baseline | checks ❌, meme résultat qu'au scénario 3 : les workflows échouent à l'étape Semgrep, le finding de Vuln.java reste détecté | [Github commit](https://github.com/mami0510/tache3/commit/7507a6dfb9cbcd133460b60efd19132fae24de77) |
| 5 | Commit qui retire le fichier Vuln.java | checks ✅, build exécuté | checks ✅, Les workflows repassent : l'étape Semgrep est passe une fois le fichier retiré | [Github commit](https://github.com/mami0510/tache3/commit/2fee7727b308c802e7462fc0696624139c510267) |

Pour la derniere validation ([6](https://github.com/mami0510/tache3/pull/1)), on crée une branch à partir de `main` et on ajoute le fichier `selftest/Vuln.java` et on fait un pull request vers `main`, en changeant un setting dans branch protection rules pour que le pull request ne peut pas etre merge si les checks ne passent pas. On fait cette validation pour tester les workflows qui regarde les pulls comme signal.

**Résultat:** on voit que le pull request est bloqué et on ne peut pas le merger tant que le fichier Vuln.java n'est pas retiré ou le finding corrigé.


# Vulnérabilités introduites dans Apache Tika

## Objectif et portée

Introduire **7 types différents de faiblesses de sécurité détectables par Semgrep** dans une copie d'Apache Tika et comparer les résultats avant et après les modifications.

- Dépôt : https://github.com/PhuongAnh-Tran/tika
- Branche : `tache3-vulnerabilites`
- Commit de départ : `a2d75c2c10563b261148e9bafe4046832b33c1a6`
- Fichiers modifiés : `DigestHelper.java`, `NetworkParser.java`, `XMLReaderUtils.java` et `H2Util.java`.

Les ajouts sont destinés uniquement à l'exercice. Le cas MD5 est activable explicitement; les six autres exemples sont des méthodes de démonstration non appelées par le fonctionnement habituel de Tika. **Les détections Semgrep ne prouvent pas que les fonctionnalités normales de Tika sont exploitables.**

## Configuration et comparaison Semgrep

- Version : **Semgrep 1.185.0**
- Règles : **`p/java`** (60 règles Java au moment des analyses)
- Périmètre : dépôt Tika, fichiers suivis par Git, avec les exclusions Semgrep habituelles.
- Rapport initial : `semgrep_avant_nouveau.json`
- Rapport final : `semgrep_apres_final.json`

Commande initiale :

```bash
semgrep scan --config p/java --metrics=off --json --output semgrep_avant_nouveau.json .
```

Commande finale :

```bash
semgrep scan --config p/java --metrics=off --json --output semgrep_apres_final.json .
```

**Résultats : 11 détections avant, 19 après, soit 8 nouvelles détections réparties en 7 types différents.**

Les deux scans portent sur 1 468 fichiers Java. La différence a été vérifiée à partir des identifiants de règles, des chemins des fichiers et des emplacements des détections.

Deux règles signalent le même usage d'AES-ECB : cela compte pour **un seul type** de vulnérabilité.

## Vulnérabilités introduites

### 1. Hachage faible : MD5

- **Fichier :** `tika-core/src/main/java/org/apache/tika/digest/DigestHelper.java`, ligne **109**.
- **Pourquoi ce fichier :** `DigestHelper` calcule les empreintes de flux pendant l'analyse de documents.
- **Modification :** ajout d'une branche activable par un marqueur `ParseContext`, qui calcule `MessageDigest.getInstance("MD5")` et conserve l'empreinte dans les métadonnées.
- **Risque :** MD5 ne résiste pas suffisamment aux collisions pour les usages de sécurité qui exigent cette propriété.
- **Semgrep :** `java.lang.security.audit.crypto.use-of-md5.use-of-md5` — **1 nouvelle détection**.

### 2. Chiffrement AES en mode ECB

- **Fichier :** `tika-core/src/main/java/org/apache/tika/digest/DigestHelper.java`, ligne **155**.
- **Pourquoi ce fichier :** la classe traite déjà des opérations cryptographiques liées aux empreintes de documents; elle sert ici de support à la démonstration.
- **Modification :** ajout de `ift3913DemoEcb`, qui initialise `Cipher` avec `AES/ECB/NoPadding`.
- **Risque :** ECB chiffre chaque bloc indépendamment; des blocs identiques peuvent révéler des motifs. Ce mode ne fournit pas non plus d'authentification.
- **Semgrep :**
  - `java.lang.security.audit.crypto.ecb-cipher.ecb-cipher`
  - `java.lang.security.audit.crypto.use-of-aes-ecb.use-of-aes-ecb`
- **Résultat :** 2 nouvelles détections, mais 1 seul type de vulnérabilité.

### 3. Réutilisation d'un nonce GCM

- **Fichier :** `tika-core/src/main/java/org/apache/tika/digest/DigestHelper.java`, ligne **166**.
- **Pourquoi ce fichier :** même contexte de démonstration cryptographique que l'exemple AES-ECB.
- **Modification :** ajout de `ift3913DemoGcm`, qui utilise un nonce constant avec `AES/GCM/NoPadding`.
- **Risque :** réutiliser un nonce avec la même clé GCM compromet les garanties de confidentialité et d'authenticité.
- **Semgrep :** `java.lang.security.audit.crypto.gcm-nonce-reuse.gcm-nonce-reuse` — **1 nouvelle détection**.

### 4. Communication réseau non chiffrée

- **Fichier :** `tika-core/src/main/java/org/apache/tika/parser/NetworkParser.java`, ligne **180**.
- **Pourquoi ce fichier :** `NetworkParser` contient déjà du code de communication réseau.
- **Modification :** ajout de `ift3913DemoPlainSocket`, qui crée une connexion avec `new Socket(host, port)` sans protection TLS.
- **Risque :** si des données sensibles circulent sur cette connexion sans protection supplémentaire, elles peuvent être interceptées ou modifiées.
- **Semgrep :** `java.lang.security.audit.crypto.unencrypted-socket.unencrypted-socket` — **1 nouvelle détection**.

Une autre occurrence de cette règle, à la ligne 71, existait déjà avant les modifications. Elle n'est donc pas comptée comme une nouvelle détection.

### 5. Version obsolète de TLS

- **Fichier :** `tika-core/src/main/java/org/apache/tika/parser/NetworkParser.java`, ligne **189**.
- **Pourquoi ce fichier :** la configuration du protocole de transport relève des communications réseau.
- **Modification :** ajout de `ift3913DemoWeakTls`, qui demande `SSLContext.getInstance("TLSv1")`.
- **Risque :** TLS 1.0 est obsolète et n'offre pas les garanties attendues d'une configuration TLS moderne.
- **Semgrep :** `java.lang.security.audit.weak-ssl-context.weak-ssl-context` — **1 nouvelle détection**.

### 6. Analyse XML sans protections XXE explicites

- **Fichier :** `tika-core/src/main/java/org/apache/tika/utils/XMLReaderUtils.java`, ligne **1314**.
- **Pourquoi ce fichier :** cette classe centralise la création et la configuration des analyseurs XML de Tika.
- **Modification :** ajout de `ift3913DemoUnsafeXml`, qui construit un `DocumentBuilderFactory` puis un `DocumentBuilder` sans désactiver explicitement les entités externes.
- **Risque :** suivant la configuration effective du parseur, un document XML malveillant peut déclencher la résolution d'entités externes (XXE), avec des conséquences possibles sur les fichiers locaux ou les accès réseau.
- **Semgrep :** `java.lang.security.audit.xxe.documentbuilderfactory-disallow-doctype-decl-missing.documentbuilderfactory-disallow-doctype-decl-missing` — **1 nouvelle détection**.

Une autre détection liée aux XXE existait déjà à la ligne 307. Elle n'est pas comptée comme nouvelle.

### 7. Construction non sûre d'une requête SQL

- **Fichier :** `tika-eval/tika-eval-app/src/main/java/org/apache/tika/eval/app/db/H2Util.java`, ligne **101**.
- **Pourquoi ce fichier :** `H2Util` contient déjà les opérations JDBC liées à la base H2 utilisée par Tika Eval.
- **Modification :** ajout de `ift3913DemoUnsafeSql`, qui concatène un nom de table fourni en paramètre dans `SELECT * FROM ...` et exécute la requête avec `Statement`.
- **Risque :** sans validation stricte du nom de table, la concaténation peut permettre de modifier la requête SQL exécutée.
- **Semgrep :** `java.lang.security.audit.formatted-sql-string.formatted-sql-string` — **1 nouvelle détection**.

Une occurrence à la ligne 70 existait déjà avant les modifications. Elle n'est pas comptée comme nouvelle.

## Validation

### Analyse Semgrep

L'analyse finale a été exécutée avec succès.

- Avant les modifications : **11 détections**
- Après les modifications : **19 détections**
- Différence : **8 nouvelles détections**
- Types de vulnérabilités représentés : **7**
- Règles Java utilisées : **60**
- Fichiers Java analysés : **1 468**

Les nouvelles détections ont été identifiées en comparant les identifiants de règles, les chemins et les emplacements des résultats des deux rapports JSON.

### Compilation Maven

Commande exécutée :

```bash
./mvnw -pl tika-core,tika-eval/tika-eval-app -am -DskipTests -Dcheckstyle.skip=true test-compile
```

**Résultat : `BUILD SUCCESS`**

Les sources principales et les sources de test des modules concernés et de leurs dépendances ont été compilées avec succès.

### Vérification Git

La commande suivante a été exécutée :

```bash
git diff --cached --check
```

Aucune erreur n'a été signalée.

### Limites de la validation

- Les tests unitaires n'ont pas été exécutés (`-DskipTests`).
- Checkstyle a été désactivé pour la compilation (`-Dcheckstyle.skip=true`).
- Une exécution antérieure de Checkstyle avait signalé 59 erreurs qui n'ont pas été analysées dans cette validation.
- Semgrep a ignoré certains fichiers selon ses règles d'exclusion et sa limite de taille. Le scan indiquait environ 99,9 % des lignes analysées.
- Le paquet `p/java` peut évoluer au fil du temps; les résultats correspondent à la configuration utilisée lors de l'exercice.
- Les exemples introduits sont isolés ou activables explicitement. Ils montrent des motifs de code considérés comme dangereux par analyse statique, sans démontrer leur exploitation dans le fonctionnement normal de Tika.

## Utilisation de l'IA

ChatGPT a été utilisé pour proposer des modifications Java, interpréter les résultats Semgrep et aider à rédiger la documentation.

Les modifications ont été appliquées par les auteurs. Les détections Semgrep et la compilation Maven ont été vérifiées dans leur environnement.

Les validations non exécutées, notamment les tests unitaires et Checkstyle complet, ne sont pas présentées comme réussies.
