FROM node:20-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci --omit=dev
RUN apk add --no-cache wget

COPY . .

ENV NODE_ENV=production
EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD wget -qO- http://localhost:8080/api/tutorials || exit 1

CMD ["node", "server.js"]
