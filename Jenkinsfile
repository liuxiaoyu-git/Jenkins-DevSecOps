// Jenkinsfile
pipeline {
  agent { label 'maven' }


  stages {

    stage('Checkout') {
      steps {
       git url: "http://gogs-ocp-workshop.${JENKINS_APP_DOMAIN}/${JENKINS_GOGS_USER}/SecurityDemos.git"
      } // steps
    } // stage

    stage('Build') {
    steps {
         sh "mvn -Dmaven.test.skip=true clean package"
      } // steps
    } // stage

    stage('Run tests') {
    steps {
         sh "mvn test"
         junit 'target/surefire-reports/*.xml'
      } // steps

    } // stage

    stage('SonarQube Scan') {
      steps {
        sh "mvn sonar:sonar -Dsonar.host.url=http://sonarqube.ocp-workshop.svc:9000 -Dsonar.projectkey=${JENKINS_GOGS_USER}-ecommerce -Dsonar.projectName=\"${JENKINS_GOGS_USER} E-Commerce Project\""
      } // steps
    } // stage

    stage('Archive to nexus') {
      steps {
        sh "mvn --settings mvn.xml deploy -Dmaven.test.skip=true"
      } // steps
    } // stage

    stage('Build Image') {
      steps {
        sh "oc login -u ${JENKINS_GOGS_USER} -p openshift --insecure-skip-tls-verify ${JENKINS_OCP_API_ENDPOINT}"
        sh "oc new-build --name ecommerce --strategy=docker --binary -n ${JENKINS_GOGS_USER} || true"
        sh 'oc patch bc/ecommerce -p \'{"spec" : {"strategy" : { "dockerStrategy" : { "from": {"kind" : "DockerImage", "name" : "quayecosystem-quay-quay-enterprise.' + "${JENKINS_APP_DOMAIN}/admin/hardened-openjdk-golden:${JENKINS_GOGS_USER}" + '" }}}}}\' -n ${JENKINS_GOGS_USER}'
        sh "mkdir deploy || true"
        sh "cp target/spring-boot-angular-ecommerce-0.0.1-SNAPSHOT.jar deploy"
        sh "cp Dockerfile deploy"
        sh "oc set build-secret --pull bc/ecommerce quay -n basic-spring-boot-build -n ${JENKINS_GOGS_USER}"
        sh "oc start-build ecommerce --from-dir=deploy --follow --wait -n ${JENKINS_GOGS_USER}"
        //sh "oc start-build ecommerce --from-dir=deploy --follow --wait --build-arg BASE_IMAGE=registry.access.redhat.com/redhat-openjdk-18/openjdk18-openshift:1.0"
      } // steps
    } // stage

    stage('Push Image to Quay') {
      agent { label 'image-management' }
      steps {
        sh "oc login -u ${JENKINS_GOGS_USER} -p openshift --insecure-skip-tls-verify ${JENKINS_OCP_API_ENDPOINT}"
        sh 'skopeo --debug copy --src-creds="$(oc whoami)":"$(oc whoami -t)" --src-tls-verify=false --dest-tls-verify=false' + " --dest-creds=admin:admin123 docker://${JENKINS_INTERNAL_REGISTRY}/${JENKINS_GOGS_USER}/ecommerce:latest docker://quayecosystem-quay-quay-enterprise.${JENKINS_APP_DOMAIN}/admin/ecommerce:${JENKINS_GOGS_USER} || true"
      } // steps
    } //stage

    stage('OpenSCAP Scans') {
      agent { label 'master' }
      steps {

      script {
         def remote = [:]
         remote.name = "bastion"
         //remote.host = "bastion.${JENKINS_GUID}.openshiftworkshop.com"
         remote.host = "${JENKINS_BASTION}"
         remote.allowAnyHosts = true
         remote.user="${JENKINS_GOGS_USER}"
         remote.password="${JENKINS_SSH_PASSWORD}"

         sshCommand remote: remote, command: "oc login -u ${JENKINS_GOGS_USER} -p openshift --insecure-skip-tls-verify ${JENKINS_OCP_API_ENDPOINT}"
         sshCommand remote: remote, command: "sudo docker login -u ${JENKINS_GOGS_USER} -p " + '"$(oc whoami -t)"' + " ${JENKINS_INTERNAL_REGISTRY}"
         sshCommand remote: remote, command: "sudo docker pull ${JENKINS_INTERNAL_REGISTRY}/${JENKINS_GOGS_USER}/ecommerce:latest"
         sshCommand remote: remote, command: "sudo oscap-docker ${JENKINS_INTERNAL_REGISTRY}/${JENKINS_GOGS_USER}/ecommerce:latest xccdf eval --fetch-remote-resources --profile xccdf_org.ssgproject.content_profile_stig --report report.html /usr/share/xml/scap/ssg/content/ssg-rhel7-ds.xml || true"
         sshCommand remote: remote, command: "wget -O - https://www.redhat.com/security/data/oval/v2/RHEL7/rhel-7.oval.xml.bz2 | bzip2 --decompress > rhel-7.oval.xml"
         sshCommand remote: remote, command: "sudo oscap-docker ${JENKINS_INTERNAL_REGISTRY}/${JENKINS_GOGS_USER}/ecommerce:latest oval eval --report report-cve.html rhel-7.oval.xml"
         sshGet remote: remote, from: "/home/${JENKINS_GOGS_USER}/report.html", into: 'openscap-compliance-report.html', override: true
         sshGet remote: remote, from: "/home/${JENKINS_GOGS_USER}/report-cve.html", into: 'openscap-cve-report.html', override: true
         publishHTML([alwaysLinkToLastBuild: false, keepAll: false, reportDir: './', reportFiles: 'openscap-compliance-report.html', reportName: 'OpenSCAP Compliance Report', reportTitles: 'OpenSCAP Compliance Report'])
         publishHTML([alwaysLinkToLastBuild: false, keepAll: false, reportDir: './', reportFiles: 'openscap-cve-report.html', reportName: 'OpenSCAP Vulnerability Report', reportTitles: 'OpenSCAP Vulnerability Report'])
         archiveArtifacts 'openscap-compliance-report.html,openscap-cve-report.html'
        } // script
      } // steps
    } // stage

    stage('Deploy') {
      steps {
        sh "oc new-app ecommerce -n ${JENKINS_GOGS_USER} || true"
        sh "oc set env deployment/ecommerce JAVA_ARGS=/deployments/root.jar -n ${JENKINS_GOGS_USER}"
        sh "oc expose svc/ecommerce -n ${JENKINS_GOGS_USER} || true"
        sh "oc rollout status deployment/ecommerce -n ${JENKINS_GOGS_USER}"
      } // steps
    } // stage

    stage('OWASP ZAP Scan') {
      agent { label 'zap' }
      steps {
        script {
          sh "/zap/zap-baseline.py -r owasp-zap-baseline.html -t http://ecommerce.${JENKINS_GOGS_USER}.svc:8080/ -t http://ecommerce.${JENKINS_GOGS_USER}.svc:8080/api/products -t http://ecommerce.${JENKINS_GOGS_USER}.svc:8080/api/orders || true"
          sh "cp /zap/wrk/owasp-zap-baseline.html ."
          publishHTML([alwaysLinkToLastBuild: false, keepAll: false, reportDir: './', reportFiles: 'owasp-zap-baseline.html', reportName: 'OWASP ZAP Baseline Report', reportTitles: ''])
          archiveArtifacts 'owasp-zap-baseline.html'
        } // script
      } // steps
    } // stage

    stage('Configure Stage Project') {
      steps {
        script {
          sh "set +x ; oc login -u ${JENKINS_GOGS_USER} -p ${JENKINS_SSH_PASSWORD} --insecure-skip-tls-verify https://kubernetes.default.svc"
          sh "oc create is ecommerce -n ${JENKINS_GOGS_USER}-stage || true"
          sh "oc new-app ecommerce --image-stream=ecommerce --allow-missing-images --allow-missing-imagestream-tags -n ${JENKINS_GOGS_USER}-stage || true"
          sh "oc expose deployment/ecommerce -n ${JENKINS_GOGS_USER}-stage || true"
          sh "oc expose deployment/ecommerce --port 8080 -n ${JENKINS_GOGS_USER}-stage || true"
          sh "oc expose svc/ecommerce -n ${JENKINS_GOGS_USER}-stage || true"
        } // script
      }// steps
    } // stage

    stage('Promote to Stage?') {
      steps {
        timeout(time: 7, unit: 'DAYS') {
          input message: "Do you want to deploy to ${JENKINS_GOGS_USER}-stage?"
        } // timeout
        sh "oc tag ${JENKINS_GOGS_USER}/ecommerce:latest ${JENKINS_GOGS_USER}-stage/ecommerce:latest"
        sh "oc rollout status deployment/ecommerce -n ${JENKINS_GOGS_USER}-stage"
      } // steps
    } // stage

    stage('Configure Prod Project') {
      steps {
        script {
          sh "set +x ; oc login -u ${JENKINS_GOGS_USER} -p ${JENKINS_SSH_PASSWORD} --insecure-skip-tls-verify https://kubernetes.default.svc"
          sh "oc create is ecommerce -n ${JENKINS_GOGS_USER}-prod || true"
          sh "oc new-app ecommerce --image-stream=ecommerce --allow-missing-images --allow-missing-imagestream-tags -n ${JENKINS_GOGS_USER}-prod || true"
          sh "oc expose deployment/ecommerce -n ${JENKINS_GOGS_USER}-prod || true"
          sh "oc expose deployment/ecommerce --port 8080 -n ${JENKINS_GOGS_USER}-prod || true"
          sh "oc expose svc/ecommerce -n ${JENKINS_GOGS_USER}-prod || true"
        } // script
      }// steps
    } // stage

    stage('Promote to Prod?') {
      steps {
        timeout(time: 7, unit: 'DAYS') {
          input message: "Do you want to deploy to ${JENKINS_GOGS_USER}-prod?"
        } // timeout
        sh "oc tag ${JENKINS_GOGS_USER}-stage/ecommerce:latest ${JENKINS_GOGS_USER}-prod/ecommerce:latest"
        sh "oc rollout status deployment/ecommerce -n ${JENKINS_GOGS_USER}-prod"
      } // steps
    } // stage

  } // stages

} // pipeline
