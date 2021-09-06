//Groovy Pipeline
node () { //node('worker_node')
   properties([
      parameters([
           gitParameter(branchFilter: 'origin/(.*)', defaultValue: 'development', name: 'BRANCH', type: 'PT_BRANCH'),
           string(defaultValue: '', name: 'VERSION', trim: true),
           choice(choices: ['DEV', 'QA' , 'PROD'], name: 'ENVIRONMENT'),
           string(defaultValue: 'dapplnl064', name: 'SERVER', trim: true),
           booleanParam(defaultValue: false,  name: 'ROLLBACK'),
      ]),
      disableConcurrentBuilds()
   ])
   
   def repoUrl = 'https://github.com/d-synchronized/spring-boot-ci-cd-demo.git'
   def previousPomVersion = ''
   def tagVersionCreated = ''
   try {
      stage('Checkout Source Code') { 
         echo "***Checking out source code from repo url ${repoUrl},branchName ${params.BRANCH}***"
             //bat "git config user.name 'Dishant Anand'"
             //bat "git config user.email d.synchronized@gmail.com"
          
         checkout([$class: 'GitSCM', 
                    branches: [[name: "*/${params.BRANCH}"]], 
                    extensions: [], 
                    userRemoteConfigs: [[credentialsId: 'github-credentials', url: "${repoUrl}"]]])
      }
      
      stage('Build & Deploy') {
         DEPLOY_TO_PROD = "${params.ENVIRONMENT}"  == 'PROD' ? true : false
         DEPLOY_TO_QA = "${params.ENVIRONMENT}" == 'QA' ? true : false
         DEPLOY_TO_DEV = "${params.ENVIRONMENT}"  == 'DEV' ? true : false
         
         VERSION_SET = "${params.VERSION}" == '' ? false : true
         
         if(DEPLOY_TO_DEV || VERSION_SET){
            echo "*******Build & Deploy, Version Set  ${VERSION_SET} , Environment ${params.ENVIRONMENT}********"         
             
            echo "Building SNAPSHOT Artifact"
            bat([script: 'mvn clean install deploy']) 
            
            echo "Dropping SNAPSHOT from the version"
            bat "mvn versions:set -DremoveSnapshot -DgenerateBackupPoms=false"
            
            echo "Building RELEASE Artifact"
            bat([script: 'mvn clean install deploy'])
         } else{
              echo "*******Skipping Build & Deploy, Version Set  ${VERSION_SET} , Environment ${params.ENVIRONMENT}********"
          }     
     }
      
     
     stage('Publish To Application/Web Server') {
         pom = readMavenPom file: 'pom.xml'
         echo "artifact is ${pom.artifactId}"
         echo "artifact is ${pom.version}"
     }
     
       currentBuild.result = 'SUCCESS'
       if("${WITH_ERROR}" == 'true'){
          error("Build failed because of this and that..")
       }
   } catch(Exception err) {
      echo "Error occurred while running the job '${env.JOB_NAME}'"
      currentBuild.result = 'FALIURE'
      //revertParentPOM("${previousPomVersion}")
      if("${params.RELEASE}" == 'true'){
         deleteTag("${tagVersionCreated}")
      }
   } finally {
       //deleteDir()
       echo '***************************************************'
       echo '***************************************************'
       echo '****POST******BUILD*****ACTION*********START*******'
       //mail to: 'd.synchronized@gmail.com', cc: 'vision4cloud@gmail.com,d.xcption13@gmail.com', bcc: 'slayer4cloud@gmail.com', 
       //     body: "Status for ${env.JOB_NAME} (${env.JOB_URL}) is ${currentBuild.result}", 
       //     subject: "Status of pipeline : ${currentBuild.fullDisplayName}"
       emailext attachLog: true, 
                body: 'Status for ${env.JOB_NAME} (${env.JOB_URL}) is ${currentBuild.result}', 
                subject: 'Status of pipeline : ${currentBuild.fullDisplayName',
                to: 'd.synchronized@gmail.com,d.xcption13@gmail.com'
       echo '****POST******BUILD*****ACTION*********END*********'
       echo '***************************************************'
       echo '***************************************************'
   }
   
}

def deleteTag(String tagVersionCreated) { 
      echo "deleting the TAG ${tagVersionCreated}"
      withCredentials([usernamePassword(credentialsId: 'github-account', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
          bat "git push --delete https://${env.GIT_USERNAME}:${env.GIT_PASSWORD}@github.com/d-synchronized/ci-cd-demo.git ${tagVersionCreated}"
      }
}
   
def revertParentPOM(String previousPomVersion) {
      echo "reverting pom version to ${previousPomVersion}"
      bat "mvn -U versions:set -DnewVersion=${previousPomVersion}"
      withCredentials([usernamePassword(credentialsId: 'github-account', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
          bat "git add pom.xml"
          bat "git commit -m \"Reverting pom version from ${NEW_VERSION} to ${projectVersion} \""
          bat "git push https://${env.GIT_USERNAME}:${env.GIT_PASSWORD}@github.com/d-synchronized/ci-cd-demo.git HEAD:${BRANCH}"
      }
}
