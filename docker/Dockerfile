# Pre-build stage
FROM ruby:3.2.2-alpine3.19 AS pre-builder

# ARG e ENV para configurações
ARG BUNDLE_WITHOUT="development:test"
ENV BUNDLE_WITHOUT=${BUNDLE_WITHOUT}
ENV BUNDLER_VERSION=2.5.11

ARG RAILS_ENV=production
ENV RAILS_ENV=${RAILS_ENV}

ARG NODE_OPTIONS="--openssl-legacy-provider"
ENV NODE_OPTIONS=${NODE_OPTIONS}

ENV BUNDLE_PATH="/gems"

# Instalação de dependências
RUN apk update && apk add --no-cache \
  openssl \
  tar \
  build-base \
  tzdata \
  postgresql-dev \
  postgresql-client \
  nodejs \
  yarn \
  git \
  ruby-dev \
  linux-headers \
  g++ \
  && gem install bundler -v ${BUNDLER_VERSION}

WORKDIR /app

# Copiar Gemfile e instalar gems
COPY Gemfile Gemfile.lock ./

# Configuração para forçar Ruby em Alpine
RUN apk add --no-cache build-base musl ruby-full musl-dev openssl-dev xz vips \
  && bundle config set --local force_ruby_platform true \
  && bundle install -j4 --retry 3 --without development test

# Instalar dependências JS
COPY package.json yarn.lock ./
RUN yarn install

# Copiar restante da aplicação
COPY . /app

# Criar diretório de logs
RUN mkdir -p /app/log

# Gerar assets de produção
RUN if [ "$RAILS_ENV" = "production" ]; then \
  SECRET_KEY_BASE=precompile_placeholder RAILS_LOG_TO_STDOUT=enabled \
  bundle exec rake assets:precompile \
  && rm -rf spec node_modules tmp/cache; \
  fi

# Gerar arquivo .git_sha com hash do commit
RUN git rev-parse HEAD > /app/.git_sha

# Limpeza de arquivos desnecessários
RUN rm -rf /gems/ruby/3.2.0/cache/*.gem \
  && find /gems/ruby/3.2.0/gems/ \( -name "*.c" -o -name "*.o" \) -delete \
  && rm -rf .git \
  && rm .gitignore

# Final build stage
FROM ruby:3.2.2-alpine3.19

# ARG e ENV para configurações
ARG BUNDLE_WITHOUT="development:test"
ENV BUNDLE_WITHOUT=${BUNDLE_WITHOUT}
ENV BUNDLER_VERSION=2.5.11

ARG EXECJS_RUNTIME="Disabled"
ENV EXECJS_RUNTIME=${EXECJS_RUNTIME}

ARG RAILS_ENV=production
ENV RAILS_ENV=${RAILS_ENV}
ENV BUNDLE_PATH="/gems"

# Instalação de dependências necessárias
RUN apk update && apk add --no-cache \
  build-base \
  openssl \
  tzdata \
  postgresql-client \
  imagemagick \
  git \
  vips \
  && gem install bundler -v ${BUNDLER_VERSION}

# Copiar arquivos do pre-builder
COPY --from=pre-builder /gems/ /gems/
COPY --from=pre-builder /app /app
COPY --from=pre-builder /app/.git_sha /app/.git_sha

WORKDIR /app

# Permissões necessárias
RUN chmod -R 777 ./docker

# Expor porta
EXPOSE 3000

CMD ["bundle", "exec", "rails", "server", "-b", "0.0.0.0", "-p", "3000"]
