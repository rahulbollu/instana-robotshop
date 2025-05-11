FROM maven:3.9.3-eclipse-temurin-17

# Install Node.js 18 and Docker
RUN apt-get update && \
    apt-get install -y curl gnupg && \
    curl -fsSL https://deb.nodesource.com/setup_18.x | bash - && \
    apt-get install -y nodejs docker.io && \
    npm -v && node -v && docker -v

# Set working directory
WORKDIR /app