FROM node:20-alpine

# Set working directory
WORKDIR /app

# Copy dependency files
COPY package*.json ./

# Install production dependencies
RUN npm install --omit=dev

# Copy application source
COPY . .

# Run as non-root user
USER node

# Expose service port
ARG PORT=3000
ENV PORT=${PORT}
EXPOSE ${PORT}

# Start application
CMD ["node", "app.js"]
