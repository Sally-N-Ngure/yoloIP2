# Start from the official lightweight Node.js 16 image based on Alpine Linux
FROM node:16-alpine

# Set the working directory inside the container to /app
WORKDIR /app

# Copy only package.json and package-lock.json into the container 
# This allows Docker to cache installed dependencies between builds if package files don’t change
COPY package*.json ./

# Install production dependencies exactly as specified in package-lock.json
RUN npm ci

# Copy the rest of the application source code into the container, this includes server.js and any other app files
COPY . .

# Expose port 5000 to the host machine so it knows which port the backend is listening on
EXPOSE 5000

# start the Node.js backend server by running server.js
CMD ["node", "server.js"]