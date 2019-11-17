FROM node:10.12.0-alpine

WORKDIR /app

COPY package.json package-lock.json ./

COPY . /app/

RUN npm install

CMD npm start