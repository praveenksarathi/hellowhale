#!/bin/bash +x
//TODO - Make SVN and GIT Checkout steps perfect with Jenkins way. Do not use Shell way.
def temporaryDockerRegistry = tempDockerRegistry
def permanentDockerRegistry = permDockerRegistry


// This update is for Bug ID : 531
def httpProxy = ''
def httpsProxy = ''
  
node {
  echo "Parameters"
  echo "Temp Docker Registry: ${tempDockerRegistry}"
  echo "Permenant Docker Registry: ${permDockerRegistry}"
  echo "Docker Repository: ${dockerRepo}"
  echo "Docker Image Name: ${dockerImageName}"
  

  
  echo "SCM Type: ${scmSourceRepo}"
  echo "SCM Path: ${scmPath}"
  echo "SCM User: ${scmUsername}"
  echo "MEC User: ${userId}"
  
  echo "HTTP Proxy: ${httpProxy}"
  echo "HTTPS Proxy: ${httpsProxy}"  
  
  //-------------------------------------- big bucket
  //To escape all Special Charecters in a given input string password
  def pwdstr = scmPassword
  scmPassword = pwdstr.replaceAll( /([^a-zA-Z0-9])/, '\\\\$1' )
  
  def usrstr = scmUsername
  scmUsername = usrstr.replaceAll( /([^a-zA-Z0-9])/, '\\\\$1' )
  
  def pwdstr2 = scmPassword
  def usrstr2 = scmUsername
  scmPassword = pwdstr2.replaceAll( /([@])/, '%40' )
  scmUsername = usrstr2.replaceAll( /([@])/, '%40' ) 
  //----------------------------------------

  stage('Code Pickup') {
    echo "Source Code Repository Type : ${scmSourceRepo}"
    echo "Source Code Repository Path : ${scmPath}"
    
    if("${scmSourceRepo}".toUpperCase()=='SVN'){
        //Not a perfect solution. Reimplement with SVN Step or checkout Step
        echo 'SCM is SVN'
        sh "svn co --username ${scmUsername} --password ${scmPassword} ${scmPath} ."
        
    } else if("${scmSourceRepo}".toUpperCase()=='GIT' || "${scmSourceRepo}".toUpperCase()=='GITHUB'){
        //Not a perfect Solution. Reimplement with Git Step or checkout Step
        echo 'SCM is GIT Based'
        //Check if the Git clone URL already has username and password in it.
        //def strToMatch = "${scmPath}"
        def extractedCredentialsFromPath="";
        if(scmPath.indexOf("@")>-1 && scmPath.indexOf("//") >-1) {
            extractedCredentialsFromPath=scmPath.substring(scmPath.indexOf("//")+2,scmPath.indexOf("@"))
        }
      
        if(extractedCredentialsFromPath && extractedCredentialsFromPath.length()>0) {
          echo "Looks like the Username or Password or Both are found in the GIT Repo path itself"
          extractedCredentialsFromPath=extractedCredentialsFromPath.replaceAll( /([^a-zA-Z0-9])/, '\\\\$1' )
          extractedCredentialsFromPath=extractedCredentialsFromPath.replaceAll( /([@])/, '%40' )
          echo "Extracted Credentials and ecapped: ${extractedCredentialsFromPath}"
          echo "User & Password: ${scmUsername}\\:${scmPassword}"
          if ("${extractedCredentialsFromPath}" == "${scmUsername}" + "\\:" + "${scmPassword}") {
            echo "Username and password already present in url. No need to append"
          } else if("${extractedCredentialsFromPath}" == "${scmUsername}" && !scmPath.startsWith("ssh://")) {
            echo "Username already present in url. Just handle the password" 
            scmPath = scmPath.substring(0,scmPath.indexOf("@"))+":" + scmPassword +scmPath.substring(scmPath.indexOf("@"), scmPath.length());
          } else if("${extractedCredentialsFromPath}" == "${scmUsername}" && scmPath.startsWith("ssh://")) {
            echo "Username and password already present in url. Since this is SSH, no need to append password" 
            //Password should not be appended for SSH. It should be passed through SSHPASS only
          } else {
            echo "Something wrong. The credentials in the URL did not match with the values passed" 
          }
        } else {
          echo 'Credentials not found in path. Frame it'
          //SSH based git should not append password. It will not work. The password should be passed through sshpass
          if(scmPath.startsWith("ssh://")){
              scmPath = scmPath.substring(0, scmPath.indexOf("//")+2) + scmUsername + "@" +scmPath.substring(scmPath.indexOf("//")+2, scmPath.length());
          } else {         
              scmPath = scmPath.substring(0, scmPath.indexOf("//")+2) + scmUsername + ":" + scmPassword + "@" +scmPath.substring(scmPath.indexOf("//")+2, scmPath.length());
          }  
        }
      
        //echo "GIT PATH: ${scmPath}"
        try {
            //If we use git clone, it will not clone in the same path if we rebuild the pipeline
            sh 'ls -a | xargs rm -fr'
        } catch (error) {
        }
      
        if(scmPath.startsWith("ssh://")){
          if(httpsProxy != null && httpProxy!=null && httpsProxy.length()>0 && httpProxy.length()>0){
            echo "Looks like this Jenkins behind Proxy"
            sh "export https_proxy=${httpsProxy} && export http_proxy=${httpProxy} && sshpass -p ${scmPassword}   git clone --recursive ${scmPath} ."
          } else {
            echo "Looks like this Jenkins is not behind Proxy"
            sh "sshpass -p ${scmPassword}   git clone --recursive ${scmPath} ."
          }            
        } else {
            //Need this line for the recursive sub module clone to work without asking credentials
            sh "git config --global credential.helper 'cache --timeout=120'"
            if(httpsProxy != null && httpProxy!=null && httpsProxy.length()>0 && httpProxy.length()>0) {
              echo "Looks like this Jenkins behind Proxy"
              sh "export https_proxy=${httpsProxy} && export http_proxy=${httpProxy} && git clone --recursive ${scmPath} ."
            } else {
              echo "Looks like this Jenkins is not behind Proxy"
              sh "git clone --recursive ${scmPath} ."
            }  
            //Need this line for the recursive sub module clone to work without asking credentials -- Clear the cache. 
            sh "git credential-cache exit"
        } 
        
        //The below solutions not working with username and password
        //git "${scmPath}" //Alternate option
        //checkout scm: [$class:'GitSCM', userRemoteConfigs: [[url: scmPath ]]]
    } else {
        error 'Unknown Source code repository. Only GIT and SVN are supported'
    }
  } 
  //---------------------------------------
  
  //Preparing for Build & Package
  def appModuleSeperated = fileExists 'app'
  def testModuleSeperated = fileExists 'test'
  def appPath = ''
  def testPath = ''  
  if (appModuleSeperated) {
    echo 'There is a directory called app and hence assuming that /app is the working directory for application.'
    appPath='app/'
  } else {
    echo 'There is no directory named app. Hence assuming that the current directory is the working directory.'
    appPath = ''
  }

  if (testModuleSeperated) {
    echo 'There is a directory called test and hence assuming that /test is the integration test automation suite.'
    testPath = 'test/'  
  } else {
    echo 'There is no directory named test. Hence assuming that there is no integration testing automation suite.'
    testPath = ''  
  }
  
  def isBuildAndPackageRequired = true
  def buildDockerFile = appPath + 'Dockerfile.build'
  def distDockerFile = appPath + 'Dockerfile.dist'
  if (fileExists(buildDockerFile) && fileExists(distDockerFile)) {
    echo 'It looks like this application is compiler based application and hence there is a seperate dockerfile found for compile, build and packaging.'
    isBuildAndPackageRequired = true;    
  } else if (appPath + fileExists('Dockerfile')) {
    echo 'It looks like there is only one docker file. May be this is interpreter based application technology.'
    isBuildAndPackageRequired = false;
    distDockerFile = appPath + 'Dockerfile'
  } else {
    echo 'Dockerfile not found under ' + appPath
  }
  def appWorkingDir = (appPath=='') ? '.' : appPath.substring(0, appPath.length()-1)
  //End of Preparation for Build and Package
  
  //TODO - Tune it later. Dirty solution to identify the Jenkins generated artifacts for Nexus
        sh 'echo Nexus>Nexus.txt'
  //Dirty solution ends.
  
  //---------------------------------------
  if(isBuildAndPackageRequired){
    echo 'Compile and Package runs as seperate stage due to this app requirements.'
    stage('Compile, Unit Test & Package') {
      echo 'Working Directory for Docker Build file: ' + appWorkingDir
      echo "Build Tag Name: ${dockerRepo}/${dockerImageName}-build:${env.BUILD_NUMBER}"
      echo "Build params: --file ${buildDockerFile} ${appWorkingDir}"
      
      appCompileAndPackageImg = docker.build("${dockerRepo}/${dockerImageName}-build:${env.BUILD_NUMBER}", "--file ${buildDockerFile} ${appWorkingDir}")      
      
      //Reading the CMD from Docker file and would be executing within the container. This is due to the behaviour of this plugin
      //TODO - Danger zone. This approach is not based on grammer. So in case if the CMD is not shell (if exec) or ENTRYPOINT is given, this approach would not work
      def dockerCMD = readFile buildDockerFile
      echo dockerCMD.substring(dockerCMD.indexOf('CMD')+3, dockerCMD.length())      
      appCompileAndPackageImg.inside('--net=host') {        
        sh dockerCMD.substring(dockerCMD.indexOf('CMD')+3, dockerCMD.length())
      }
      //End of Danger Zone code      
    }
  } else {
    echo 'There is no Compile and Package as seperate step'
  }
  //---------------------------------------
  
  if("${stage}".toUpperCase() == 'BUILD') {
    echo 'The requested stage is Build only. Hence pushing the successful image into temporary repo'
    //---------------------------------------
    // We are pushing to a private Temporary Docker registry as this is just Build case.
    // 'docker-registry-login' is the username/password credentials ID as defined in Jenkins Credentials.
    // This is used to authenticate the Docker client to the registry.
    //docker.withRegistry("http://${temporaryDockerRegistry}/", 'docker-registry-login') {
    //withDockerRegistry([credentialsId: 'docker-registry-login', url: temporaryDockerRegistry]) {
   docker.withRegistry("http://${temporaryDockerRegistry}/", 'docker-registry-login') {
      def pcImg
      stage('Dockerization & Stage') {
        // Let us tag and push the newly built image. Will tag using the image name provided. No need of Docker hostname as it appends itself.
        echo "BUILD TAG USED FOR IMAGE : ${env.BUILD_NUMBER}";
        pcImg = docker.build("${dockerRepo}/${dockerImageName}:${env.BUILD_NUMBER}", "--network host --file ${distDockerFile} ${appWorkingDir} ")
        pcImg.push("${env.BUILD_NUMBER}");
      }
    }   
  } else if ("${stage}".toUpperCase() == 'DEPLOY') {
    echo 'The requested stage is Deploy. Hence without certifying, the image is pushed to permanent repo'
    //withDockerRegistry([credentialsId: 'docker-registry-login', url: permanentDockerRegistry]) {
    docker.withRegistry("https://${permanentDockerRegistry}/", 'docker-registry-login') {
      def pcImg
      stage('Dockerization & Publish') {
        // Let us tag and push the newly built image. Will tag using the image name provided. No need of Docker hostname as it appends itself.
        echo "DEPLOY TAG USED FOR IMAGE is always latest";
        pcImg = docker.build("${dockerRepo}/${dockerImageName}:${env.BUILD_NUMBER}", "--network host --file ${distDockerFile} ${appWorkingDir} ")
        pcImg.push('latest');
      }
    }    
  } else if ("${stage}".toUpperCase() == 'CERTIFY'){
    echo 'The requested stage is Certify. Hence just publishing to temporary repo and provisioning sandbox'
    docker.withRegistry("http://${temporaryDockerRegistry}/", 'docker-registry-login') {
      def pcImg
      stage('Certify') {
        // Let us tag and push the newly built image. Will tag using the image name provided. No need of Docker hostname as it appends itself.
        //pcImg = docker.build("${temporaryDockerRegistry}/${dockerRepo}/${dockerImageName}:${env.BUILD_NUMBER}", "--file ${distDockerFile} ${appWorkingDir}")
        echo "CERTIFY TAG USED FOR IMAGE : ${env.BUILD_NUMBER}";
        pcImg = docker.build("${dockerRepo}/${dockerImageName}:${env.BUILD_NUMBER}", "--network host --file ${distDockerFile} ${appWorkingDir} ")
        pcImg.push("${env.BUILD_NUMBER}");
      }
    }   
  }
  //---------------------------------------
  
  
}
