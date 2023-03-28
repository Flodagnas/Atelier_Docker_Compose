# Atelier_Docker_Compose
## Objectifs
Dans ce TP, vous allez :

1. Créer une application multi-conteneurs avec Docker Compose   
2. Utiliser les volumes pour partager des données entre les conteneurs    
3. Utiliser les réseaux pour connecter les conteneurs   
4. Utiliser un reverse proxy pour exposer votre application sur Internet    

## Prérequis
Avant de commencer, assurez-vous d'avoir les éléments suivants installés sur votre système :    

. Docker Desktop (si vous utilisez Windows ou macOS) ou Docker Engine (si vous utilisez Linux)    
. Docker Compose  
. Un éditeur de texte de votre choix (par exemple, Visual Studio Code)  

##Étape 1 : Créer une application multi-conteneurs avec Docker Compose
Dans cette partie, vous allez créer une application multi-conteneurs avec Docker Compose. Cette application sera basée sur une application de vote en temps réel à plusieurs salles avec Node.js et Redis.

1. Créez un nouveau répertoire pour votre application et placez-vous dedans.
2. Créez un fichier docker-compose.yml avec le contenu suivant :
