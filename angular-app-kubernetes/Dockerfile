# Multi-stage Dockerfile for Angular 6 app

# 1) Build stage
FROM node:16-alpine AS build
WORKDIR /app

COPY package*.json ./
RUN npm ci --legacy-peer-deps

COPY . .
RUN npm run build -- --configuration production

# 2) Runtime stage
FROM nginx:1.27-alpine
RUN apk update && apk upgrade --no-cache \
	&& rm -rf /usr/share/nginx/html/*

# Angular SPA routing + security headers + asset caching
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=build /app/dist/angular-app /usr/share/nginx/html

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
