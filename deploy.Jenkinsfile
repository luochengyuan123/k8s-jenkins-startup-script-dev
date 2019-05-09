// 测试环境Jenkinsfile

def projectName, targetDir, mvnArgs = ""
if (params.subProject != null) {
    projectName = params.subProject.trim()
    targetDir = "${projectName}/target"
    mvnArgs = "-pl ${projectName} -am -amd"
} else {
    projectName = env.JOB_NAME.substring(2, env.JOB_NAME.length())
    targetDir = "target"
}

def gitUrl = "https://github.com/luochengyuan123/${env.JOB_NAME.substring(2, env.JOB_NAME.length())}.git"
def gitCredential = "gitlabjenkins"
def gitBranch = "master"
def imageTag = "latest-${env.BUILD_ID}"
def namespace = "kube-ops"

if (params.BRANCH != null) {
    gitBranch = params.BRANCH.trim()
    if (gitBranch != "master") {
        imageTag = ${gitBranch} + "-${env.BUILD_ID}"
    }
}

def registryUrl = "harbor.haimaxy.com"
def registryCredential = "dockerHub"


def dockerFileContext = """
FROM openjdk:8-jdk-alpine
MAINTAINER oliver <oliverluo@gmail.com>
ENV LANG en_US.UTF-8
ENV LANGUAGE en_US:en
ENV LC_ALL en_US.UTF-8
ENV TZ=Asia/Shanghai
RUN mkdir /app
WORKDIR /app
COPY target/polls-0.0.1-SNAPSHOT.jar /app/polls.jar
EXPOSE 8080
ENTRYPOINT ["java", "-Djava.security.egd=file:/dev/./urandom", "-jar","/app/polls.jar"]
"""

def deployContext = """
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: polling-server
  namespace: kube-ops
  labels:
    app: polling-server
spec:
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: polling-server
    spec:
      restartPolicy: Always
      imagePullSecrets:
        - name: myreg
      containers:
      - image: harbor.haimaxy.com/${namespace}/${projectName}:${imageTag}
        name: polling-server
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
          name: api
        env:
        - name: DB_HOST
          value: mysql
        - name: DB_PORT
          value: "3306"
        - name: DB_NAME
          value: polling_app
        - name: DB_USER
          value: polling
        - name: DB_PASSWORD
          value: polling321
---
kind: Service
apiVersion: v1
metadata:
  name: polling-server
  namespace: kube-ops
spec:
  selector:
    app: polling-server
  type:  ClusterIP
  ports:
  - name: api-port
    port: 8080
    targetPort:  api
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: mysql
  namespace: kube-ops
spec:
  template:
    metadata:
      labels:
        app: mysql
    spec:
      restartPolicy: Always
      containers:
      - name: mysql
        image: mysql:5.7
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 3306
          name: dbport
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: rootPassW0rd
        - name: MYSQL_DATABASE
          value: polling_app
        - name: MYSQL_USER
          value: polling
        - name: MYSQL_PASSWORD
          value: polling321
        volumeMounts:
        - name: db
          mountPath: /var/lib/mysql
      volumes:
      - name: db
        hostPath:
          path: /var/lib/mysql
---
kind: Service
apiVersion: v1
metadata:
  name: mysql
  namespace: kube-ops
spec:
  selector:
    app: mysql
  type:  ClusterIP
  ports:
  - name: dbport
    port: 3306
    targetPort: dbport
"""

node('jenkins-master') {
    def mvnHome
    stage('Preparation') {
        git branch: gitBranch, credentialsId: gitCredential, url: gitUrl
        mvnHome = tool 'M3'
    }

    stage('Maven build') {
        sh "'${mvnHome}/bin/mvn' clean package -Dmaven.test.skip=true"
        archive 'target/*.jar'

        def remote = [:]
        remote.name = "jenkins-docker"
        remote.host = "192.168.56.62"
        remote.allowAnyHosts = true
        withCredentials([usernamePassword(credentialsId: registryCredential, usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
            remote.user = USERNAME
            remote.password = PASSWORD
            sshCommand remote: remote, command: "mkdir -p /var/jenkins_home/workspace/build_docker && rm -rf /var/jenkins_home/workspace/build_docker/*"
            sshPut remote: remote, from: "${targetDir}/${projectName}.jar", into: "/var/jenkins_home/workspace/build_docker/"
        }
    }
}

node('slave-docker') {
    stage('Build docker image') {
        dir('/var/jenkins_home/workspace/build_docker') {
            writeFile encoding: 'UTF-8', file: "Dockerfile", text: dockerFileContext
            docker.withRegistry("https://${registryUrl}", "${registryCredential}") {
                def image = docker.build("${registryUrl}/${namespace}/${projectName}:${imageTag}", ".")
                image.push()
            }
        }
    }

    stage('Deploy to k8s') {
        dir("/var/jenkins_home/workspace/manifests/${namespace}/${projectName}") {
            sh "rm -rf ./*.yaml"
            writeFile encoding: 'UTF-8', file: "deploy-${projectName}.yaml", text: deployContext

            sh "/opt/k8s/bin/kubectl apply -f ."
        }
    }

    stage('Health check') {
        sleep(30)
        def stepSec = 15
        def failureTimes = 0
        while (true) {
            def healthState = sh(script: "/opt/kubernetes/bin/kubectl -n ${namespace} get pod  | grep ${projectName} | awk '{print \$2}'", returnStdout: true).trim().split('/')
            if (healthState[0] != healthState[1]) {
                if (failureTimes == 20) {
                    currentBuild.result = "FAILURE"
                    throw new Exception("${projectName} health check failed")
                }
                failureTimes++
                sleep(stepSec)
            } else {
                break
            }
        }
    }
}

