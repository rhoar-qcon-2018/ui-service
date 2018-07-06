def ciProject = 'labs-ci-cd'
def testProject = 'labs-test'
def devProject = 'labs-dev'
openshift.withCluster() {
  openshift.withProject() {
    ciProject = openshift.project()
    println(ciProject)
    testProject = ciProject.replaceFirst(/^labs-ci-cd/, 'labs-test')
    println(testProject)
    devProject = ciProject.replaceFirst(/^labs-ci-cd/, 'labs-dev')
    println(devProject)
  }
}

def buildConfig = { project, namespace, buildSecret, fromImageStream ->
  if (!fromImageStream) {
    fromImageStream = 'registry.access.redhat.com/redhat-openjdk-18/openjdk18-openshift'
  }
  def template = """
---
apiVersion: build.openshift.io/v1
kind: BuildConfig
metadata:
  labels:
    build: ${project}
  name: ${project}
  namespace: ${namespace}
spec:
  failedBuildsHistoryLimit: 5
  nodeSelector: null
  output:
    to:
      kind: ImageStreamTag
      name: '${project}:latest'
  postCommit: {}
  resources: {}
  runPolicy: Serial
  source:
    binary: {}
    type: Binary
  strategy:
    sourceStrategy:
      from:
        kind: DockerImage
        name: '${fromImageStream}'
    type: Source
  successfulBuildsHistoryLimit: 5
"""
  openshift.withCluster() {
    openshift.apply(template, "--namespace=${namespace}")
  }
}

def deploymentConfig = {project, ciNamespace, targetNamespace ->
  def template = """
---
apiVersion: v1
kind: List
items:
- apiVersion: v1
  kind: ImageStream
  metadata:
    labels:
      build: '${project}'
    name: '${project}'
    namespace: '${targetNamespace}'
  spec: {}
- apiVersion: v1
  kind: DeploymentConfig
  metadata:
    labels:
      app: '${project}'
    name: '${project}'
  spec:
    replicas: 1
    selector:
      name: '${project}'
    strategy:
      activeDeadlineSeconds: 21600
      resources: {}
      rollingParams:
        intervalSeconds: 1
        maxSurge: 25%
        maxUnavailable: 25%
        timeoutSeconds: 600
        updatePeriodSeconds: 1
      type: Rolling
    template:
      metadata:
        creationTimestamp: null
        labels:
          name: '${project}'
      spec:
        containers:
          - image: '${project}'
            imagePullPolicy: Always
            name: '${project}'
            env:
            - name: KUBERNETES_NAMESPACE
              value: ${targetNamespace}
            ports:
              - containerPort: 5800
                protocol: TCP
              - containerPort: 8778
                protocol: TCP
              - containerPort: 8080
                protocol: TCP
            livenessProbe:
                failureThreshold: 3
                httpGet:
                  path: /api/v1/health
                  port: 8080
                  scheme: HTTP
                initialDelaySeconds: 5
                periodSeconds: 10
                successThreshold: 1
                timeoutSeconds: 1
            readinessProbe:
              failureThreshold: 3
              httpGet:
                path: /api/v1/health
                port: 8080
                scheme: HTTP
              initialDelaySeconds: 10
              periodSeconds: 10
              successThreshold: 1
              timeoutSeconds: 1
            resources: {}
            terminationMessagePath: /dev/termination-log
        dnsPolicy: ClusterFirst
        restartPolicy: Always
        securityContext: {}
        terminationGracePeriodSeconds: 30
    test: false
    triggers:
      - type: ConfigChange
      - imageChangeParams:
          automatic: true
          containerNames:
            - '${project}'
          from:
            kind: ImageStreamTag
            name: '${project}:latest'
        type: ImageChange
- apiVersion: v1
  kind: Service
  metadata:
    labels:
      name: '${project}'
    name: '${project}'
  spec:
    ports:
      - name: http
        port: 8080
        protocol: TCP
        targetPort: 8080
      - name: jolokia
        port: 8778
        protocol: TCP
        targetPort: 8778
      - name: cluster
        port: 5800
        protocol: TCP
        targetPort: 5800
    selector:
      name: '${project}'
    sessionAffinity: None
    type: ClusterIP
- apiVersion: v1
  kind: Route
  metadata:
    labels:
      name: '${project}'
    name: '${project}'
  spec:
    port:
      targetPort: http
    to:
      kind: Service
      name: '${project}'
      weight: 100
    wildcardPolicy: None
"""
  openshift.withCluster() {
    openshift.apply(template, "--namespace=${targetNamespace}")
  }
}

pipeline {
  agent {
    label 'jenkins-slave-mvn'
  }
  environment {
    PROJECT_NAME = 'ui-service'
    KUBERNETES_NAMESPACE = "${ciProject}"
  }
  stages {
    stage('Quality And Security') {
      parallel {
        stage('OWASP Dependency Check') {
          steps {
            sh 'mvn -T 2 dependency-check:check'
          }
        }
        stage('Compile & Test') {
          steps {
            sh 'mvn -T 2 package'
          }
        }
        stage('Ensure SonarQube Webhook is configured') {
          when {
            expression {
              withSonarQubeEnv('sonar') {
                def retVal = sh(returnStatus: true, script: "curl -u \"${SONAR_AUTH_TOKEN}:\" http://sonarqube:9000/api/webhooks/list | grep Jenkins")
                echo "CURL COMMAND: ${retVal}"
                return (retVal > 0)
              }
            }
          }
          steps {
            withSonarQubeEnv('sonar') {
              sh "curl -X POST -u \"${SONAR_AUTH_TOKEN}:\" -F \"name=Jenkins\" -F \"url=http://jenkins/sonarqube-webhook/\" http://sonarqube:9000/api/webhooks/create"
            }
          }
        }
      }
    }
    stage('Wait for SonarQube Quality Gate') {
      steps {
        script {
          withSonarQubeEnv('sonar') {
            sh 'mvn -T 2 sonar:sonar'
          }
          def qualitygate = waitForQualityGate()
          if (qualitygate.status != "OK") {
            error "Pipeline aborted due to quality gate failure: ${qualitygate.status}"
          }
        }
      }
    }
    stage('OpenShift Deployments') {
      parallel {
        stage('Publish Artifacts') {
          steps {
            sh 'mvn package deploy:deploy -DskipTests -DaltDeploymentRepository=nexus::default::http://nexus:8081/repository/maven-snapshots/'
          }
        }
        stage('Create Binary BuildConfig') {
          steps {
            script {
              buildConfig(PROJECT_NAME, ciProject, UUID.randomUUID().toString())
            }
          }
        }
        stage('Create Test Deployment') {
          steps {
            script {
              deploymentConfig(PROJECT_NAME, ciProject, testProject)
            }
          }
        }
        stage('Create Dev Deployment') {
          steps {
            script {
              deploymentConfig(PROJECT_NAME, ciProject, devProject)
            }
          }
        }
      }
    }
    stage('Build Image') {
      steps {
        script {
          openshift.selector('bc', PROJECT_NAME).startBuild("--from-file=target/${PROJECT_NAME}.jar", '--wait')
        }
      }
    }
    stage('Promote to TEST') {
      steps {
        script {
          openshift.tag("${PROJECT_NAME}:latest", "${testProject}/${PROJECT_NAME}:latest")
        }
      }
    }
    stage('Promote to DEMO') {
      input {
        message "Promote service to DEMO environment?"
        ok "PROMOTE"
      }
      steps {
        script {
          openshift.tag("${PROJECT_NAME}:latest", "${devProject}/${PROJECT_NAME}:latest")
        }
      }
    }
  }
}