
## Construire les images
```bash
FROM python:3.12-slim

ENV PYTHONDONTWRITEBYTECODE=1 
ENV PYTHONUNBUFFERED=1

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

EXPOSE 5000

CMD ["python", "app.py"]
```

```bash
docker build -t login-backend:1.0 ./backend
docker build -t login-frontend:1.0 ./frontend
```

## Créer le réseau et le répertoire persistant

```bash
docker network create app-network
sudo mkdir -p /opt/docker-login/mysql-data:Z
```

## Démarrer la base de données

```bash
docker run -d \
  --name database \
  --network app-network \
  --restart unless-stopped \
  -e MYSQL_ROOT_PASSWORD='RootPass_ChangeMe123!' \
  -e MYSQL_DATABASE='login_db' \
  -e MYSQL_USER='appuser' \
  -e MYSQL_PASSWORD='AppPass_ChangeMe123!' \
  -v /opt/docker-login/mysql-data:/var/lib/mysql \
  mysql:8.4
```

Attendre que MySQL soit prêt :

```bash
docker logs -f database
```

Quand le journal indique que MySQL est prêt, quittez l'affichage avec `Ctrl+C`. Cela ne stoppe pas le conteneur.

## Démarrer le backend

```bash
docker run -d \
  --name backend \
  --network app-network \
  --restart unless-stopped \
  -e DB_HOST='database' \
  -e DB_PORT='3306' \
  -e DB_NAME='login_db' \
  -e DB_USER='appuser' \
  -e DB_PASSWORD='AppPass_ChangeMe123!' \
  -e ADMIN_USERNAME='student' \
  -e ADMIN_PASSWORD='DockerLab123!' \
  login-backend:1.0
```

Le backend attend que MySQL soit joignable et crée automatiquement la table `users` ainsi qu'un compte de démonstration si celui-ci n'existe pas.

## Démarrer le frontend

```bash
docker run -d \
  --name frontend \
  --network app-network \
  --restart unless-stopped \
  -p 8080:80 \
  login-frontend:1.0
```

# Vérifications

### Conteneurs et images

```bash
docker ps
docker images
```

### Réseau partagé

```bash
docker network inspect app-network
```


Cette commande affiche les variables, y compris le mot de passe de démonstration. À utiliser uniquement dans cet environnement de laboratoire.

### Journaux

```bash
docker logs backend
docker logs database
docker logs frontend
```

### Tester l'API à travers le frontend

```bash
curl -i -X POST http://localhost:8080/api/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"student","password":"DockerLab123!"}'
```
Une réponse JSON avec `"success": true` est attendue.

### Tester l'Application dans le navigateur
http://localhost:8080

Utilisateur : `student`
Mot de passe : `DockerLab123!`

### Vérifier la persistance

1. Se connecter à l'application.
2. Supprimer puis recréer le conteneur MySQL en conservant le répertoire monté.
3. Vérifier que la table et l'utilisateur existent toujours.

Pour un test simple, redémarrez d'abord la base :

```bash
docker restart database
```

Pour tester réellement la recréation du conteneur, notez que supprimer le conteneur ne supprime pas le bind mount :

```bash
docker rm -f database
docker run -d \
  --name database \
  --network app-network \
  --restart unless-stopped \
  -e MYSQL_ROOT_PASSWORD='RootPass_ChangeMe123!' \
  -e MYSQL_DATABASE='login_db' \
  -e MYSQL_USER='appuser' \
  -e MYSQL_PASSWORD='AppPass_ChangeMe123!' \
  -v /opt/docker-login/mysql-data:/var/lib/mysql \
  mysql:8.4
```

Attendez que MySQL redémarre, puis vérifiez la connexion au site. Quand un répertoire de données MySQL déjà initialisé est monté, les variables d'initialisation ne recréent pas la base : les données existantes sont utilisées.
