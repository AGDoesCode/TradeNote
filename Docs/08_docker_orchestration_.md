# Chapter 8: Docker Orchestration

In [Chapter 7: Vue Application Setup](07_vue_application_setup_.md), we saw how the TradeNote frontend application gets assembled and started in your browser. That covers the user interface part. But TradeNote isn't just a frontend app; it also needs a database (MongoDB) to store all your trades, diary entries, tags, and user information. How do we make sure both the TradeNote application *and* its database are set up correctly and can talk to each other, especially if you want to run it on your own computer?

**What Problem Does This Solve?**

Imagine you're setting up a new kitchen. You need the oven (the TradeNote app) and the refrigerator (the MongoDB database). You also need to make sure they are plugged in correctly (networking) and that the refrigerator keeps your food cold even if you turn the oven off and on (data persistence). Doing this setup manually on different computers could be tricky – maybe one computer has an older version of the database software, or the network settings are different. Things might not work consistently.

**Docker Orchestration** solves this problem. It provides a single, simple "recipe" and set of instructions (`docker-compose.yml` file) that tells a tool called Docker exactly how to:
1.  Set up the TradeNote application in its own isolated environment (a "container").
2.  Set up the MongoDB database in its *own* isolated environment.
3.  Configure them with the correct settings (like network addresses, ports, usernames/passwords).
4.  Connect them so they can communicate.
5.  Ensure your data (like trades in the database) is saved safely even if you stop and restart the application.

Think of it like a complete blueprint for assembling and connecting your kitchen appliances, ensuring the setup is identical and works reliably everywhere – on your laptop, a friend's computer, or a server.

**Core Concepts**

1.  **Docker:** A tool that lets you package applications and their dependencies (like databases or specific software versions) into standardized units called **containers**.
2.  **Container:** An isolated, self-contained environment where an application runs. It's like a sealed box containing the app and everything it needs, ensuring it doesn't interfere with other software on your computer, and vice-versa. TradeNote runs in one container, and MongoDB runs in another.
3.  **Image:** A blueprint or template used to create a container. It contains the application code, libraries, and configuration. Think of it as the recipe for making the "sealed box". TradeNote has its own image, and MongoDB has a standard official image.
4.  **Docker Compose:** A tool for defining and running *multi-container* Docker applications. It reads a configuration file (usually `docker-compose.yml`) that describes all the services (containers), networks, and volumes needed for the application. It's the master instruction manual for setting up the whole kitchen (app + database).
5.  **Service:** A specific part of the application defined in the Docker Compose file. In TradeNote, we have two main services: `tradenote` (the application itself) and `mongo` (the database).
6.  **Volume:** A way to store data *outside* the container, making it persistent. When you stop or remove a container, its internal storage is usually lost. Volumes link a folder inside the container (like MongoDB's data directory) to a special location managed by Docker on your host computer. This ensures your database files survive even if the `mongo` container is restarted. It's like keeping your ingredients in a real refrigerator, not just a temporary cooler box (the container).
7.  **Network:** Docker Compose automatically creates a private network connecting the containers defined in the file. This allows the `tradenote` container to talk to the `mongo` container using simple names (like `mongo`) instead of complex IP addresses. It's the wiring connecting the appliances.
8.  **Environment Variables:** Settings passed into a container when it starts. Used to configure things like database connection details (`MONGO_URI`), application ports (`TRADENOTE_PORT`), and security keys (`APP_ID`, `MASTER_KEY`) without changing the container image itself. It's like setting the temperature dial on the oven.

**How TradeNote Uses Docker Orchestration**

TradeNote provides configuration files (`docker-compose.yml` and `docker-compose-local.yml`) that define everything needed to run the application and its database together using Docker Compose.

**Using Docker Compose**

The simplest way to get TradeNote running locally is described in the main `README.md`:

1.  **Install:** Make sure you have Docker and Docker Compose installed on your computer.
2.  **Download:** Get the `docker-compose.yml` file (or `docker-compose-local.yml` if you want to build the TradeNote image yourself from the source code).
3.  **Run:** Open a terminal or command prompt, navigate to the folder where you saved the file, and run the command:
    ```bash
    docker compose up -d
    ```
    *   `docker compose`: Tells Docker to use the Compose tool.
    *   `up`: Reads the `docker-compose.yml` file in the current directory and starts (or creates and starts) all the defined services.
    *   `-d`: Runs the containers in "detached" mode (in the background), so your terminal is free.

That's it! Docker Compose reads the file and automatically:
*   Downloads the necessary images (MongoDB and, for `docker-compose.yml`, the pre-built TradeNote image). Or, for `docker-compose-local.yml`, it builds the TradeNote image first using instructions in `docker/Dockerfile`.
*   Creates a persistent volume for the database data.
*   Creates a network for the services to communicate.
*   Starts the `mongo` container.
*   Starts the `tradenote` container, injecting the required environment variables so it knows how to connect to the `mongo` container.
*   Maps the port `8080` from the `tradenote` container to port `8080` on your computer, so you can access the app in your browser at `http://localhost:8080`.

**Exploring `docker-compose-local.yml`**

Let's look at some key parts of the `docker-compose-local.yml` file (which is very similar to `docker-compose.yml` but builds the app locally):

```yaml
# docker-compose-local.yml (Simplified Snippets)

# Define the services (containers) needed
services:
  # The TradeNote application service
  tradenote:
    container_name: tradenote_app # Give the container a specific name
    # How to build the image for this container:
    build:
      context: . # Use the current directory as the build context
      dockerfile: ./docker/Dockerfile # Specify the recipe file
    image: tradenote:latest # Name the built image
    ports:
      - 8080:8080 # Map host port 8080 to container port 8080
    links: # (Older way to link, networks are preferred now)
      - mongo
    # Configuration settings passed into the container
    environment:
      MONGO_URI: mongodb://tradenote:tradenote@mongo:27017/tradenote?authSource=admin
      TRADENOTE_DATABASE: tradenote
      APP_ID: 123456 # Should change these for security
      MASTER_KEY: 123456 # Should change these for security
      TRADENOTE_PORT: 8080

  # The MongoDB database service
  mongo:
    image: mongo:latest # Use the official MongoDB image from Docker Hub
    container_name: tradenote_db
    # Define where to store persistent data
    volumes:
      - tradenote_db:/data/db # Map 'tradenote_db' volume to MongoDB's data dir
    ports:
      - 27017:27017 # Map host port 27017 to container port 27017 (optional)
    # Configuration for initializing the database
    environment:
      MONGO_INITDB_ROOT_USERNAME: tradenote
      MONGO_INITDB_ROOT_PASSWORD: tradenote
      MONGO_INITDB_DATABASE: tradenote

# Define the persistent volume for the database
volumes:
  tradenote_db:
    name: tradenote_db # Give the volume a specific name
```

*   **`services:`**: Defines the containers to run. We have `tradenote` and `mongo`.
*   **`tradenote.build`**: Tells Docker Compose to build the `tradenote` image using the `Dockerfile` in the `docker` folder, instead of downloading a pre-built one. (`docker-compose.yml` uses `image: eleventrading/tradenote` here instead).
*   **`tradenote.ports`**: Makes the TradeNote app accessible on your computer's port 8080.
*   **`tradenote.environment`**: Sets crucial configuration. Notice `MONGO_URI` uses `mongo:27017` – `mongo` is the service name of the database container, which Docker Compose makes resolvable within the internal network.
*   **`mongo.image`**: Pulls the standard MongoDB image.
*   **`mongo.volumes`**: This is key for data persistence. It links the `/data/db` folder *inside* the `mongo` container (where MongoDB stores its data) to a Docker-managed volume named `tradenote_db` on your host machine.
*   **`mongo.environment`**: Configures the initial username, password, and database name for MongoDB when it starts for the first time.
*   **`volumes:`**: Declares the named volume `tradenote_db` so Docker can manage it.

**Internal Implementation: Under the Hood**

What actually happens when you type `docker compose up -d`?

**Step-by-Step Walkthrough**

1.  **Read File:** Docker Compose reads the `docker-compose.yml` (or `docker-compose-local.yml`) in the current directory.
2.  **Check Network:** It checks if a default network for this project exists. If not, it creates one (e.g., `tradenote_tradenote_net`).
3.  **Check Volume:** It checks if the named volume `tradenote_db` exists. If not, it creates it.
4.  **Check Images:**
    *   For `mongo`: Checks if you have the `mongo:latest` image locally. If not, it downloads it from Docker Hub.
    *   For `tradenote` (using `docker-compose-local.yml`): It runs the Docker build process using `docker/Dockerfile` to create the `tradenote:latest` image locally. (If using `docker-compose.yml`, it checks for `eleventrading/tradenote` locally and downloads if needed).
5.  **Start Dependencies:** It sees that `tradenote` depends on `mongo` (implicitly via the network connection in `MONGO_URI`). It starts the `mongo` container first.
    *   The `mongo` container starts.
    *   Its internal `/data/db` directory is linked to the `tradenote_db` volume.
    *   The environment variables are used to set up the initial user/password/database (if the volume is empty/new).
    *   It joins the project network.
6.  **Start Application:** It starts the `tradenote` container.
    *   The specified environment variables (like `MONGO_URI`, `APP_ID`, etc.) are injected into the container.
    *   Port 8080 inside the container is mapped to port 8080 on your host machine.
    *   It joins the project network, allowing it to reach the `mongo` service at the hostname `mongo`.
    *   The application inside the container starts up, connects to MongoDB using the provided `MONGO_URI`, and begins listening for web requests on port 8080.
7.  **Ready:** Both containers are running in the background. You can now access TradeNote at `http://localhost:8080`.

**Sequence Diagram**

```mermaid
sequenceDiagram
    participant User
    participant ComposeCLI as Docker Compose CLI
    participant DockerEngine as Docker Engine
    participant DockerHub as Docker Hub (or Local Cache)
    participant Network as Docker Network
    participant Volume as Docker Volume (tradenote_db)
    participant MongoContainer as Mongo Container
    participant AppContainer as TradeNote Container

    User->>ComposeCLI: docker compose up -d
    ComposeCLI->>DockerEngine: Parse docker-compose.yml
    DockerEngine->>Network: Create Network (if needed)
    DockerEngine->>Volume: Create Volume (if needed)
    DockerEngine->>DockerHub: Pull/Check mongo:latest Image
    DockerEngine->>DockerHub: Build/Pull tradenote Image
    DockerEngine->>MongoContainer: Create & Start Container
    MongoContainer->>Volume: Mount /data/db
    MongoContainer->>Network: Join Network
    DockerEngine->>AppContainer: Create & Start Container (with Env Vars)
    AppContainer->>Network: Join Network
    AppContainer->>MongoContainer: Connect via Network (using 'mongo' hostname)
    AppContainer->>User: Expose Port 8080
```

This orchestration ensures that both the TradeNote application and its necessary database are configured correctly, connected, and running with persistent data storage, all initiated by a single command. It makes setting up the development or deployment environment incredibly simple and consistent.

**Conclusion**

You've reached the end of our TradeNote tutorial series and learned about Docker Orchestration!

*   You understand **why** it's useful: for easy, consistent setup of the application and its database across different environments.
*   You know the key concepts: **Docker**, **Containers**, **Images**, **Docker Compose**, **Services** (`tradenote`, `mongo`), **Volumes** (for data persistence), **Networks** (for communication), and **Environment Variables** (for configuration).
*   You learned how a simple command (`docker compose up -d`) uses the `docker-compose.yml` file to automate the entire setup process.
*   You saw how the configuration file defines each service, how they connect (`MONGO_URI` using the service name `mongo`), and how data is persisted (`volumes`).

Docker orchestration provides the final piece of the puzzle, packaging the entire TradeNote system – the frontend built with Vue ([Chapter 7: Vue Application Setup](07_vue_application_setup_.md)), the data processing logic ([Chapter 4: Trade Data Processing & Aggregation](04_trade_data_processing___aggregation_.md)), the database interaction ([Chapter 6: Application Utilities & Core Logic](06_application_utilities___core_logic_.md)), and the database itself – into a easily deployable and runnable unit.

Congratulations on completing the TradeNote tutorial series! You now have a solid overview of how the different parts of the application work together, from navigating the UI to processing your trade data and running the entire stack locally using Docker.

---

Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge)