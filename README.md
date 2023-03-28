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
```
version: '3.9'

services:
  frontend:
    build: .
    restart: always
    ports:
      - 80:3000
    depends_on:
      - redis
    environment:
      - REDIS_URL=redis://redis:6379
  redis:
    image: redis:latest
    restart: always
volumes:
  redis:
```
Ce fichier décrit deux services : frontend et redis. Le service frontend est basé sur l'image Node.js et définit les variables d'environnement nécessaires pour se connecter à Redis. Le service redis utilise l'image Redis et n'a pas besoin de configuration supplémentaire.   

3. Créez un fichier Dockerfile avec le contenu suivant :
```FROM node:latest
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
CMD [ "npm", "start" ]
```
Ce fichier décrit la configuration du conteneur pour le service frontend.
4. Créez un fichier index.js avec le contenu suivant :
```js
const express = require('express');
const redis = require('redis');

const app = express();
const client = redis.createClient({
  host: process.env.REDIS_URL || 'localhost',
});

app.get('/:room', (req, res) => {
  const { room } = req.params;
  client.incr(room, (err, count) => {
    if (err) {
      return res.status(500).send(err.message);
    }
    res.send(`Salle ${room} : ${count}`);
  });
});

app.listen(3000, () => {
  console.log('Server started on port 3000');
});
```
