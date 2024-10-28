# TP-DevOps

## Question TP part 02 – Github Action

## 2-1 What are testcontainers?

La bibliothèque Java Testcontainers facilite les tests qui requièrent des ressources externes, telles que des bases de données, en fournissant des conteneurs Docker légers et temporaires. Grâce à elle, les tests peuvent être réalisés dans des environnements séparés, où chaque test possède son propre conteneur Docker temporaire, qui peut être facilement créé et supprimable. Dans notre projet, Testcontainers gère un conteneur PostgreSQL Docker pour assurer une intégration fluide avec l’application lors des tests.

--------------------------------
## 2-2 Document your Github Actions configurations.

#### Configuration des Secrets

Pour sécuriser les informations de connexion à DockerHub, deux secrets doivent être configurés dans les **GitHub Repository Secrets**. Ces secrets permettent d'éviter de stocker les identifiants directement dans le code et garantissent ainsi une meilleure sécurité.

1. **DOCKERHUB_USERNAME** : Votre nom d'utilisateur DockerHub.
2. **DOCKERHUB_TOKEN** : Un jeton d'accès généré depuis DockerHub.

#### Workflow CI/CD
```yaml
name: CI devops 2024

# Déclenche le workflow lors de chaque push ou pull request sur la branche main
on:
  push:
    branches: main
  pull_request:

jobs:
  # Job pour tester le backend
  test-backend:
    runs-on: ubuntu-22.04
    steps:
      # Clone le code depuis le dépôt GitHub
      - name: Checkout code
        uses: actions/checkout@v2.5.0

      # Installe JDK 17 pour le projet
      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'corretto'

      # Compile et teste l'application avec Maven
      - name: Build and test with Maven
        run: mvn clean install --file "./simple-api/pom.xml"

  # Job pour construire et pousser les images Docker
  build-and-push-docker-image:
    # Ce job dépend de la réussite des tests du backend
    needs: test-backend
    runs-on: ubuntu-22.04

    steps:
      - name: Checkout code
        uses: actions/checkout@v2.5.0

      # Connexion à DockerHub avec des secrets qui sont dans le github Action pour protéger les informations sensibles
      - name: Login to DockerHub
        run: docker login -u ${{ secrets.DOCKERHUB_USERNAME }} -p ${{ secrets.DOCKERHUB_TOKEN }}

      # Crée une liste des services à builder et publier
      - name: Build and push Docker images
        uses: docker/build-push-action@v3
        with:
          # Définit le chemin vers chaque Dockerfile et tag pour chaque image
          context: ./simple-api
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/tp-devops-simple-api:latest
          push: ${{ github.ref == 'refs/heads/main' }}

      - name: Build and push database image
        uses: docker/build-push-action@v3
        with:
          context: ./database
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/tp-devops-database:latest
          push: ${{ github.ref == 'refs/heads/main' }}

      - name: Build and push httpd image
        uses: docker/build-push-action@v3
        with:
          context: ./http-server
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/tp-devops-http-server:latest
          push: ${{ github.ref == 'refs/heads/main' }}
```
Pour déclencher le workflow, il faut faire un `push` ou une `pull request` vers la branche `main`.

--------------------------------
## For what purpose do we need to push docker images?

Pousser les images Docker sur un registre comme DockerHub est essentiel dans un pipeline CI/CD pour plusieurs raisons :

1. **Déploiement Consistant** : Les images Docker capturent l'ensemble de l'environnement, assurant ainsi que chaque déploiement utilise une version stable et testée de l'application, éliminant ainsi les variations de configuration.

2. **Travail en équipe** : Une image centralisée sur DockerHub facilite la récupération de la dernière version validée de l'application pour les tests ou le déploiement pour toute l'équipe.

3. **Automatisation CI/CD** : En ajoutant le push des images dans le pipeline, chaque modification de code provoque une mise à jour de l'image Docker, ce qui accélère et garantit la fiabilité du processus de mise en production.

4. **Gestion des Versions** : DockerHub permet de taguer les versions (`latest`, `v1.0`, etc.), facilitant le suivi des versions et le retour rapide à une version antérieure si nécessaire.

5. **Compatibilité Multi-Environnements** : Les images Docker permettent une portabilité de l'application entre différents environnements, assurant ainsi une cohérence dans son fonctionnement, que ce soit en production, en développement ou en test.

Pousser les images Docker permet donc un déploiement **rapide**, **fiable** et **répétable** tout en facilitant la collaboration et la gestion des versions.

--------------------------------
## Document your quality gate configuration.

La configuration du Quality Gate a pour objectif d'assurer une qualité de code élevée et de détecter les vulnérabilités potentielles avant qu'elles n'atteignent la branche principale. Pour cela, nous utilisons SonarCloud, une plateforme d'analyse de qualité de code, intégrée au pipeline GitHub Actions.

#### Étapes de configuration

1. **Inscription à SonarCloud**
   
2. **Création des Tokens Sécurisés** : Ajout du token d'authentification `SONAR_TOKEN` dans les secrets du dépôt GitHub pour garantir une connexion sécurisée avec SonarCloud.

3. **Configuration de l'Analyse Sonarcloud dans le Pipeline** :
  
Modification du fichier `main.yml` pour inclure une étape d’analyse SonarCloud :


   ```yaml
   - name: analyze with SonarCloud
     run: mvn -B verify sonar:sonar \
       -Dsonar.projectKey=EvAn1712_TP2_DevOps \
       -Dsonar.organization=evan1712 \
       -Dsonar.host.url=https://sonarcloud.io \
       -Dsonar.login=${{ secrets.SONAR_TOKEN }} \
       --file ./simple-api/pom.xml
```

Cette commande déclenche une analyse SonarCloud à chaque commit.
