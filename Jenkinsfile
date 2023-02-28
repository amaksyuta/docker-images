// Show changes since last Failed build
def showChangesSinceLastFailed() {
  def changes = "Changes:\n"
  build = currentBuild
  while(build != null && build.result != 'SUCCESS') {
      changes += "In ${build.id}:\n"
      for (changeLog in build.changeSets) {
        for(entry in changeLog.items) {
          for(file in entry.affectedFiles) {
            changes += "* ${file.path}\n"
          }
        }
      }
  build = build.previousBuild
  }
  echo changes
}

// List changes since last build in case builds not SUCCESS
def getChangesSinceLastBuild() {
    def changes = []
    def build = currentBuild
    while (build != null && build.result != 'SUCCESS') {
        changes += (build.changeSets.collect { changeSet ->
            (changeSet.items.collect { item ->
                (item.affectedFiles.collect { affectedFile ->
                    affectedFile.path
                }).flatten()
            }).flatten()
        }).flatten()
        build = build.previousBuild
    }
    return changes.unique()
}

// Get GitHub changes related to files and folders
def getChangesSinceLastFailed() 
{
    def changes = []
    def build = currentBuild.getPreviousBuild()
    while(build.result != 'SUCCESS') {
        changes += listChangedBuildObjects(build)
        build = build.getPreviousBuild()
    }
    return changes.unique()
}

// Check changes depending on previous build resuls and return it in array
def getLastChanges() {
    def changes = []
    if(currentBuild.changeSets.size() > 0) {
        if(currentBuild.getPreviousBuild().result != 'SUCCESS') {
            current = getChangesSinceLastBuild()
            last = getChangesSinceLastFailed()
            result = [current, last].flatten().findAll{it}
            changes = result
        }
        else {
            changes = getChangesSinceLastBuild()
        }
    }
    else {
        if(currentBuild.getPreviousBuild().result != 'SUCCESS') {
            changes = getChangesSinceLastBuild()
        }
        else {
            changes = []
        }
    }
    return changes.unique()
}

// Get last short commit hash from GitHub
def getLastCommitHash() {
    def lastCommit = powershell(script: "git rev-parse --short HEAD", returnStdout: true).trim()
    return lastCommit
}

// Wrapper for each stage and more clear output jenkins log
def echoBanner(def ... msgs) {
   echo createBanner(msgs)
}

def errorBanner(def ... msgs) {
   error(createBanner(msgs))
}

def createBanner(def ... msgs) {
   return """
      ===================================================
            ${msgFlatten(null, msgs).join("\n        ")}
      ===================================================
   """
}

// Wrapper Flatten function
// NOTE: works well on all nested collections except a Map
def msgFlatten(def list, def msgs) {
   list = list ?: []
   if (!(msgs instanceof String) && !(msgs instanceof GString)) {
       msgs.each { msg ->
           list = msgFlatten(list, msg)
       }
   }
   else {
       list += msgs
   }
   return  list
}

def parallelTasks(labels, changes, branch) 
{
  // Check if we have any changes. Skipp the build in case no any changes
  if(! changes.isEmpty()){
    println("[INFO]: changes NOT emptyList")
  }
  else{
    println("[WARNING]: changes emptyList")
    currentBuild.result = 'SUCCESS'
    return
  }
  // labels for Jenkins node types we will build on
  def builders = [:]
  def winchanges = []
  def linuxchanges = []
  for(item in changes){
    if (item ==~ /.*linux.*|.*ubuntu.*/){
      linuxchanges += item
    }
    else{
      winchanges += item
    }
  }
  for (x in labels){
    def label = x // Need to bind the label variable before the closure - can't do 'for (label in labels)'
      // Create a map to pass in to the 'parallel' step so we can fire all the builds at once
      builders[label] = {
      if(label == 'windocker'){
        if(! winchanges.isEmpty()){
          println("[INFO]: winchanges NOT emptyList")
          println(winchanges)
        }
        else{
          println("[WARNING]: winchanges emptyList")
          currentBuild.result = 'SUCCESS'
          return
        }
        node(label){
          //------------- Stages windocker
          //------------- Start
          unstash name: 'buildSources'
          stage("Stage: Check environment on ${label} Node"){
            echoBanner("***** Check environment on ${label} START *****")
              script{
                powershell("""
                  & ${WORKSPACE}\\check-env.ps1
                  """)
              }
          }
            
          stage("Stage: Artifactory Auth on ${label} Node"){
            echoBanner("***** Artifactory Auth stage on ${label} START *****")
                script {
                  echo "[INFO]: DOCKERUSER is ${env.DOCKER_USER}"
                  echo "[INFO]: ARTIFACTORY URL is ${env.ARTIFACTORY_URL}"
                  powershell("""
                  & ${WORKSPACE}\\auth.ps1 -apitoken "${env.DOCKER_PASS}" -username "${env.DOCKER_USER}" -repourl "${env.ARTIFACTORY_URL}"
                  """)
                }   
          }
            
          stage("Stage: Build on ${label} Node"){
            echoBanner("***** Build images on ${label} START *****")
            withCredentials([usernamePassword(credentialsId: 'ccbuild-artifactory-api', usernameVariable: 'artfuser', passwordVariable: 'artftoken')]){
              script{
                def artfurl = "https://artifactory.teradyne.com"
                powershell("""
                & ${WORKSPACE}\\build-image.ps1 -branch "${branch}" -changes "${winchanges}" -githash "${githash}" -workdir "$WORKSPACE" -artfuser "${env.DOCKER_USER}" -artftoken "${env.DOCKER_PASS}" -artfurl "${artfurl}"
                """)
                println("Build on ${label} label node")
              }
            }
          }
            
          stage("Stage: Deploy on ${label} Node"){
            echoBanner("***** Deploy images on ${label} START *****")
            script {
              powershell("""
              & ${WORKSPACE}\\deploy-image.ps1 -branch "${branch}" -changes "${changes}" -githash "${githash}" -workdir "$WORKSPACE"
              """)
            }
          }    
          //------------- Stages
          //------------- End for windocker
        }
      }
      else{
        node(label){
        if(! linuxchanges.isEmpty()){
          println("[INFO]: linuxchanges NOT emptyList")
          println(linuxchanges)
        }
        else{
          println("[WARNING]: linuxchanges emptyList")
          currentBuild.result = 'SUCCESS'
          return
        }
          //------------- Stages windocker-linux
          //------------- Start for linuxchanges
          unstash name: 'buildSources'
          stage("Stage: Check environment on ${label} Node"){
            echoBanner("***** Check environment on ${label} START *****")
            script{
              powershell("""
              & ${WORKSPACE}\\check-env.ps1
              """)
            }
          }

          stage("Stage: Switch daemon to Linux mode on ${label} Node"){
            echoBanner("***** Switch daemon to Linux mode on ${label} START *****")
              script {
					      powershell("""
                & ${WORKSPACE}\\switch-daemon.ps1 -mode "linux"
                """)
                println(linuxchanges)
              }
          }

          stage("Stage: Artifactory Auth on ${label} Node"){
            echoBanner("***** Artifactory Auth stage on ${label} START *****")
              //withCredentials([usernamePassword(credentialsId: '82a08425-7324-4fa2-9eaf-cd67b5ac57a5', usernameVariable: 'dockeruser', passwordVariable: 'dockerpasswd')]){            
                  script {
                    echo "[INFO]: DOCKERUSER is ${env.DOCKER_USER}"
                    echo "[INFO]: ARTIFACTORY URL is ${ARTIFACTORY_URL}"
                    powershell("""
                    & ${WORKSPACE}\\auth.ps1 -username "${env.DOCKER_USER}" -apitoken "${env.DOCKER_PASS}" -repourl "${env.ARTIFACTORY_URL}"
                    """)
                  }
              //}
          }

          stage("Stage: Build on ${label} Node"){
            echoBanner("***** Build images on ${label} START *****")
            withCredentials([usernamePassword(credentialsId: 'ccbuild-artifactory-api', usernameVariable: 'artfuser', passwordVariable: 'artftoken')]){
              script{
                def artfurl = "https://artifactory.teradyne.com"
                powershell("""
                & ${WORKSPACE}\\build-image.ps1 -branch "${branch}" -changes "${linuxchanges}" -githash "${githash}" -workdir "$WORKSPACE" -artfuser "${artfuser}" -artftoken "${artftoken}" -artfurl "${artfurl}"
                """)
                println("Build on ${label} label node")
              }
            }
          }

          stage("Stage: Deploy on ${label} Node"){
            echoBanner("***** Deploy images on ${label} START *****")
              script {
                powershell("""
                & ${WORKSPACE}\\deploy-image.ps1 -branch "${branch}" -changes "${changes}" -githash "${githash}" -workdir "$WORKSPACE"
                """)
              }
          }

          stage("Stage: Switch daemon to Windows mode on ${label} Node"){
            echoBanner("***** Switch daemon to Windows mode on ${label} START *****")
            script {
					        powershell("""
                  & ${WORKSPACE}\\switch-daemon.ps1 -mode "windows"
                  """)
              }
          }     
          //------------- Stages
          //------------- End for windocker-linux
        } 
      }
    }
  }
  parallel builders
}

def buildChanges(propertyFile, changes) {
    def properties = readProperties file: propertyFile
    def agents = properties.keySet()

    for (def agent : agents) {
        def folders = properties[agent].split(',')
        for (def folder : folders) {
            echo "Building ${folder} on ${agent}"
            def workspace = pwd()
            node(agent) {
                stage("Checkout code on ${agent}") {
                    checkout([$class: 'GitSCM', 
                          branches: [[name: 'main']], 
                          doGenerateSubmoduleConfigurations: false, 
                          extensions: [], 
                          submoduleCfg: [], 
                          userRemoteConfigs: [[url: 'https://github.com/amaksyuta/docker-images.git']]])
                }
                stage("Build ${folder} on ${agent}") {
                    sh "cd ${folder} && <build command>"
                }
            }
        }
    }
    def changeList = changes.split(",")
    echo "Changes to build: ${changeList}"
    for (def change : changeList) {
        for (def agent : agents) {
            def folders = properties[agent].split(',')
            if (folders.contains(change)) {
                echo "Building ${change} on ${agent}"
                def workspace = pwd()
                node(agent) {
                    stage("Checkout code on ${agent}") {
                        checkout([$class: 'GitSCM', 
                        branches: [[name: 'main']], 
                        doGenerateSubmoduleConfigurations: false, 
                        extensions: [], 
                        submoduleCfg: [], 
                        userRemoteConfigs: [[url: 'https://github.com/amaksyuta/docker-images.git']]])
                    }
                    stage("Build ${change} on ${agent}") {
                        sh "cd ${change} && <build command>"
                    }
                    
                }
            }
        }
    }
}



def labels = ['windocker', 'mainnode']


pipeline {
    agent {
        label "windocker || mainnode"
    }
    stages {
        stage('Build on Linux') {
            agent {
                label "mainnode"
            }
          
            steps {
                script {
                    sh "cd ${WORKSPACE}/ && ls -la"
                }
            }
        }
        stage('Build on Windows') {
            agent {
                label "windocker"
            }
            steps {
                script {
                    bat "cd %WORKSPACE%\\ && dir"
                }
            }
        }
    }
}

