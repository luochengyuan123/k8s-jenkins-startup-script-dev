// 测试环境Jenkinsfile
def envKey = env.JOB_NAME.substring(0, 1)
def domainName, namespace, portPrefix, springConfigLabel, serviceAddr
switch (envKey) {
    case "d":
        domainName = "d.payeco.com"
        namespace = "payeco-dev"
        springConfigLabel = "dev"
        portPrefix = ""
        serviceAddr = "\${NODE_NAME}"
        break
    case "t":
        domainName = "t.payeco.com"
        namespace = "payeco-test"
        springConfigLabel = "test"
        portPrefix = "1"
        serviceAddr = "\$(telnet \${serviceName} 2>/dev/null | awk '/Trying/{split(\$2,a,\".\");print a[1]\".\"a[2]\".\"a[3]\".\"a[4]}')"
        break
    case "j":
        domainName = "test.payeco.com"
        namespace = "payeco-joint"
        springConfigLabel = "debug"
        portPrefix = "2"
        serviceAddr = "\$(telnet \${serviceName} 2>/dev/null | awk '/Trying/{split(\$2,a,\".\");print a[1]\".\"a[2]\".\"a[3]\".\"a[4]}')"
        envKey = "test"
        break
    default:
        println("No matching case found!!")
}

def projectName, targetDir, mvnArgs = ""
if (params.subProject != null) {
    projectName = params.subProject.trim()
    targetDir = "${projectName}/target"
    mvnArgs = "-pl ${projectName} -am -amd"
} else {
    projectName = env.JOB_NAME.substring(2, env.JOB_NAME.length())
    targetDir = "target"
}

def gitUrl = "http://10.123.107.106/payeco/${env.JOB_NAME.substring(2, env.JOB_NAME.length())}.git"
def gitCredential = "692a8f27-c716-42c8-a334-81db86d891a5"
def gitBranch = "master"
def imageTag = "latest-${env.BUILD_ID}"

if (params.BRANCH != null) {
    gitBranch = params.BRANCH.trim()
    if (gitBranch != "master") {
        imageTag = gitBranch + "-${env.BUILD_ID}"
    }
}

def registryUrl = "harbor.t.payeco.com"
def registryCredential = "9b041abc-e0e6-43ec-a87c-f65a74a125c8"

def appConfig
node('jenkins-master') {
    dir('/var/jenkins_home/workspace/tmp/') {
        git branch: "master", credentialsId: gitCredential, url: "http://10.123.107.106/payeco/k8s-jenkins-startup-script-dev.git"
        appConfig = readJSON file: 'services.json'
    }
}

def port, jvmArgs, bootArgs, webui, uriPath, dataVolumePath
for (i = 0; i < appConfig.services.size(); i++) {
    if (appConfig.services[i].name.trim() == "${projectName}") {
        port = appConfig.services[i].port
        jvmArgs = appConfig.services[i].jvmArgs
        if (projectName != "eureka" || projectName != "config-server") {
            bootArgs = appConfig.services[i].bootArgs + " --spring.cloud.config.label=${springConfigLabel}"
        } else {
            bootArgs = appConfig.services[i].bootArgs
        }
        webui = appConfig.services[i].webui
        dataVolumePath = appConfig.services[i].dataVolumePath
        if (appConfig.services[i].uriPath.trim() == "") {
            uriPath = projectName
        } else {
            uriPath = appConfig.services[i].uriPath
        }
    }
}

def dockerFileContext = """
FROM ${registryUrl}/library/jdk/ubuntu-jdk1.8.191
MAINTAINER ops "ops@k8s.payeco.com"
ARG svcport=${port}
ARG svcname=${projectName}
ENV APP_NAME \${svcname}
ENV APP_HOME /opt/app/
ENV APP_PORT \${svcport}
RUN mkdir -p /opt/app /opt/logs
# App log directory is a volume
VOLUME /opt/logs/
# for main web interface:
EXPOSE \${svcport}
WORKDIR /opt/app/
COPY \${svcname}.jar /opt/app/\${svcname}.jar
COPY run.sh /opt/app/run.sh
RUN chmod +x /opt/app/run.sh
ENTRYPOINT ["/opt/app/run.sh"]
"""

def runScriptContext = """#!/bin/bash
# run service
serviceName="${projectName}"
serviceIP="${serviceAddr}"
jarName="\${serviceName}.jar"
jvmArgs="-server -Djava.security.egd=file:/dev/./urandom 
${jvmArgs}
-XX:+AggressiveOpts
-XX:+UseBiasedLocking
-XX:+DisableExplicitGC
-XX:+UseParNewGC
-XX:+UseConcMarkSweepGC
-XX:+CMSParallelRemarkEnabled
-XX:+UseFastAccessorMethods
-XX:+UseCMSInitiatingOccupancyOnly
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/opt/logs/heapdump/\${serviceName}"
bootArgs="--logging.path=/opt/logs
--logging.level.root=info
--eureka.instance.prefer-ip-address=true --eureka.instance.ip-address=\${serviceIP}
${bootArgs}"
appDir=/opt/app/
echo "Starting \${serviceName}..."
cd \${appDir}
java \$jvmArgs -jar \$jarName \$bootArgs
"""

def deployContext = """
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: ${projectName}
    release: ${projectName}
  name: ${projectName}
  namespace: ${namespace}
spec:
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: ${projectName}
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: ${projectName}
    spec:
      serviceAccount: default
      serviceAccountName: default
      containers:
      - env:
          - name: NODE_NAME
            valueFrom:
              fieldRef:
                fieldPath: spec.nodeName
        image: ${registryUrl}/${namespace}/${projectName}:${imageTag}
        imagePullPolicy: IfNotPresent
        livenessProbe:
          failureThreshold: 3
          httpGet:
            path: /actuator/info
            port: ${port}
            scheme: HTTP
          initialDelaySeconds: 240
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 1
        name: ${projectName}
        ports:
        - containerPort: ${port}
          name: http
          protocol: TCP
        readinessProbe:
          failureThreshold: 3
          httpGet:
            path: /actuator/info
            port: ${port}
            scheme: HTTP
          initialDelaySeconds: 60
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 1
        resources:
          limits:
            cpu: "1"
            memory: 2Gi
          requests:
            cpu: 500m
            memory: 1Gi
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /opt/logs
          name: ${projectName}-data
          subPath: log
        - mountPath: ${dataVolumePath}
          name: ${projectName}-data
          subPath: data
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
      volumes:
      - name: ${projectName}-data
        persistentVolumeClaim:
          claimName: pvc-${projectName}
"""

def serviceContext = """
apiVersion: v1
kind: Service
metadata:
  annotations:
  labels:
    app: ${projectName}
    release: ${projectName}
  name: ${projectName}
  namespace: ${namespace}
spec:
  ports:
  - name: http
    nodePort: ${portPrefix}${port}
    port: ${port}
    protocol: TCP
    targetPort: ${port}
  selector:
    app: ${projectName}
  sessionAffinity: None
  type: NodePort
"""

def pvContext = """
apiVersion: v1
kind: PersistentVolume
metadata:
  finalizers:
  - kubernetes.io/pv-protection
  labels:
    name: pv-${namespace}-${projectName}
  name: pv-${namespace}-${projectName}
spec:
  accessModes:
  - ReadWriteMany
  capacity:
    storage: 200Gi
  cephfs:
    monitors:
    - k8s-m1:6789
    - k8s-m2:6789
    - k8s-m3:6789
    path: /data/payeco/pv-${namespace}-${projectName}
    secretRef:
      name: ceph-rbd-secret
    user: admin
  persistentVolumeReclaimPolicy: Retain
"""

def pvcContext = """
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-${projectName}
  namespace: ${namespace}
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 200Gi
  selector:
    matchLabels:
      name: pv-${namespace}-${projectName}
  storageClassName: ""   
"""

def ingressOpenapiExample = """
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/cors-allow-headers: Accept,Authorization,DNT,X-CustomHeader,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,X-GHB-VERSION,origin,access-control-expose-headers,token
    nginx.ingress.kubernetes.io/cors-allow-methods: PUT, GET, POST, OPTIONS
    nginx.ingress.kubernetes.io/cors-allow-origin: '*'
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: 10m
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
  name: ingress-openapi
  namespace: ${namespace}
spec:
  rules:
  - host: openapi.${domainName}
    http:
      paths:
      - path: /gateway
        backend:
          serviceName: api-gateway
          servicePort: 8140
  tls:
  - hosts:
    - openapi.${domainName}
    secretName: ingress-tls-${namespace}-openapi
"""

node('jenkins-master') {
    def mvnHome
    stage('Preparation') {
        git branch: gitBranch, credentialsId: gitCredential, url: gitUrl
        mvnHome = tool 'M3'
    }

    stage('Maven build') {
        sh "'${mvnHome}/bin/mvn' -Dfile.encoding=utf-8 -Dmaven.test.failure.ignore clean package ${mvnArgs} -U -P publish -Dmaven.pmd.skip=true -Dmaven.test.skip=true"
        archive 'target/*.jar'

        def remote = [:]
        remote.name = "jenkins-docker"
        remote.host = "10.123.125.2"
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
            writeFile encoding: 'UTF-8', file: "run.sh", text: runScriptContext
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
            writeFile encoding: 'UTF-8', file: "service-${projectName}.yaml", text: serviceContext
            writeFile encoding: 'UTF-8', file: "pvc-${projectName}.yaml", text: pvcContext
            writeFile encoding: 'UTF-8', file: "pv-${projectName}.yaml", text: pvContext

            if (webui == "enable") {
                def ingressData
                def ingressName = sh(script: "/opt/k8s/bin/kubectl -n ${namespace} get ingress |grep ingress-openapi | awk '{print \$1}'", returnStdout: true).trim()
                if (ingressName.length() == 0) {
                    ingressData = readYaml text: ingressOpenapiExample
                } else {
                    ingressData = readYaml text: sh(script: "/opt/k8s/bin/kubectl -n ${namespace} get ingress ingress-openapi -o yaml", returnStdout: true)
                }
                def uPath = [path: "/${uriPath}", backend: [serviceName: "${projectName}", servicePort: port.toInteger()]]
                def uriList = []
                for (item in ingressData.spec.rules[0].http.paths) {
                    uriList.add(item.path.substring(1, item.path.length()))
                }
                if (!uriList.contains(uriPath)) {
                    ingressData.spec.rules[0].http.paths.add(uPath)
                }

                writeYaml encoding: 'UTF-8', file: "ingress-openapi.yaml", data: ingressData
            }
            sh "if [ -e /mnt/mycephfs/data/payeco/pv-${namespace}-${projectName} ];then echo '${projectName} already exist!' ;else sudo mkdir /mnt/mycephfs/data/payeco/pv-${namespace}-${projectName};fi"
            sh "/opt/k8s/bin/kubectl apply -f ."
        }
    }

    stage('Health check') {
        sleep(30)
        def stepSec = 15
        def failureTimes = 0
        while (true) {
            def healthState = sh(script: "/opt/k8s/bin/kubectl -n ${namespace} get pod  | grep ${projectName} | awk '{print \$2}'", returnStdout: true).trim().split('/')
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

