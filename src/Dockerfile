FROM node:12
WORKDIR /usr/src/app
COPY . /usr/src/app
RUN npm ci --only=production
CMD "npm" "start"