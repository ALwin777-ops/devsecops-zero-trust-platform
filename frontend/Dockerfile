# Stage 1: Build static assets using Node 20 LTS
FROM node:20-alpine AS build

ARG VITE_BACKEND_HOST
ENV VITE_BACKEND_HOST=$VITE_BACKEND_HOST

WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci

COPY index.html vite.config.js ./
COPY src ./src

RUN npm run build

# Stage 2: Serve using an unprivileged non-root Nginx container
FROM nginxinc/nginx-unprivileged:alpine-slim

COPY --from=build /app/dist /usr/share/nginx/html
COPY ./nginx.conf /etc/nginx/conf.d/default.conf

# Port 8080 is used by default for nginx-unprivileged (non-root users cannot bind to port 80)
EXPOSE 8080

CMD ["nginx", "-g", "daemon off;"]