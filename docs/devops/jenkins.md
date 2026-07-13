# Jenkins

Reach for this when writing a `Jenkinsfile` (declarative pipeline) or debugging a build.

## Declarative pipeline - the skeleton

```groovy
pipeline {
    agent any                          // where it runs (any available agent)

    tools {
        jdk 'temurin-25'               // names configured in Manage Jenkins → Tools
        maven 'maven-3.9'
    }

    options {
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '20'))
    }

    environment {
        REGISTRY = 'registry.example.com'
        IMAGE    = "${REGISTRY}/my-app:${env.BUILD_NUMBER}"
    }

    stages {
        stage('Test') {
            steps { sh './mvnw --batch-mode verify' }
            post { always { junit 'target/surefire-reports/*.xml' } }
        }
        stage('Build') {
            steps { sh './mvnw --batch-mode package -DskipTests' }
        }
        stage('Docker') {
            steps { sh "docker build -t ${IMAGE} ." }
        }
    }

    post {
        success { echo 'Green.' }
        failure { echo 'Broke.' }      // hook Slack/email here
        always  { cleanWs() }          // clean workspace
    }
}
```

`pipeline → stages → stage → steps`. `steps` is where actual work (`sh`, etc.) happens.

## Agents

```groovy
agent any                              // any node
agent none                             // set per-stage instead (see below)
agent { label 'linux && docker' }      // node matching labels
agent {                                // run the whole thing in a container
    docker { image 'eclipse-temurin:25-jdk-alpine'; args '-v $HOME/.m2:/root/.m2' }
}
```

Per-stage agents (e.g. `agent none` at top, then):
```groovy
stages {
  stage('Build') { agent { docker { image 'maven:3.9-eclipse-temurin-25' } } steps { ... } }
}
```

## Credentials (never hardcode secrets)

```groovy
environment {
    // 'aws-creds' is a username/password credential ID from Manage Jenkins → Credentials
    AWS = credentials('aws-creds')     // sets AWS_USR and AWS_PSW
}
steps {
    withCredentials([string(credentialsId: 'api-token', variable: 'TOKEN')]) {
        sh 'curl -H "Authorization: Bearer $TOKEN" https://api.example.com'
    }
}
```

Types: `string`, `usernamePassword`, `sshUserPrivateKey`, `file`. Jenkins masks credential
values in console output.

## Parameters & manual input

```groovy
parameters {
    choice(name: 'ENV', choices: ['staging', 'prod'], description: 'Target')
    booleanParam(name: 'SKIP_TESTS', defaultValue: false)
    string(name: 'VERSION', defaultValue: 'latest')
}
// reference as params.ENV

stage('Deploy prod') {
    when { expression { params.ENV == 'prod' } }
    steps {
        input message: 'Deploy to production?', ok: 'Ship it'   // pauses for a human
        sh './deploy.sh prod'
    }
}
```

## Conditional stages (`when`)

```groovy
when { branch 'main' }
when { expression { params.ENV == 'prod' } }
when { changeset '**/*.java' }
when { allOf { branch 'main'; environment name: 'DEPLOY', value: 'true' } }
```

## Parallel stages

```groovy
stage('Checks') {
    parallel {
        stage('Unit')        { steps { sh './mvnw test' } }
        stage('Lint')        { steps { sh './mvnw spotless:check' } }
        stage('Integration') { steps { sh './mvnw verify -Pit' } }
    }
}
```

## Shared libraries (DRY across repos)

```groovy
@Library('my-shared-lib') _            // top of Jenkinsfile
mvnBuild()                             // a step defined in vars/mvnBuild.groovy
```
Library repo layout: `vars/` (global steps), `src/` (Groovy classes), `resources/`.

## post conditions

| Condition | Fires when |
|---|---|
| `always` | every time (cleanup, publish reports) |
| `success` | build passed |
| `failure` | build failed |
| `unstable` | tests failed but build didn't error |
| `changed` | status differs from previous run |
| `aborted` | manually stopped / timed out |

## Gotchas / things I always forget

- **Declarative vs scripted.** Declarative (`pipeline { }`) is the standard - structured, validated. Scripted (`node { }`) is raw Groovy, only for edge cases. Don't mix unless using a `script { }` block inside declarative.
- `sh 'cd foo && ...'` - a `cd` doesn't persist across `sh` steps (each is a new shell). Chain in one `sh` or use `dir('foo') { }`.
- `credentials('id')` on a `usernamePassword` gives you `NAME_USR` and `NAME_PSW`, not the raw value - trips everyone up.
- `environment` values are evaluated once at the start; for dynamic values compute them in a `script { }` step.
- `input` blocks an executor while waiting - put it in a stage with `agent none` or it holds a node hostage.
- `parallel` stages need enough executors/agents or they queue instead of running concurrently.
- `post { always { cleanWs() } }` - without it, workspaces accumulate and fill the disk.
- Test reports (`junit`) must publish in `post { always }`, or a failed build skips them and you can't see *why* it failed.

## Quick reference

| Task | Snippet |
|---|---|
| Run in container | `agent { docker { image '...' } }` |
| Only on main | `when { branch 'main' }` |
| Secret env | `credentials('id')` → `ID_USR` / `ID_PSW` |
| Wrap a secret | `withCredentials([...]) { sh ... }` |
| Human gate | `input message: 'Deploy?'` |
| Run stages concurrently | `parallel { stage ... }` |
| Publish tests | `post { always { junit '...' } }` |
| Change dir | `dir('sub') { sh ... }` |
