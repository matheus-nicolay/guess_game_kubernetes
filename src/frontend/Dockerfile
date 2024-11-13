# frontend/Dockerfile
# Etapa de build do frontend
FROM node:14 AS build

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .

ENV REACT_APP_BACKEND_URL=/api

RUN npm run build

FROM nginx:alpine

WORKDIR /usr/share/nginx/html

RUN rm -rf ./*

COPY --from=build /app/build .

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
