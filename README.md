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

## Étape 1 : Créer une application multi-conteneurs avec Docker Compose
Dans cette partie, vous allez créer une application multi-conteneurs avec Docker Compose. Cette application sera basée sur une application de vote en temps réel à plusieurs salles avec Node.js et Redis.

1. Créez un nouveau répertoire pour votre application et placez-vous dedans.
2. Créez un fichier docker-compose.yml avec le contenu suivant :
```yml
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
```Dockerfile
FROM node:latest
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
Ce fichier décrit la logique de l'application de vote en temps réel. Lorsqu'un utilisateur visite une URL avec le numéro d'une salle, le serveur incrémente le compteur pour cette salle et renvoie le nombre actuel de votes pour cette salle.     
5. Exécutez la commande suivante pour démarrer les conteneurs :
```
docker-compose up -d
```
6. Vérifiez que les conteneurs sont en cours d'exécution avec la commande suivante :      
```
docker-compose ps
```
Vous devriez voir les deux conteneurs.    
7. Testez l'application en accédant à l'URL http://localhost. Vous devriez voir une page avec des boutons pour voter pour les salles 1, 2, 3 et 4. Chaque fois que vous cliquez sur un bouton, le nombre de votes pour cette salle devrait augmenter.
## Étape 2 : Utiliser les volumes pour partager des données entre les conteneurs
Dans cette partie, vous allez utiliser les volumes pour partager des données entre les conteneurs. Vous allez ajouter une base de données MySQL pour stocker les résultats des votes.     
1. Modifiez le fichier docker-compose.yml comme suit :    
```yml
version: '3.9'

services:
  frontend:
    build: .
    restart: always
    ports:
      - 80:3000
    depends_on:
      - redis
      - db
    environment:
      - REDIS_URL=redis://redis:6379
      - MYSQL_HOST=db
      - MYSQL_USER=root
      - MYSQL_PASSWORD=secret
      - MYSQL_DATABASE=votes
  redis:
    image: redis:latest
    restart: always
  db:
    image: mysql:latest
    restart: always
    environment:
      - MYSQL_ROOT_PASSWORD=secret
      - MYSQL_DATABASE=votes
    volumes:
      - mysql:/var/lib/mysql
volumes:
  redis:
  mysql:
```
Ce fichier ajoute un nouveau service db qui utilise l'image MySQL. Il définit également les variables d'environnement nécessaires pour se connecter à la base de données.     

Le volume mysql est utilisé pour stocker les données de la base de données.     
2. Créez un fichier docker-entrypoint.sh avec le contenu suivant :      
```sh
#!/bin/bash

set -e

if [ ! -d /var/lib/mysql/mysql ]; then
  mysql_install_db --user=mysql --datadir=/var/lib/mysql
  mysqld --user=mysql --datadir=/var/lib/mysql --skip-networking &
  mysql -u root -p"$MYSQL_ROOT_PASSWORD" -e "CREATE DATABASE $MYSQL_DATABASE;"
  mysql -u root -p"$MYSQL_ROOT_PASSWORD" -e "CREATE USER '$MYSQL_USER'@'%' IDENTIFIED BY '$MYSQL_PASSWORD';"
  mysql -u root -p"$MYSQL_ROOT_PASSWORD" -e "GRANT ALL PRIVILEGES ON $MYSQL_DATABASE.* TO '$MYSQL_USER'@'%';"
  mysqladmin -u root -p"$MYSQL_ROOT_PASSWORD" shutdown
fi

exec "$@"
```
Ce script sera utilisé comme point d'entrée pour le conteneur de la base de données MySQL. Il installe et configure la base de données si elle n'existe pas déjà.     
3. Modifiez le fichier docker-compose.yml pour ajouter les lignes suivantes au service db :     
```yml
command: ["bash", "/docker-entrypoint.sh", "mysqld"]
entrypoint: ["/usr/local/bin/docker-entrypoint.sh"]
```
Ces lignes configurent le script docker-entrypoint.sh comme point d'entrée pour le conteneur de la base de données MySQL.     
4. Modifiez le fichier index.js pour ajouter le code suivant :    
```js
const mysql = require('mysql');

const connection = mysql.createConnection({
  host: process.env.MYSQL_HOST || 'localhost',
  user: process.env.MYSQL_USER || 'root',
  password: process.env.MYSQL_PASSWORD || '',
  database: process.env.MYSQL_DATABASE || 'votes',
});

connection.connect((err) => {
  if (err) {
    console.error('Error connecting to MySQL database', err);
  } else {
    console.log('Connected to MySQL database');
  }
});

app.get('/results', (req, res) => {
  connection.query('SELECT * FROM votes', (err, rows) => {
    if (err) {
      console.error('Error getting votes from MySQL database', err);
      res.status(500).send('Internal Server Error');
    } else {
      const results = {};
      rows.forEach((row) => {
        results[row.id] = row.votes;
      });
      res.json(results);
    }
  });
});

app.post('/vote', (req, res) => {
  const roomId = req.body.roomId;
  if (!roomId || !VALID_ROOM_IDS.includes(roomId)) {
    res.status(400).send('Invalid room ID');
  } else {
    connection.query('UPDATE votes SET votes = votes + 1 WHERE id = ?', [roomId], (err, result) => {
      if (err) {
        console.error('Error updating votes in MySQL database', err);
        res.status(500).send('Internal Server Error');
      } else {
        console.log(`Vote added for room ${roomId}`);
        res.send('OK');
      }
    });
  }
});

const server = app.listen(process.env.PORT || 3000, () => {
  console.log(`Server listening on port ${server.address().port}`);
});

```
Ce code crée une connexion à la base de données MySQL et définit les routes /results et /vote pour récupérer les résultats des votes et ajouter un vote, respectivement.      
5. Construisez et lancez les conteneurs en exécutant la commande suivante :     
```
docker-compose up
```
6. Testez l'application en accédant à l'URL http://localhost. Vous devriez voir une page avec des boutons pour voter pour les salles 1, 2, 3 et 4. Chaque fois que vous cliquez sur un bouton, le nombre de votes pour cette salle devrait augmenter.   
7. Testez également la route /results en accédant à l'URL http://localhost/results. Vous devriez voir un objet JSON avec les résultats des votes pour chaque salle.   
## Conclusion
Dans ce TP, vous avez appris à utiliser Docker et Docker Compose pour créer une application Node.js avec une base de données Redis et MySQL. Vous avez également appris à utiliser les volumes pour partager des données entre les conteneurs.    

Il existe de nombreuses autres fonctionnalités avancées dans Docker et Docker Compose, comme les réseaux, les variables d'environnement, les fichiers de configuration, les mises à l'échelle, etc. Vous pouvez continuer à explorer ces fonctionnalités pour créer des applications plus complexes et plus avancées.   
