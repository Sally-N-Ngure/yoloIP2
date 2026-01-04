# IP2 - Dockerized E-commerce Application

For consistency, I used node:16-alpine across both Dockerfiles. This ensures a uniform base environment while maintaining the lightweight advantages of Alpine Linux.

# Client Dockerfile

To ensure stability and security, I opted for node:16-alpine—a stable Long Term Support version that receives regular updates and security patches.
Below is the Dockerfile for the Client with comments explaining each decision:

```Dockerfile
Stage 1: Build the React app
FROM node:16-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

Stage 2: Serve the built app
FROM node:16-alpine
WORKDIR /app
RUN npm install -g serve
COPY --from=builder /app/build ./build
EXPOSE 3000
CMD [ "serve", "-s", "build", "-l", "3000" ]
CMD [ "serve", "-s", "build", "-l", "3000" ]
```


During the creation of the above Dockerfile, I encountered one major challenge:

Virtual Machine Connectivity: I initially attempted to run Docker within an Ubuntu virtual machine. However, I faced persistent connectivity and performance issues that prevented the Docker daemon from functioning correctly. To overcome this obstacle and proceed with the containerization of my application, I made the decision to install Ubuntu directly on my machine. This switch to a native environment provided the stable foundation needed to successfully run Docker commands and manage my containers without the overhead and complications of the virtualized layer. I faced fewer compatibility issues with the build process in the native environment.

To build the client image, use the following command (executed from the client directory):
bash
```bash
docker build -t snngure/yolo-client:latest .
```

To run the client image, use the following command (executed from the client directory):

```bash
docker run -p 3000:3000 [image_id_or_name]
```

# Backend Dockerfile

Below is the Dockerfile for the Backend with comments to explain everything in great detail:

```Dockerfile
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
```

During the creation of the above Dockerfile, I encountered one major challenge:

The .env file: It is customary to exclude the .env file when copying the application directory to avoid exposing sensitive information in the Docker image. However, a critical issue arose when my environment variable (specifically the MONGODB_URI value) contained double quotes within the .env file. This led to errors when attempting to run the container with the --env-file .env flag, as Docker misinterpreted the environment variable value (including the quotes as part of the string), which prevented the backend from connecting to MongoDB correctly.

Solution: Ensure your .env file in the backend directory contains the connection string without quotes.

To build the backend image, use the following command :

```bash
docker build -t snngure/yolo-backend:latest .
```
To run the backend image, I used the following command :

```bash
docker run -p 5000:5000 --env-file .env [image_id_or_name]
```

# Docker Compose

To orchestrate the client, backend, and MongoDB database services—ensuring they run together, communicate effectively, and utilize persistent storage—a docker-compose.yaml file is used. This file defines a multi-container Docker application with a single configuration.


```yaml

# Use Docker Compose file format version 3.8.
version: "3.8"

# Define the multi-container application services.
services:

  # Define the frontend service, labeled 'client'.
  client:
    # The 'build' section specifies how to build the Docker image for the client.
    build:
      # 'context' tells Docker where to look for the Dockerfile and associated files.
      # Here, it will look inside the './client' directory relative to this file.
      context: ./client
      # Specifies the Dockerfile used to build the image. This allows custom naming if needed.
      dockerfile: Dockerfile
    # Names the built image. Useful for pushing to DockerHub or referencing locally.
    image: snngure/yolo-client:latest
    # Assigns a fixed container name for easier debugging and consistency.
    container_name: yolo-client
    # Maps the container's internal port 3000 (used by most React apps) to the host's port 3000.
    # This allows the developer or user to access the client app via localhost:3000.
    ports:
      - "3000:3000"
    # Specifies that the 'client' service should only start after the 'backend' is ready.
    # It does not wait for the backend to be "fully ready", just that the container is running.
    depends_on: 
      - backend
    # Connects this service to a custom Docker network called 'snnetwork'.
    networks:
      - snnetwork
    # Mounts a named volume called 'client_modules' at '/app/node_modules' inside the container.
    # This ensures node_modules is not overridden during bind-mounting the codebase in dev setups.
    volumes:
      - client_modules:/app/node_modules

  # Define the backend service, labeled 'backend'.
  backend:
    # Build configuration for the backend image.
    build:
      # Tells Docker to look inside the './backend' directory for build context.
      context: ./backend
      # Uses the specified Dockerfile inside the backend folder.
      dockerfile: Dockerfile
    # Names the Docker image for the backend. Can be pushed or reused.
    image: snngure/yolo-backend:latest
    # Assigns a fixed container name to make it easier to refer to in logs or exec commands.
    container_name: yolo-backend
    # Exposes the backend's internal port 5000 (commonly used by Express or Flask apps) to the host.
    ports:
      - "5000:5000"
    # Loads environment variables from a .env file located in the backend directory.
    # This is useful for storing sensitive values like API keys, database URLs, etc.
    env_file:
      - ./backend/.env
    # Connects this service to the same custom network as the client for inter-service communication.
    networks:
      - snnetwork
    # Mounts a named volume 'backend_modules' to persist or isolate node_modules in the container.
    volumes:
      - backend_modules:/app/node_modules

  # Add a MongoDB service for data persistence
  mongo:
    image: mongo:6.0
    container_name: yolo-mongo
    restart: unless-stopped
    ports:
      - "27017:27017"
    volumes:
      - mongo_data:/data/db
    networks:
      - snnetwork

# Define custom Docker networks used by the services.
networks:
  # Declare a bridge network named 'snnetwork' for internal communication between services.
  snnetwork:
    name: snnetwork
    # 'bridge' is the default Docker network driver for local containers.
    # It allows the services to communicate with each other using container names.
    driver: bridge

# Define persistent named volumes used by the services to avoid overwriting node_modules.
volumes:
  # Volume for the client service's node_modules directory.
  client_modules:
  # Volume for the backend service's node_modules directory.
  backend_modules:
  # Volume to persist MongoDB data between container restarts
  mongo_data:
  ```

### Explanation of Docker Compose Components
<ul> <li><strong><code>services</code>:</strong> This is the core section where you define each individual container (service) that makes up your application. Each service corresponds to a container that Docker Compose will manage.</li> <li><strong><code>build</code>:</strong> <ul> <li><code>context</code>: Specifies the path to the directory containing the <code>Dockerfile</code> and the application code needed for building the image.</li> <li><code>dockerfile</code>: Specifies the name of the <code>Dockerfile</code> if it's not the default <code>Dockerfile</code>.</li> </ul> </li> <li><strong><code>container_name</code>:</strong> Assigns a specific, human-readable name to the container instance. This makes it easier to identify and manage the container using <code>docker ps</code> or <code>docker logs</code>.</li> <li><strong><code>image</code>:</strong> Assigns a specific Docker image to be used for the service. If the image is not available locally, Docker Compose will attempt to pull it from a Docker registry (Docker Hub). If used in conjunction with build, it will tag the newly built image with the specified name.</li> <li><strong><code>ports</code>:</strong> Maps ports from your host machine to the container's exposed ports. <ul> <li>Example: <code>"5000:5000"</code> means traffic on port <code>5000</code> on your host machine will be forwarded to port <code>5000</code> inside the container.</li> </ul> </li> <li><strong><code>environment</code>:</strong> This attribute sets environment variables directly in the service configuration. <ul> <li><code>MONGO_URI=mongodb://mongo:27017/productsdb</code>: In our configuration, we set the MongoDB connection string directly. This ensures the backend container can connect to the MongoDB service by using the service name <code>mongo</code> as the hostname.</li> </ul> </li> <li><strong><code>networks</code>:</strong> This section defines which Docker networks a service will connect to. <ul> <li><code>ecommerce_net</code>: All three services (<code>mongo</code>, <code>backend</code>, and <code>client</code>) are connected to this custom network. This allows them to communicate with each other using their service names (e.g., the client can make requests to <code>http://backend:5000</code> and the backend can connect to <code>mongo:27017</code> without needing to know container IP addresses). Using a custom network provides better isolation and easier service discovery compared to Docker's default bridge network.</li> </ul> </li> <li><strong><code>volumes</code>:</strong> This attribute is used for data persistence and sharing between the host and containers, or between containers. <ul> <li><strong>Named Volumes (<code>mongo_data</code>, <code>backend_node_modules</code>, <code>client_node_modules</code>):</strong> These are Docker-managed volumes that persist data even if the containers are removed. The <code>mongo_data</code> volume ensures MongoDB data survives container restarts. The <code>*_node_modules</code> volumes cache dependencies so <code>npm ci</code> won't have to re-download all dependencies every time you rebuild or restart containers.</li> <li><strong>Bind Mounts (for development):</strong> During development, you might add bind mounts like <code>./backend:/app</code> and <code>./client:/app</code>. These directly link directories on your host machine to directories inside the container, allowing live code changes without rebuilding images.</li> </ul> </li> <li><strong><code>depends_on</code>:</strong> This specifies that a service depends on another service. Docker Compose will start the dependencies in the correct order. <ul> <li><code>depends_on: - mongo</code>: Ensures the MongoDB database is ready before the backend starts.</li> <li><code>depends_on: - backend</code>: Ensures the backend API is available before the client starts.</li> </ul> </li> <li><strong>Top-level <code>networks</code> section:</strong> This is where you define the custom networks used by your services. <ul> <li><code>ecommerce_net</code>: Declares a new network named <code>ecommerce_net</code> with the <code>bridge</code> driver. This creates a virtual network that services can join.</li> </ul> </li> <li><strong>Top-level <code>volumes</code> section:</strong> This is where you declare the named volumes that your services will use. Declaring them here makes them available for use by any service in the <code>docker-compose.yaml</code>.</li> </ul>
    
### How to Run the Application
<ol> <li><strong>Clone the Repository</strong>: First, clone the project repository to your local machine. 

```bash
git clone https://github.com/Sally-N-Ngure/yoloIP2.git cd yoloIP2
```
 
 </li> <li><strong>Navigate to Project Root</strong>: Ensure you're in the <code>yoloIP2/</code> directory where the <code>docker-compose.yaml</code> file is located.</li> <li><strong>Run Docker Compose</strong>: Execute the following command: 

 ```bash
  docker compose up --build 
  ``` 
  <ul> <li><strong>up</strong>: This command starts all services defined in your <code>docker-compose.yaml</code> file.</li> <li><strong>--build</strong>: This flag forces Docker Compose to rebuild the images for your services. This is crucial when you've made changes to your Dockerfiles or application code. You can omit this flag on subsequent runs if no changes have occurred.</li> </ul> </li> <li><strong>Access the Application</strong>: <ul> <li>Frontend: Open your browser to <code>http://localhost:3000</code></li> <li>Backend API: Access at <code>http://localhost:5000</code></li> <li>MongoDB: Available internally at <code>mongodb://localhost:27017</code> for database tools</li> </ul> </li> </ol>


### To push images to Docker Hub:

Build and tag your images:
```bash

docker compose build
docker tag yoloip2-backend snngure/yolo-backend:v1.0.0
docker tag yoloip2-client snngure/yolo-client:v1.0.0
```

Log in to Docker Hub:
```bash
docker login
```

Push the tagged images:
```bash
docker push snngure/yolo-backend:v1.0.0
docker push snngure/yolo-client:v1.0.0
```
