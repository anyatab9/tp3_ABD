
devops-livecoding
Description du Projet
Ce projet illustre une implémentation DevOps basée sur les technologies suivantes :

Testcontainers pour le testing des bases de données en environnement conteneurisé.
GitHub Actions pour l'intégration continue (CI).
Ansible pour le déploiement automatisé des applications avec Docker.
1. Testcontainers : Pourquoi et Quand ?
Testcontainers est une bibliothèque Java qui facilite l'écriture de tests JUnit en fournissant des instances éphémères de services comme :

Bases de données légères (ex: PostgreSQL).
Navigateurs Selenium pour tests UI.
Tout service pouvant tourner dans un conteneur Docker.
Cela garantit des tests reproductibles et isolés de l'environnement local.

2. Documentation du Workflow GitHub Actions
📄 Fichier : main.yml
Ce workflow CI exécute des vérifications automatiques à chaque modification du code.

⚙️ Conditions de Déclenchement
Le workflow s'exécute lorsque :

Un push est effectué sur les branches main ou dev.
Une Pull Request est ouverte.
🧪 Job Principal : test-backend
Environnement : ubuntu-22.04
Objectif : Construction et tests du projet.
Étapes Clés :
Récupération du Code
Utilisation de actions/checkout@v2.5.0 pour cloner le repository.

Configuration de Java (JDK 17)
Configuration de l'environnement Java avec actions/setup-java@v3 (Amazon Corretto).

Compilation et Tests
Exécution de Maven :


mvn clean verify
Note : Le répertoire de travail est spécifié via working-directory.

🚀 Objectif du Workflow
Ce processus garantit la validation automatique du code avant chaque fusion, assurant la stabilité des branches main et dev.

3. Pourquoi Pousser des Images Docker ?
La centralisation des images Docker dans un registre permet :

La cohérence des déploiements sur différents environnements.
La gestion des versions et le rollback facilité en cas d'erreur.
Une intégration simplifiée dans les workflows CI/CD pour des builds et déploiements automatisés.
Dans une architecture microservices, cela facilite l'indépendance des composants pour le scaling et les mises à jour.

4. Configuration Ansible
📁 Structure du Projet
Le répertoire Ansible est organisé comme suit :

devops-livecoding/
└── ansible/
    └── inventories/
        └── setup.yml
🗂 Fichier d'Inventaire : setup.yml
Ce fichier définit :

Les hôtes (ex : environnements de production).
Les variables globales telles que l'utilisateur SSH et la clé privée.
Exemple de Configuration :

all:
  vars:
    ansible_user: admin
    ansible_ssh_private_key_file: /chemin/vers/cle_privee
  children:
    prod:
      hosts:
        192.168.1.10:
🔧 Commandes Ansible de Base
Tester la connectivité :

ansible all -i inventories/setup.yml -m ping

Lister les hotes : 
ansible-inventory -i inventories/setup.yml --list

Afficher les variables : 
ansible-inventory -i inventories/setup.yml --host hostname_or_IP

Collecter les infos système : 
ansible all -i inventories/setup.yml -m setup -a "filter=ansible_distribution*"


5. Déploiement Automatisé avec Ansible et Docker
📜 Playbook Principal
Le playbook Ansible orchestre le déploiement des différents rôles dans cet ordre :

- hosts: your_server
  become: true
  roles:
    - cleanup_containers
    - install_docker
    - create_network
    - create_volumes
    - launch_database
    - launch_app
    - launch_proxy


Détails des Rôles
1. cleanup_containers
Supprime les conteneurs existants pour repartir sur un environnement propre.
- name: Stop all running containers
  shell: "docker stop $(docker ps -q)"
  ignore_errors: true

- name: Remove all containers
  shell: "docker rm $(docker ps -aq)"
  ignore_errors: true


2. install_docker
Installe Docker et le SDK Python nécessaire.

- name: Install Docker and dependencies
  ansible.builtin.package:
    name: "{{ item }}"
    state: present
  loop:
    - docker
    - python3-pip

- name: Install Docker SDK for Python
  ansible.builtin.pip:
    name: docker

3. create_network
Crée un réseau Docker pour la communication inter-conteneurs.
- name: Create Docker network
  community.docker.docker_network:
    name: my-network
    state: present

4. create_volumes
Crée un volume pour les données persistantes de la base de données.
- name: Create Docker volume for database
  community.docker.docker_volume:
    name: db-volume
    state: present

5. launch_database
Lance le conteneur de la base de données avec les variables nécessaires.
- name: Run Database
  community.docker.docker_container:
    name: my-db
    image: wandrilledioubate/tp-devops-simple-api-database
    env:
      POSTGRES_DB: db
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pwd
    networks:
      - name: my-network
    volumes:
      - db-volume:/var/lib/postgresql/data
    state: started

6. launch_app
Déploie l'application backend et configure sa connexion à la base de données.

- name: Run Backend Application
  community.docker.docker_container:
    name: my-api
    image: wandrilledioubate/tp-devops-simple-api-backend
    networks:
      - name: my-network
    env:
      DATABASE_HOST: my-db
      DATABASE_PORT: "5432"
    state: started

7. launch_proxy
Lance un serveur HTTP pour exposer l'application.
- name: Run HTTP Server
  community.docker.docker_container:
    name: httpd
    image: your_http_server_image
    ports:
      - "80:80"
    networks:
      - name: my-network
    state: started

🚀 Exécution du Déploiement
Pour exécuter le playbook :
ansible-playbook -i inventories/setup.yml playbook.yml
