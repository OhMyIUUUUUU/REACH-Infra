FROM node:24-alpine AS build

WORKDIR /app
COPY reach/Frontend/package*.json ./
RUN npm ci
COPY reach/Frontend/ ./
RUN npm run build

FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
COPY deploy/frontend/default.conf.template /etc/nginx/templates/default.conf.template
ENV NGINX_ENVSUBST_FILTER=^(PORT|BACKEND_URL)$
EXPOSE 8080
