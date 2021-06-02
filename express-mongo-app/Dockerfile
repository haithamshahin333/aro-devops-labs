FROM node:12
WORKDIR /usr/src/app
COPY . /usr/src/app
RUN chown -R 1001:0 /usr/src/app && \
    chmod -R 777 /usr/src/app
RUN npm ci --only=production
USER 1001
CMD "npm" "start"