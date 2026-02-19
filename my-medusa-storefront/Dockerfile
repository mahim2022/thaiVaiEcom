FROM node:20-alpine

WORKDIR /app

# Copy package manager files
COPY package.json yarn.lock .yarnrc.yml ./
COPY .yarn/releases .yarn/releases

# Install dependencies
RUN yarn install

# Copy source code
COPY . .

# Build the application
RUN yarn build

EXPOSE 8000

CMD ["yarn", "start"]

    