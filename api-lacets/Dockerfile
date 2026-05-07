# Étape 1 : build (installation des dépendances)
FROM node:20-alpine AS builder

WORKDIR /app

# Copier les fichiers de dépendances pour profiter du cache Docker
COPY package*.json ./

# Installer uniquement les dépendances de production
RUN npm install --only=production

# Copier le reste du code de l'API
COPY . .

# Étape 2 : image finale minimale
FROM node:20-alpine

WORKDIR /app

# Copier les fichiers depuis l'étape builder
COPY --from=builder /app /app

# Variables d'environnement par défaut
ENV NODE_ENV=production \
    PORT=3000

# Le port utilisé par l'API (configurable dans la suite via env si besoin)
EXPOSE 3000

# Commande de démarrage (définie dans package.json)
CMD ["npm", "start"]
