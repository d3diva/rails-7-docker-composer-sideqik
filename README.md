# README

To run a Rails 7 in Docker with Sidekiq

1. Create a Dockerfile

```
FROM ruby:3.1.1

RUN curl https://deb.nodesource.com/setup_12.x | bash
RUN curl https://dl.yarnpkg.com/debian/pubkey.gpg | apt-key add -
RUN echo "deb https://dl.yarnpkg.com/debian/ stable main" | tee /etc/apt/sources.list.d/yarn.list

RUN apt-get update && apt-get install -y nodejs yarn postgresql-client
RUN yarn add jquery
RUN yarn add github:nathanvda/cocoon#c24ba53
RUN yarn add @popperjs/core


WORKDIR /myapp
COPY Gemfile /myapp/Gemfile
COPY Gemfile.lock /myapp/Gemfile.lock
RUN bundle install

# Add a script to be executed every time the container starts.
COPY entrypoint.sh /usr/bin/
RUN chmod +x /usr/bin/entrypoint.sh
ENTRYPOINT ["entrypoint.sh"]
EXPOSE 3000

# Configure the main process to run when running the image
CMD ["rails", "server", "-b", "0.0.0.0"]

```

2. Create a Gemfile

```
source "https://rubygems.org"
git_source(:github) { |repo| "https://github.com/#{repo}.git" }

ruby "3.1.1"

# Bundle edge Rails instead: gem "rails", github: "rails/rails", branch: "main"
gem "rails", "~> 7.0.2", ">= 7.0.2.2"

```

3. Create a Gemfile.lock

`touch Gemfile.lock`

4. Create a docker-compose.yml file

```
version: "3.9"
services:
  db:
    image: postgres
    volumes:
      - ./tmp/db:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD: password
  web:
    build: .
    command: bash -c "rm -f tmp/pids/server.pid && bundle exec rails s -p 3000 -b '0.0.0.0'"
    volumes:
      - .:/myapp
    
    ports:
      - "3000:3000"
    depends_on:
      - db
  
```

5. Generate the project

`docker-compose run web rails new . --force --database=postgresql`

6. Build the containers

`docker-compose build`

7. User permission

`sudo chown -R $USER:$USER .`

8. Update the database config/database.yml

```
default: &default
  adapter: postgresql
  encoding: unicode
  host: db
  username: postgres
  password:
  pool: 5

development:
  <<: *default
  database: myapp_development


test:
  <<: *default
  database: myapp_test
```

9. Create the database

`docker-compose run web rake db:create`

10. Add sidekiq to Gemfile

`gem 'sidekiq'`

11. Update Gemfile.lock

`docker-compose run web bundle install`

12. Update docker-compose.yml 

```
version: "3.9"
services:
  db:
    image: postgres
    volumes:
      - ./tmp/db:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD: password
  web:
    build: .
    command: bash -c "rm -f tmp/pids/server.pid && bundle exec rails s -p 3000 -b '0.0.0.0'"
    volumes:
      - .:/myapp
    environment:
      REDIS_URL: "redis://redis:6379/12"
    ports:
      - "3000:3000"
    depends_on:
      - db
  redis:
    image: redis
    volumes:
      - ./tmp/db:/var/lib/redis/data
  sidekiq:
    build: .
    command: 'bundle exec sidekiq'
    volumes:
      - .:/myapp
    environment:
      REDIS_URL: "redis://redis:6379/12"
    depends_on:
      - redis

```

13. Build the containers

`docker-compose build`

14. Start Docker

 `docker-compose up` 

14. Rails 7 starting page

 `localhost:3000` 