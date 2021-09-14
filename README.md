# Rails com MySql rodando em Docker

Este projeto é um projeto modelo.

## Criando novo projeto:

- Inicializando novo projeto rails com bd MySql:

  ```sh
  rails new rails-6-docker-mysql-8 --database=mysql -T
  ```

- Crie um dockerignore:

  ```sh
  touch .dockerignore
  ```

- Em seguida, acesse https://gist.github.com/yizeng/eeeb48d6823801061791cc5581f7e1fc , copie o conteúdo e adicione ao aquivo

- Crie um Dockerfile:

  ```sh
  touch Dockerfile
  ```

- Em seguida, copie conteúdo abaixo e adicione ao arquivo

  ```dockerfile
  FROM ruby:3

  ENV LANG C.UTF-8
  ENV APP_ROOT /app

  RUN curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | apt-key add - && \
  echo "deb https://dl.yarnpkg.com/debian/ stable main" | tee /etc/apt/sources.list.d/yarn.list && \
  apt-get update -qq && \
  apt-get install -y --no-install-recommends \
  build-essential \
  nodejs \
  yarn && \
  apt-get clean && \
  rm --recursive --force /var/lib/apt/lists/*

  RUN mkdir $APP_ROOT
  WORKDIR $APP_ROOT

  COPY Gemfile $APP_ROOT/Gemfile
  COPY Gemfile.lock $APP_ROOT/Gemfile.lock
  RUN bundle install --jobs 4 --retry 3

  COPY . $APP_ROOT

  COPY entrypoint.sh /usr/bin/
  RUN chmod +x /usr/bin/entrypoint.sh
  ENTRYPOINT ["entrypoint.sh"]
  EXPOSE 3000

  CMD ["bundle", "exec", "rails", "server", "-b", "0.0.0.0"]
  ```

- Crie um docker-compose:

  ```sh
  touch docker-compose.yml
  ```

- Em seguida, copie conteúdo abaixo e adicione ao arquivo

  ```yaml
  version: "3"

  services:
  db:
    image: "mysql:8"
    ports:
      - "3306:3306"
    restart: always
    volumes:
      - mysql:/var/lib/mysql:delegated
    environment:
    MYSQL_ROOT_PASSWORD: root
    command: --default-authentication-plugin=mysql_native_password

  web:
    depends_on:
      - "db"
    build: .
    command: bash -c "rm -f tmp/pids/server.pid && bundle exec rails s -p 3000 -b '0.0.0.0'"
    ports:
      - "3000:3000"
    environment:
      - DATABASE_HOST=db
    volumes:
      - .:/app:cached
      - bundle:/usr/local/bundle:delegated
      - node_modules:/app/node_modules
      - tmp-data:/app/tmp/sockets

  volumes:
    mysql:
    bundle:
    node_modules:
    tmp-data:
  ```

- Crie um entrypoint:

  ```sh
  touch entrypoint.sh
  ```

- Em seguida, copie conteúdo abaixo e adicione ao arquivo

  ```sh
    #!/bin/bash
    set -e

    # Remove a potentially pre-existing server.pid for Rails.
    rm -f /myapp/tmp/pids/server.pid

    # Then exec the container's main process (what's set as CMD in the Dockerfile).
    exec "$@"
  ```

- Execute:

  ```sh
  docker-compose run --rm web bin/rails webpacker:install
  ```

- Em seguida, copie conteúdo abaixo e adicione ao arquivo config/database.yml

  ```yaml
  default: &default
    adapter: mysql2
    encoding: utf8mb4
    pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
    username: <%= ENV.fetch('MYSQL_USERNAME') { 'root' } %>
    password: <%= ENV.fetch('MYSQL_PASSWORD') { 'password' } %>
    host: <%= ENV.fetch('MYSQL_HOST') { 'db' } %>

  development:
    <<: *default
    database: rails_app_dev

  test:
    <<: *default
    database: rails_app_test

  production:
    <<: *default
    database: rails_app_prd
    username: app
    password: hoge
  ```

- Execute:

  ```sh
  docker-compose build
  docker-compose up -d
  docker-compose run web bin/rails db:create
  ```
