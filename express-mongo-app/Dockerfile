FROM node:14
USER 0
WORKDIR /usr/src/app
COPY . /usr/src/app
RUN chown -R 1001:0 /usr/src/app && \
    chmod -R 777 /usr/src/app 
USER 1001
RUN npm install
CMD "npm" "start"