FROM node:22-alpine

WORKDIR /app

# Copy package files first for layer caching
COPY package*.json ./

# Install dependencies (warms the named volume on first run)
RUN npm install

# Copy the rest of the source
COPY . .

EXPOSE 5173

CMD ["npm", "run", "dev"]