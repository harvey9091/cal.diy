FROM node:20-alpine AS build

ARG DATABASE_DIRECT_URL
ARG DATABASE_URL
ARG NODE_ENV

WORKDIR /calcom

RUN set -eux;
RUN ln -s /usr/lib/libssl.so.3 /lib/libssl.so.3

ENV NODE_ENV=${NODE_ENV:-production}
ENV NODE_OPTIONS="--max-old-space-size=8192"
ENV DATABASE_DIRECT_URL=${DATABASE_DIRECT_URL}
ENV DATABASE_URL=${DATABASE_URL}
ENV USE_POOL="true"

COPY . .

RUN yarn install --mode=skip-build
RUN yarn workspace @calcom/api-v2 run generate-schemas
RUN yarn workspace @calcom/api-v2 run build:docker
RUN yarn workspace @calcom/api-v2 run build

EXPOSE 80

HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
  CMD node -e "const http=require('http');http.get('http://localhost:5555/health',r=>process.exit(r.statusCode===200?0:1)).on('error',()=>process.exit(1))"

CMD [ "yarn", "workspace", "@calcom/api-v2", "start:prod"]
