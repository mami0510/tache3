# Tache 3

## Semgrep pour la détection des vulnérabilités dans GitHub Actions

### Objectif: 

L'objectif de la modification des workflows Github Actions est de faire échouer le build si une nouvelle vulnérabilité est introduite dans le code, meme si le code on possède déjà on ne veut pas en ajouter.

En considère une vulnérabilité tout finding qui n'existait pas dans le code de base (baseline) et qui est détectée par Semgrep.

### Conception: 

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
