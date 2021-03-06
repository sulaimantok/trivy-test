stage('Configure') {
    abort = false
    inputConfig = input id: 'InputConfig', message: 'Docker registry and configuration', parameters: [string(defaultValue: 'https://index.docker.io/v1/', description: 'URL of the docker registry for staging images before analysis', name: 'dockerRegistryUrl', trim: true), string(defaultValue: 'docker.io', description: 'Hostname of the docker registry', name: 'dockerRegistryHostname', trim: true), string(defaultValue: 'anchore28/trivy-test', description: 'Name of the docker repository', name: 'dockerRepository', trim: true), credentials(credentialType: 'com.cloudbees.plugins.credentials.impl.UsernamePasswordCredentialsImpl', defaultValue: 'docker', description: 'Credentials for connecting to the docker registry', name: 'dockerCredentials', required: true)]

    for (config in inputConfig) {
        if (null == config.value || config.value.length() <= 0) {
          echo "${config.key} cannot be left blank"
          abort = true
        }
    }

    if (abort) {
        currentBuild.result = 'ABORTED'
        error('Aborting build due to invalid input')
    }
}

node {
  def app
  def dockerfile
  def repotag

  try {
    stage('Checkout') {
      // Clone the git repository
      checkout scm
      def path = sh returnStdout: true, script: "pwd"
      path = path.trim()
      dockerfile = path + "/Dockerfile"
    }

    stage('Build') {
      // Build the image and push it to a staging repository
      repotag = inputConfig['dockerRepository'] + ":${BUILD_NUMBER}"
      docker.withRegistry(inputConfig['dockerRegistryUrl'], inputConfig['dockerCredentials']) {
        app = docker.build(repotag)
        app.push()
      }
    }

    stage('Parallel') {
      parallel Test: {
        app.inside {
            sh 'echo "Dummy - tests passed"'
        }
      },
      Analyze: {
        repotag = inputConfig['dockerRepository'] + ":${BUILD_NUMBER}"
        sh(script: """trivy $repotag""")
      }
    }

  } 

  finally {
    stage('Cleanup') {
      // Delete the docker image and clean up any allotted resources
      sh script: "docker rmi " + repotag
    }
  }
}
