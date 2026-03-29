# Jenkins Cheatsheet

## Key Concepts

| Term | What It Is |
|------|------------|
| Controller | The Jenkins server that schedules and coordinates builds |
| Agent | A machine (or container) that runs build steps |
| Job / Project | A configured unit of work |
| Pipeline | A multi-stage job defined in a `Jenkinsfile` |
| Jenkinsfile | A text file (Groovy DSL) that defines the pipeline |
| Stage | A named phase of the pipeline (Build, Test, Deploy) |
| Step | A single action within a stage (sh, echo, withCredentials) |
| Executor | A slot on an agent that runs one build at a time |

---

## Declarative Jenkinsfile Structure

```groovy
pipeline {
    agent any               // Run on any available agent

    environment {
        IMAGE = 'myapp'     // Available as env.IMAGE in steps
    }

    options {
        timeout(time: 20, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
        disableConcurrentBuilds()
    }

    triggers {
        githubPush()        // Trigger on GitHub push (requires webhook)
        pollSCM('H/5 * * * *')  // Poll every 5 minutes (alternative)
    }

    stages {
        stage('Build') {
            steps {
                sh 'docker build -t $IMAGE .'
            }
        }

        stage('Test') {
            steps {
                sh 'pytest tests/'
            }
            post {
                always {
                    junit 'test-results/*.xml'  // Archive test results
                }
            }
        }

        stage('Deploy') {
            when {
                branch 'main'           // Only run on main branch
            }
            steps {
                sh 'kubectl apply -f k8s/'
            }
        }
    }

    post {
        success { echo "Pipeline succeeded" }
        failure { echo "Pipeline failed" }
        always  { cleanWs() }           // Clean workspace after every build
    }
}
```

---

## Common Steps

```groovy
// Shell command
sh 'npm install'
sh """
    docker build -t myapp:${IMAGE_TAG} .
    docker push myapp:${IMAGE_TAG}
"""

// Echo
echo "Building version ${env.BUILD_NUMBER}"

// Capture output of a shell command
def tag = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()

// Check exit code
def status = sh(script: 'make test', returnStatus: true)
if (status != 0) { error "Tests failed" }

// Read a file
def content = readFile('version.txt').trim()

// Write a file
writeFile file: 'output.txt', text: 'Hello'

// Archive build artifacts
archiveArtifacts artifacts: 'dist/**', allowEmptyArchive: true

// Stash files between stages (when using different agents)
stash name: 'build-output', includes: 'dist/**'
unstash 'build-output'
```

---

## Credentials

```groovy
// Username + Password (e.g., Docker Hub)
withCredentials([usernamePassword(
    credentialsId: 'dockerhub-creds',
    usernameVariable: 'DOCKER_USER',
    passwordVariable: 'DOCKER_PASS'
)]) {
    sh 'echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin'
}

// Secret text (e.g., API key)
withCredentials([string(credentialsId: 'slack-webhook', variable: 'SLACK_URL')]) {
    sh 'curl -X POST "$SLACK_URL" -d \'{"text":"deployed"}\''
}

// Secret file (e.g., kubeconfig)
withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
    sh 'kubectl get nodes'
}

// SSH key
withCredentials([sshUserPrivateKey(
    credentialsId: 'my-ssh-key',
    keyFileVariable: 'SSH_KEY'
)]) {
    sh 'ssh -i $SSH_KEY ec2-user@10.0.0.1 "systemctl restart myapp"'
}
```

---

## Environment Variables

```groovy
// Built-in Jenkins variables
env.BUILD_NUMBER         // e.g., "42"
env.BUILD_URL            // Full URL of this build
env.JOB_NAME             // e.g., "my-pipeline"
env.GIT_BRANCH           // e.g., "origin/main"
env.GIT_COMMIT           // Full commit SHA
env.WORKSPACE            // Path to the build workspace on the agent
env.NODE_NAME            // Name of the agent running this build
```

---

## Conditional Execution

```groovy
// Run only on a specific branch
when { branch 'main' }

// Run only when a file changed
when { changeset 'src/**' }

// Run based on a parameter
when { expression { return params.DEPLOY == 'true' } }

// Run when triggered manually
when { triggeredBy 'UserIdCause' }

// Combine conditions
when {
    allOf {
        branch 'main'
        not { changeRequest() }
    }
}
```

---

## Parameters

```groovy
parameters {
    string(name: 'IMAGE_TAG', defaultValue: 'latest', description: 'Docker image tag')
    booleanParam(name: 'DEPLOY', defaultValue: false, description: 'Deploy to production?')
    choice(name: 'ENV', choices: ['dev', 'staging', 'production'], description: 'Target environment')
}

// Access in steps
sh "kubectl set image deployment/app app=${params.IMAGE_TAG}"
```

---

## Parallel Stages

```groovy
stage('Test') {
    parallel {
        stage('Unit Tests') {
            steps { sh 'pytest tests/unit' }
        }
        stage('Lint') {
            steps { sh 'flake8 .' }
        }
        stage('Security Scan') {
            steps { sh 'trivy image myapp:latest' }
        }
    }
}
```

---

## Managing Jenkins via CLI

```bash
# Download the Jenkins CLI jar (replace with your Jenkins URL)
curl -O http://localhost:8080/jnlpJars/jenkins-cli.jar

# List all jobs
java -jar jenkins-cli.jar -s http://localhost:8080/ list-jobs

# Trigger a build
java -jar jenkins-cli.jar -s http://localhost:8080/ build my-pipeline -w

# Get build log
java -jar jenkins-cli.jar -s http://localhost:8080/ console my-pipeline lastBuild

# Reload configuration from disk
java -jar jenkins-cli.jar -s http://localhost:8080/ reload-configuration
```

---

## Important Plugins

| Plugin | Purpose |
|--------|---------|
| Pipeline | Core pipeline support |
| Git | Checkout from Git/GitHub |
| GitHub Integration | Webhooks and PR status |
| Docker Pipeline | `docker.build()`, `docker.push()` DSL |
| Kubernetes | Run agents as K8s pods |
| Credentials Binding | `withCredentials` step |
| Blue Ocean | Visual pipeline UI |
| Slack Notification | Slack alerts |
| SonarQube Scanner | Code quality analysis |
| Timestamper | Adds timestamps to build logs |

---

## Webhook Setup (GitHub)

1. In GitHub: **Repo → Settings → Webhooks → Add webhook**
2. Payload URL: `http://<jenkins-host>/github-webhook/`
3. Content type: `application/json`
4. Events: **Just the push event** (or select events you need)
5. In Jenkins: Pipeline job → **Build Triggers → GitHub hook trigger for GITScm polling**

---

## Common Errors

| Error | Likely Cause | Fix |
|-------|-------------|-----|
| `No such DSL method 'withCredentials'` | Credentials Binding plugin missing | Install the plugin |
| `Permission denied` | Agent user cannot run docker | Add user to docker group on agent |
| `git: command not found` | Git not installed on agent | Install git on the agent |
| `KUBECONFIG: unbound variable` | Secret file credential not set | Check credential ID matches Jenkinsfile |
| Build stuck in queue | No agents available | Check agent is online and has free executors |
| `Cannot contact node` | Agent disconnected | Reconnect agent or restart agent process |
