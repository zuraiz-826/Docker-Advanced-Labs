Lab 16: Managing Docker Secrets
Objectives
By the end of this lab, students will be able to:

Understand the importance of secure secret management in containerized environments
Create and manage Docker secrets using the Docker CLI
Deploy services that securely consume secrets for database credentials
Use Docker Swarm to distribute secrets across multiple nodes
Implement best practices for secret security and avoid exposing sensitive data in logs
Verify that secrets are properly encrypted and stored securely
Prerequisites
Before starting this lab, students should have:

Basic understanding of Docker containers and images
Familiarity with Docker Compose and basic YAML syntax
Knowledge of Linux command-line operations
Understanding of database connection concepts
Completion of previous Docker labs covering basic container operations
Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides pre-configured Linux-based cloud machines for this lab. Simply click Start Lab to access your environment - no need to build your own VM or install Docker manually.

Your lab environment includes:

Ubuntu 20.04 LTS with Docker Engine installed
Docker Swarm mode capability
Pre-installed text editors (nano, vim)
Network connectivity for pulling Docker images
Task 1: Initialize Docker Swarm and Create Basic Secrets
Subtask 1.1: Initialize Docker Swarm Mode
Docker secrets are only available in Docker Swarm mode. Let's start by initializing our swarm cluster.

# Initialize Docker Swarm mode
docker swarm init

# Verify swarm status
docker node ls
Expected Output: You should see your node listed as a manager with "Leader" status.

Subtask 1.2: Create Your First Docker Secret
Let's create a simple secret to understand the basic workflow.

# Create a secret from command line input
echo "mySecretPassword123" | docker secret create db_password -

# Create another secret for database username
echo "admin_user" | docker secret create db_username -

# List all secrets
docker secret ls
Key Concept: The - at the end tells Docker to read the secret value from standard input (stdin).

Subtask 1.3: Inspect Secret Properties
# Inspect the secret (note: actual secret value won't be shown)
docker secret inspect db_password

# View secret details in a more readable format
docker secret inspect db_password --pretty
Important Note: Docker never exposes the actual secret values through inspect commands for security reasons.

Task 2: Deploy a Service Using Secrets for Database Credentials
Subtask 2.1: Create a MySQL Database Service with Secrets
Now let's deploy a MySQL database service that uses our secrets for authentication.

# Deploy MySQL service using secrets
docker service create \
  --name mysql_db \
  --secret db_username \
  --secret db_password \
  --env MYSQL_ROOT_PASSWORD_FILE=/run/secrets/db_password \
  --env MYSQL_USER_FILE=/run/secrets/db_username \
  --env MYSQL_PASSWORD_FILE=/run/secrets/db_password \
  --env MYSQL_DATABASE=testdb \
  --publish 3306:3306 \
  mysql:8.0
Key Concept: The _FILE suffix tells MySQL to read the values from files instead of environment variables, which is more secure.

Subtask 2.2: Verify Service Deployment
# Check service status
docker service ls

# View service details
docker service ps mysql_db

# Check service logs (notice secrets are not exposed)
docker service logs mysql_db
Subtask 2.3: Examine Secret Files Inside Container
# Get the container ID
CONTAINER_ID=$(docker ps --filter "name=mysql_db" --format "{{.ID}}")

# Examine the secrets directory inside the container
docker exec $CONTAINER_ID ls -la /run/secrets/

# View the content of secret files (for educational purposes)
docker exec $CONTAINER_ID cat /run/secrets/db_username
docker exec $CONTAINER_ID cat /run/secrets/db_password
Security Note: In production, you should limit access to containers and avoid examining secret contents directly.

Task 3: Advanced Secret Management with Docker Service Create
Subtask 3.1: Create Secrets from Files
Let's create more complex secrets using files, which is often more practical for real-world scenarios.

# Create a directory for our secret files
mkdir -p ~/lab_secrets

# Create a database configuration file
cat > ~/lab_secrets/db_config.json << EOF
{
  "host": "mysql_db",
  "port": 3306,
  "database": "testdb",
  "ssl_mode": "required",
  "connection_timeout": 30
}
EOF

# Create an API key file
echo "api_key_abc123xyz789" > ~/lab_secrets/api_key.txt

# Create secrets from files
docker secret create db_config ~/lab_secrets/db_config.json
docker secret create api_key ~/lab_secrets/api_key.txt

# Verify secrets were created
docker secret ls
Subtask 3.2: Deploy Application Service with Multiple Secrets
# Deploy a web application that uses multiple secrets
docker service create \
  --name web_app \
  --secret db_username \
  --secret db_password \
  --secret db_config \
  --secret api_key \
  --env DB_USERNAME_FILE=/run/secrets/db_username \
  --env DB_PASSWORD_FILE=/run/secrets/db_password \
  --env DB_CONFIG_FILE=/run/secrets/db_config \
  --env API_KEY_FILE=/run/secrets/api_key \
  --publish 8080:80 \
  nginx:alpine

# Verify the service is running
docker service ps web_app
Subtask 3.3: Custom Secret Mount Points
You can also specify custom locations for secrets within containers.

# Create a service with custom secret mount points
docker service create \
  --name custom_app \
  --secret source=db_config,target=/app/config/database.json \
  --secret source=api_key,target=/app/keys/api.key \
  --publish 9090:80 \
  nginx:alpine

# Verify custom mount points
CUSTOM_CONTAINER_ID=$(docker ps --filter "name=custom_app" --format "{{.ID}}")
docker exec $CUSTOM_CONTAINER_ID ls -la /app/config/
docker exec $CUSTOM_CONTAINER_ID ls -la /app/keys/
Task 4: Multi-Node Docker Swarm Secret Management
Subtask 4.1: Simulate Multi-Node Environment
Since we're working with a single machine, we'll demonstrate how secrets work across the swarm by scaling services.

# Scale the web application across multiple replicas
docker service scale web_app=3

# View service distribution
docker service ps web_app

# Check that all replicas have access to secrets
for i in {1..3}; do
  echo "Checking replica $i secrets..."
  CONTAINER_ID=$(docker ps --filter "name=web_app" --format "{{.ID}}" | sed -n "${i}p")
  if [ ! -z "$CONTAINER_ID" ]; then
    docker exec $CONTAINER_ID ls -la /run/secrets/ 2>/dev/null || echo "Container $i not ready yet"
  fi
done
Subtask 4.2: Update Secrets in Running Services
Docker secrets are immutable, but you can update services to use new secrets.

# Create a new version of the API key
echo "api_key_new_version_456" | docker secret create api_key_v2 -

# Update the service to use the new secret
docker service update \
  --secret-rm api_key \
  --secret-add api_key_v2 \
  web_app

# Verify the update
docker service ps web_app
Subtask 4.3: Rolling Updates with Secrets
# Create a new database password
echo "newSecurePassword456" | docker secret create db_password_v2 -

# Perform rolling update of MySQL service
docker service update \
  --secret-rm db_password \
  --secret-add db_password_v2 \
  --env-rm MYSQL_ROOT_PASSWORD_FILE \
  --env-add MYSQL_ROOT_PASSWORD_FILE=/run/secrets/db_password_v2 \
  mysql_db

# Monitor the rolling update
docker service ps mysql_db
Task 5: Security Best Practices and Verification
Subtask 5.1: Verify Secrets Are Not Exposed in Logs
# Check service logs to ensure secrets aren't visible
docker service logs web_app | grep -i password || echo "No passwords found in logs - Good!"
docker service logs mysql_db | grep -i password || echo "No passwords found in logs - Good!"

# Check container environment variables
CONTAINER_ID=$(docker ps --filter "name=web_app" --format "{{.ID}}" | head -1)
docker exec $CONTAINER_ID env | grep -i password || echo "No password environment variables - Good!"
Subtask 5.2: Examine Secret Storage Security
# Check Docker's secret storage (this will show encrypted data)
sudo ls -la /var/lib/docker/swarm/certificates/

# Verify secrets are encrypted at rest
docker secret inspect db_password --format "{{.Spec.Name}}: Created {{.CreatedAt}}"

# Show that secrets have proper access controls
docker secret inspect db_password --format "{{json .}}" | grep -i "encrypt"
Subtask 5.3: Test Secret Access Controls
# Try to access secrets from a container without proper secret assignment
docker run --rm alpine:latest ls -la /run/secrets/ 2>/dev/null || echo "No secrets accessible - Security working correctly!"

# Compare with a container that has secrets
CONTAINER_WITH_SECRETS=$(docker ps --filter "name=web_app" --format "{{.ID}}" | head -1)
docker exec $CONTAINER_WITH_SECRETS ls -la /run/secrets/
Subtask 5.4: Clean Up Unused Secrets
# Remove services first (secrets can't be deleted while in use)
docker service rm web_app custom_app mysql_db

# Wait for services to be fully removed
sleep 10

# List all secrets
docker secret ls

# Remove old secrets
docker secret rm db_password api_key

# Verify cleanup
docker secret ls
Troubleshooting Common Issues
Issue 1: "This node is not a swarm manager"
Solution: Initialize Docker Swarm mode:

docker swarm init
Issue 2: "Secret is in use by service"
Solution: Remove or update the service first:

docker service rm <service_name>
# or
docker service update --secret-rm <secret_name> <service_name>
Issue 3: Service fails to start with secrets
Solution: Check secret names and file paths:

docker service logs <service_name>
docker secret ls
Issue 4: Cannot read secret files in container
Solution: Verify the secret is properly mounted:

docker exec <container_id> ls -la /run/secrets/
Best Practices Summary
Never use environment variables for secrets - Use secret files instead
Rotate secrets regularly - Create new versions and update services
Use least privilege access - Only assign secrets to services that need them
Monitor secret usage - Regularly audit which services use which secrets
Clean up unused secrets - Remove old secrets to reduce attack surface
Use descriptive secret names - Include version numbers or dates when appropriate
Conclusion
In this lab, you have successfully learned how to manage Docker secrets securely. You accomplished the following key tasks:

Initialized Docker Swarm and understood why it's required for secret management
Created secrets using both command-line input and file-based methods
Deployed services that consume secrets securely without exposing sensitive data
Implemented multi-service architectures with shared secret access
Performed rolling updates with secret rotation
Verified security measures to ensure secrets remain protected
Why This Matters: In production environments, managing sensitive information like database passwords, API keys, and certificates is critical for security. Docker secrets provide a secure, encrypted way to distribute this information to containers without exposing it in environment variables, logs, or configuration files. This approach significantly reduces the risk of credential exposure and helps maintain compliance with security best practices.

The skills you've learned in this lab are essential for the Docker Certified Associate (DCA) certification and are directly applicable to real-world container orchestration scenarios. You now understand how to implement secure secret management in Docker Swarm environments, which is a fundamental requirement for production container deployments.

Next Steps: Consider exploring external secret management systems like HashiCorp Vault or Kubernetes secrets for more advanced secret management scenarios in larger, multi-cluster environments.
