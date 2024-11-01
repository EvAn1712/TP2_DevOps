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

=======================================================================================================

## Question TP part 03 – Github Ansible

### 3-1 Document your inventory and base commands

Pour l’inventaire, il faut créer un fichier `setup.yml` dans le dossier `inventories` :

```bash
mkdir -p my-project/ansible/inventories
```

avec la structure suivante :
```yml
all:
  vars:
    ansible_user: admin # Nom d'utilisateur pour se connecter aux hôtes
    ansible_ssh_private_key_file: /Users/evan/evan/DevOps/id_rsa  # Chemin de la clé privée SSH
  children:
    prod: # Groupe de serveurs de production
      hosts:
        evan.chinnaya.takima.cloud # IP ou le nom de l'hôte cible
 ```

#### Commandes de Base

Vérification de la connexion avec `ping`

Après avoir configuré l'inventaire, la commande suivante permet de tester la connexion entre le locale et les serveurs distants :

```bash
ansible all -i inventories/setup.yml -m ping
```

- ansible all : Exécute la commande sur tous les hôtes de l'inventaire.
- -i inventories/setup.yml : Spécifie le fichier d’inventaire (setup.yml) à utiliser.
- -m ping : Utilise le module ping pour tester la connexion aux hôtes. Cela permet de confirmer qu’Ansible peut se connecter et communiquer avec chaque serveur.

Les "facts" sont des informations collectées automatiquement par Ansible sur les hôtes cibles. Ces données incluent des détails sur le système d'exploitation, le matériel et la configuration réseau. Utilisez la commande suivante pour récupérer les informations de distribution du système d'exploitation sur chaque hôte :

```bash
ansible all -i inventories/setup.yml -m setup -a "filter=ansible_distribution*"
```
- -m setup : Utilise le module setup d’Ansible pour collecter des informations système (facts) depuis les hôtes.
- -a "filter=ansible_distribution*" : Utilise un filtre pour ne récupérer que les facts associés à ansible_distribution, comme le nom et la version de la distribution du système d'exploitation.

--------------------------------

### 3-2 Document your playbook

Notre premier playbook, `playbook.yml`, était un simple test de connectivité :

```yaml
- hosts: all
  gather_facts: false
  become: true
  tasks:
    - name: Test connection
      ping:
```
Ce playbook vérifie qu'Ansible peut se connecter à tous les hôtes définis dans l'inventaire.

**Playbook Avancé**

Nous avons créé un playbook avancé pour gérer l'installation de Docker. Ce playbook comprend les tâches suivantes :

- Installer les paquets requis : Installer les prérequis comme apt-transport-https, curl, gnupg, et plus.
- Ajouter la clé GPG de Docker : Ajouter la clé GPG officielle de Docker.
- Configurer le dépôt Docker : Mettre en place le dépôt stable de Docker pour télécharger les paquets Docker.
- Installer Docker : Installer le moteur Docker.
- Vérifier que Docker est en cours d'exécution : S'assurer que Docker est actif.

```yml
---
# roles/docker/tasks/main.yml

- name: Install prerequisites for Docker
  apt:
    name:
      - apt-transport-https
      - ca-certificates
      - curl
      - gnupg
      - lsb-release
      - python3-venv
    state: latest
    update_cache: yes

- name: Add Docker GPG key
  apt_key:
    url: https://download.docker.com/linux/debian/gpg
    state: present

- name: Add Docker APT repository
  apt_repository:
    repo: "deb [arch=amd64] https://download.docker.com/linux/debian {{ ansible_facts['distribution_release'] }} stable"
    state: present
    update_cache: yes

- name: Install Docker
  apt:
    name: docker-ce
    state: present

- name: Install Python3 and pip3
  apt:
    name:
      - python3
      - python3-pip
    state: present

- name: Create a virtual environment for Docker SDK
  command: python3 -m venv /opt/docker_venv
  args:
    creates: /opt/docker_venv  # Only runs if this directory doesn’t exist

- name: Install Docker SDK for Python in virtual environment
  command: /opt/docker_venv/bin/pip install docker

- name: Ensure Docker is running
  service:
    name: docker
    state: started
  tags: docker
```
**Rôles et Organisation des Tâches**


En utilisant la commande 
```
ansible-galaxy init roles/docker
```
Nous avons créé un rôle pour l'installation de Docker. Cette structure nous permet de séparer les préoccupations et de modulariser notre code Ansible. Les principaux répertoires utilisés au sein du rôle sont :

- tasks : Contient les tâches principales pour l'installation de Docker.
- handlers : Contient des gestionnaires pour redémarrer Docker si nécessaire.

Le playbook principal est situé dans le fichier `playbook.yml` et exécute les rôles suivants :
- `docker` : Installation et configuration de Docker.
- `create_network` : Création du réseau Docker.
- `launch_database` : Lancement de la base de données en tant que conteneur.
- `launch_app` : Lancement de l'application principale.
- `launch_proxy` : Lancement du proxy HTTP.

**Contenu de `playbook.yml` :**

```yaml
- hosts: all
  gather_facts: true
  become: true

  roles:
    - docker
    - create_network
    - launch_database
    - launch_app
    - launch_proxy
```
Le rôle `docker` installe Docker et ses dépendances, puis s’assure que le service Docker fonctionne.

**Pour exécuter le playbook :**

```bash
ansible-playbook -i inventories/setup.yml playbook.yml
```

### Document your docker_container tasks configuration.


Nous avons organisé notre projet en plusieurs rôles, chacun étant responsable d'une tâche spécifique. Voici les rôles et leur configuration :

1. Créer le Réseau Docker

Le rôle create_network configure un réseau Docker, permettant aux conteneurs de communiquer entre eux :

```yaml

# roles/create_network/tasks/main.yml

- name: Create Docker network
  docker_network:
    name: my-network
    state: present
    driver: bridge
  ```
2. Lancer le Proxy Nginx

Le rôle launch_proxy déploie un conteneur Nginx en tant que proxy :

```yaml

# roles/launch_proxy/tasks/main.yml

- name: Run Nginx Proxy
  docker_container:
    name: httpd
    image: evan024/tp-devops-http-server
    ports:
      - "80:80"
    networks:
      - name: my-network
    state: started
  ```
3. Lancer la Base de Données PostgreSQL

Le rôle launch_database configure un conteneur PostgreSQL avec les variables d'environnement nécessaires :

```yaml

# roles/launch_database/tasks/main.yml

- name: Run PostgreSQL Database
  docker_container:
    name: my-db
    image: evan024/tp-devops-database
    env:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pwd
      POSTGRES_DB: db
    networks:
      - name: my-network
    ports:
      - "5432:5432"
    state: started
  ```
4. Lancer l'Application

Le rôle launch_app déploie un conteneur pour l'application, en lui fournissant également les informations nécessaires pour se connecter à la base de données :

```yaml

# roles/launch_app/tasks/main.yml

- name: Run Application
  docker_container:
    name: my-api
    image: evan024/tp-devops-simple-api
    env:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pwd
      POSTGRES_DB: db
    networks:
      - name: my-network
    ports:
      - "8080:8080"
    state: started
```
