# Этап 1: Builder (Сборщик)
FROM node:20-alpine AS builder

RUN corepack enable pnpm

WORKDIR /app

# Копируем конфигурационные файлы package.json, pnpm-lock.yaml и pnpm-workspace.yaml
# Это позволяет закэшировать этап установки зависимостей
COPY package.json pnpm-lock.yaml pnpm-workspace.yaml ./

# Устанавливаем все зависимости (включая devDependencies для сборки) строго по lock-файлу
RUN pnpm install --frozen-lockfile

# Копируем весь остальной исходный код фронтенда
COPY . .

# Собираем SvelteKit приложение для продакшена (создаст папку build/)
RUN pnpm run build

# Убираем devDependencies, оставляем только пакеты нужные для запуска
RUN CI=true pnpm prune --prod

# Этап 2: Runner (Легковесный финальный образ)
FROM node:20-alpine

# SvelteKit по умолчанию запускает сервер в production на порту 3000
ENV NODE_ENV=production \
    PORT=3000 \
    HOST=0.0.0.0

WORKDIR /app

# Копируем только самое необходимое из первого этапа
COPY --from=builder /app/build ./build
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./package.json

# В официальных образах Node.js уже есть безопасный пользователь 'node'
# Меняем владельца файлов на этого пользователя
RUN chown -R node:node /app

# Переключаемся на пользователя без прав root
USER node

EXPOSE 3000

# Запускаем скомпилированный Node.js сервер
CMD ["node", "build/index.js"]