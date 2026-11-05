FROM node:20-alpine AS builder

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .

FROM node:20-alpine

WORKDIR /app

RUN addgroup -S app && adduser -S app -G app

COPY --from=builder --chown=app:app /app/node_modules ./node_modules
COPY --from=builder /app/package*.json ./
COPY --from=builder --chown=app:app /app/src ./src
COPY --from=builder --chown=app:app /app/public ./public
COPY --from=builder --chown=app:app /app/index.html ./
COPY --from=builder --chown=app:app /app/vite.config.ts ./
COPY --from=builder --chown=app:app /app/tsconfig*.json ./
COPY --from=builder --chown=app:app /app/eslint.config.js ./

ARG API_URL=
ENV VITE_API_URL=$API_URL

USER app

EXPOSE 5173

CMD ["npm", "run", "dev", "--", "--host", "0.0.0.0"]
