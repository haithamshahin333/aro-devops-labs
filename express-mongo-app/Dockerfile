FROM node:14
USER 0
WORKDIR /usr/src/app
COPY . /usr/src/app
RUN chown -R 1001:0 /usr/src/app && \
    chmod -R 777 /usr/src/app 
RUN npm install
USER 1001
CMD "npm" "start"