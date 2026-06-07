# Dockerfile Explanation (Enterprise Standard)

This document breaks down every line of our `Dockerfile` and explains *why* the industry standard dictates this specific approach.

## Multi-Stage Builds
We use a **Multi-Stage Build** (two `FROM` statements).
* **Why not a single stage?** If we used one stage, the final image sent to production would contain Maven, the source code, and the entire Java Development Kit (JDK). This results in a massive image (~600MB+) with a huge attack surface.
* **The Solution:** We use a heavy "Build Stage" to compile the code, and then copy *only* the final `.jar` file into a tiny, secure "Runtime Stage" which is deployed to AWS.

---

## Stage 1: The Build Stage

```dockerfile
FROM maven:3.9.6-eclipse-temurin-17 AS build
```
* **What it does:** Pulls a heavy image containing Maven and the Java 17 JDK. We explicitly name this stage `build`.

```dockerfile
WORKDIR /app
```
* **What it does:** Sets the working directory inside the container to `/app`.

```dockerfile
COPY pom.xml .
RUN mvn dependency:go-offline -B
```
* **What it does:** Copies *only* the `pom.xml` first, and tells Maven to download all the dependencies from the internet.
* **Why? (Layer Caching):** Docker builds images in layers and heavily caches them. If you change a single line of Java code, Docker will skip downloading dependencies and use the cached layer because `pom.xml` didn't change. If we copied the source code *before* downloading dependencies, a single code change would force Docker to re-download the entire internet every time!

```dockerfile
COPY src ./src
RUN mvn clean package -DskipTests
```
* **What it does:** Copies the actual Java source code and packages it into a `.jar` file.
* **Why skip tests?** In enterprise environments, the CI/CD pipeline (e.g., GitHub Actions) runs the tests *before* the Docker build is ever triggered. Running them again inside Docker wastes time.

---

## Stage 2: The Runtime Stage (Production)

```dockerfile
FROM eclipse-temurin:17-jre-jammy
```
* **What it does:** Pulls the official Java 17 Runtime Environment (JRE) based on **Ubuntu Jammy (22.04 LTS)**.
* **Why JRE instead of JDK?** The JRE can only *run* Java, not compile it. It's much smaller.
* **Why Ubuntu Jammy instead of Alpine?** While Alpine Linux is smaller, it lacks native support for Apple Silicon (M1/M2/M3) ARM64 architectures on certain Java distributions. Ubuntu Jammy is an enterprise standard that guarantees your image will run flawlessly on both a Mac laptop and an AWS server.

```dockerfile
RUN addgroup --system appgroup && adduser --system --ingroup appgroup appuser
```
* **What it does:** Creates a restricted, non-root user and group.
* **Why? (Critical Security):** By default, Docker containers run as the `root` (administrator) user. If a vulnerability is found in your application, the hacker gains `root` access inside the container, making it easy to break out and attack the host server. We *must* run the app as a restricted user.

```dockerfile
COPY --from=build /app/target/deploydemo-0.0.1-SNAPSHOT.jar app.jar
```
* **What it does:** Reaches back into the `build` stage and grabs the compiled `.jar` file, dropping it into the new, clean Alpine image as `app.jar`.

```dockerfile
RUN chown appuser:appgroup app.jar
USER appuser
```
* **What it does:** Gives the new restricted user permission to own the file, and tells Docker to switch from `root` to `appuser` for all subsequent commands.

```dockerfile
EXPOSE 8080
```
* **What it does:** Purely for documentation. It tells other engineers (and AWS) that this container listens for traffic on port 8080.

```dockerfile
ENTRYPOINT ["java", "-XX:+UseContainerSupport", "-XX:MaxRAMPercentage=75.0", "-jar", "app.jar"]
```
* **What it does:** The command that actually boots the Spring Boot application.
* **Why those specific JVM flags?** 
  * `-XX:+UseContainerSupport`: Historically, the JVM would read the *Host Server's* total RAM (e.g., 64GB) instead of the *Docker Container's* limited RAM (e.g., 1GB), causing it to over-allocate memory and crash (OOM Kill). This flag ensures Java respects Docker limits.
  * `-XX:MaxRAMPercentage=75.0`: Tells Java to safely use exactly 75% of whatever RAM AWS/Kubernetes provisions for it, leaving the remaining 25% for the Ubuntu operating system to function.

---

## Docker Compose & Container Networking (Way 3)

In our `compose.yaml` file, we defined two services (`mysql-db` and `spring-app`) inside the same file. This represents the ultimate industry standard for local development, known as the "All-in-One" method.

### 1. The Virtual Street (Docker Networks)
For two containers to communicate, they must live on the same virtual network.
* By placing both the database and the application in the same `compose.yaml` file, Docker automatically creates a secure, private virtual network for them.
* Because they are on the same network, they can communicate with each other using their container names as hostnames. This is why our Spring Boot URL is `jdbc:mysql://mysql-db:3306` instead of `localhost:3306`. The application reaches out across the network to find the computer named `mysql-db`!

### 2. Spring Boot Externalized Configuration (Relaxed Binding)
In our Java code, we hardcoded the database URL and credentials inside `application.yaml`. So why do we define `environment:` variables in `compose.yaml`?
* **Order of Precedence:** Spring Boot is built specifically for the cloud. If an Environment Variable is provided, it completely overwrites the hardcoded `application.yaml` value at runtime. This allows us to keep local development settings in our code, but securely inject production credentials via Docker.
* **Relaxed Binding:** Linux environment variables cannot contain dots (`.`). Spring Boot magically translates standard uppercase environment variables into Java properties.
  * `SPRING_DATASOURCE_URL` ➔ `spring.datasource.url`
  * `SPRING_DATASOURCE_USERNAME` ➔ `spring.datasource.username`

### 3. The `depends_on` Flag
We added `depends_on` to the `spring-app` configuration. If Spring Boot starts before the database is fully running, Spring Boot will instantly crash because it cannot connect. This flag tells Docker Compose: *"Wait until the MySQL healthcheck returns 'healthy' before you even attempt to boot the Spring Boot container."*
