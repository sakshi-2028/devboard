FROM node:20-alpine

RUN addgroup -S app && adduser -S -G app app

WORKDIR /app

COPY package*.json ./

RUN npm ci

COPY . .

RUN npm run build

RUN chown -R app:app /app

EXPOSE 4173

CMD ["npm", "run", "preview", "--", "--host", "0.0.0.0", "--port", "4173"]


