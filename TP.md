# TP de CI-CD

> Equipe:
> - JACQUOT William
> - MARCANT Aime

Nous avons choisi le projet brick-game en java avec maven. 
Pour mettre en place le CI-CD, nous avons utilisé les github actions et nous allons expliquer notre démarche.

# Sommaire

1. [Github Actions](#Github-Actions)
2. [CI](#CI)
3. [CD](#CD)


# Github Actions

Les github actions sont des workflows lancés sur des machines distantes appelées **runners**, on peut utiliser des runners de github ou des runners *"self-hosted"*.

```yml
name: Hello-World

on:
  push:
    branches: [ main, dev ]
  workflow_dispatch:

jobs:
  print:
    runs-on: ubuntu-latest
      steps:
        - name: Hello World
        run: echo "hello world"

        - name: Check out repository code
          uses: actions/checkout@v2

        - name: List files in the repository
          run: |
            ls ${{ ![image](https://user-images.githubusercontent.com/47029620/151785248-fffb7164-44dd-48bb-81d7-fec4dd10f30b.png)ithub.workspace }}
```
Ci dessus un workflow basique, qui va faire des actions assez sommaire, nous allons l'expliquer pas a pas.

**Instructions**: 

`on` : ce qui suit ce selecteur indique sur quoi l'action va se déclancher, ici cela se fait sur un `push` sur les branches main et dev.  le `workflow_dispatch` permet de trigger l'action a la main.
`jobs` : ici on retrouve les différents ensemble d'étapes/commandes (*jobs*).
`steps` : dans un job, les *steps* sont des étapes ou des commandes, Elles peuvent etre des commandes shell (ex: `run: echo "hello world"`) ou des actions qui ont été créé par d'autres et qu'on utilise dans notre worflow.  L'action `uses: actions/checkout@v2` permet au runner de recuperer le code de notre repo. 

Github nous mets aussi a disposition des variables d'environnement liée au repo ici `${{ github.workspace }}`  nous donne le chemin vers le repo dans le runner.


# CI

```yml
name: CI / Tests / main
concurrency: ci-dev-tests

on:
  pull_request:
    branches: [ main ]
  workflow_dispatch:
  
jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v2
      - name: Set up JDK 8
        uses: actions/setup-java@v2
        with:
          java-version: '8'
          distribution: 'adopt'
      - name: Tests with Maven
        run: mvn test
```

Ceci est le workflow du CI, il est trigger sur une pull request pour verifier que les tests passent avant de merge les changements. 
Pour ce faire nous avons utilisé l'action `uses: actions/setup-java@v2` qui va installer le JDK de la version précisée en paramètres. Ensuite la commande mvn test lance les tests (la commande `mvn` provient du JDK), si les tests réussissent la branche peut être merge.



# CD
```yml
name: CD / Deploy / release
concurrency: cd-production

on:
  push:
    tags:
      - 'v*.*.*'
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v2
      - name: Set up JDK 8
        uses: actions/setup-java@v2
        with:
          java-version: '8'
          distribution: 'adopt'
          
      - name: Print tag name
        run: echo ${REF_TAG:1}
        env:
          REF_TAG: ${{ github.ref_name }}
        
      - name: Build with Maven
        run: mvn compile
      - name: Package with Maven
        run: mvn package
        
      - name: Nexus Repo Publish
        uses: sonatype-nexus-community/nexus-repo-github-action@master
        with:
          serverUrl: ${{ secrets.nexus_url }}
          username: ${{ secrets.nexus_username }}
          password: ${{ secrets.nexus_password }}
          format: maven2
          repository: maven-releases
          coordinates: groupId=com.github.vitalibo artifactId=brick-game version=${{ github.ref_name }}
          assets: extension=jar
          filename: ./target/brick-game.jar
```

Voici le worklow du CD, il est trigger au moment de créer une release (un tag sur git) qui suit la regex `v*.*.*` (ex: v0.6.3). 
Comme précédemment, nous récupérons le JDK et on creer le jar avec maven `mvn package`. Nous avons utilisé une action qui permet de se connecter et d'envoyer une release sur un server nexus `uses: sonatype-nexus-community/nexus-repo-github-action@master`, qui prend en paramètres l'adresse du server nexus, le nom d'utilisateur ainsi que le mot de passe. Pour des raisons de sécurité ces informations sont stockés dans les **secrets** du repo Github, il faut les renseigner à la main dans les paramètres du repo de github. 

![image](https://user-images.githubusercontent.com/47029620/151785275-5df0d397-1a6b-4edc-9052-107ebd57b974.png)

On précise le format ici `maven2`, le repository `maven-releases` et des informations complémentaires le `groupId`, `artifactId`et la `version`. L'instruction `assets` permet de dire les propriétés du fichier a envoyer, et `filename` est le chemin du fichier en question.
