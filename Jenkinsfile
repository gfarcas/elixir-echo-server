#!groovy

node {
  currentBuild.result = "SUCCESS"
  try {
    stage('Checkout') {
      checkout scm
    }

    stage('Unit Test') {
      docker.build("elixir-echo-server-unit:${env.BUILD_ID}", "-f unit-Dockerfile ./")
      unitOutput = sh (
        script: "docker run --rm --name unit elixir-echo-server-unit:${env.BUILD_ID}",
        returnStdout: true
      )
      if (!unitOutput.contains('0 failures'))
        error("Build failed because of: " + unitOutput)
    }

    stage('Build Docker') {
      docker.withRegistry('https://registry.hub.docker.com/', 'dockerhub') {
        def customImage = docker.build("gimsys/elixir-echo-server:${env.BUILD_ID}")
        customImage.push()
      }
    }

    stage('Stage') {
      try {
        //label 'stage'
        sh """
          ssh -o StrictHostKeyChecking=no -i /var/jenkins_home/id_rsa docker@${STAGE_SWARM_MANAGER} \
          "docker service create \
          --name=ees \
          --publish=6000:6000 \
          gimsys/elixir-echo-server:${env.BUILD_ID}"
          """
      }
      catch (Exception e) {
        print e
        sh """
          ssh -o StrictHostKeyChecking=no -i /var/jenkins_home/id_rsa docker@${STAGE_SWARM_MANAGER} \
          "docker service update \
          --image gimsys/elixir-echo-server:${env.BUILD_ID} \
          ees"
          """
      }
    }

    stage('Smoke Test Stage') {
      docker.build("elixir-echo-server-test:${env.BUILD_ID}", "-f test-Dockerfile ./")
      testOutput = sh (
        script: "docker run -e HOST='${STAGE_SWARM_MANAGER}' --rm --name test elixir-echo-server-test:${env.BUILD_ID}",
        returnStdout: true
      ).trim()
      if (testOutput != 'test')
        error("Build failed because the output should have been \"test\", but it was " + testOutput + " instead")
    }

    stage('Production') {
      node {
        //label 'prod'
        timeout(time: 1, unit: 'HOURS') {
          // this only works if you're the user "admin"
          input(message: 'Shall we deploy to Production?', submitter: 'admin')
        }
        try {
          sh """
            ssh -o StrictHostKeyChecking=no -i /var/jenkins_home/id_rsa docker@${PROD_SWARM_MANAGER} \
            "docker service create \
            --name=ees \
            --publish=6000:6000 \
            gimsys/elixir-echo-server:${env.BUILD_ID}"
            """
        }
        catch (Exception e) {
          print e
          sh """
            ssh -o StrictHostKeyChecking=no -i /var/jenkins_home/id_rsa docker@${PROD_SWARM_MANAGER} \
            "docker service update \
            --image gimsys/elixir-echo-server:${env.BUILD_ID} \
            ees"
            """
        }
      }
    }

    stage('Smoke Test Production') {
      docker.build("elixir-echo-server-test:${env.BUILD_ID}", "-f test-Dockerfile ./")
      testOutput = sh (
        script: "docker run -e HOST='${PROD_SWARM_MANAGER}' --rm --name test elixir-echo-server-test:${env.BUILD_ID}",
        returnStdout: true
      ).trim()
      if (testOutput != 'test')
        error("Build failed because the output should have been \"test\", but it was " + testOutput + " instead")
    }

  }
  catch (err) {
    currentBuild.result = "FAILURE"
    throw err
  }
}
