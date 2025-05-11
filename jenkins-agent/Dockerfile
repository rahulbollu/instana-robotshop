FROM maven:3.9.9-eclipse-temurin-24-alpine

# Install Node.js 18 and Docker CLI
RUN curl -fsSL https://deb.nodesource.com/setup_18.x | bash - && \
    apt-get update && \
    apt-get install -y nodejs docker.io && \
    npm -v && node -v && docker -v

# Set working directory
WORKDIR /app