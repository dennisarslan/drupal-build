 pipeline {
  agent any
  options {
    buildDiscarder(logRotator(numToKeepStr: '10', artifactNumToKeepStr: '10'))
  }
  parameters {
    booleanParam(name: 'debug', defaultValue: false, description: 'Should I enable debug mode for verbose logging?')
  }
  environment {
    DOCKER_CREDS = credentials('amazeeiojenkins-dockerhub-password')
    COMPOSE_PROJECT_NAME = 'denpal'
    COMPOSE_PROJECT_NAMO = "denpal-${BUILD_ID}"
  }
  stages {
    stage('Docker login') {
      steps {
        sh '''
        env
        export COMPOSE_PROJECT_NAME="denpal-${BUILD_ID}"
        docker login --username amazeeiojenkins --password $DOCKER_CREDS
        '''
      }
    }
    stage('Docker Build') {
      steps {
        sh '''
        export COMPOSE_PROJECT_NAME="denpal-${BUILD_ID}"
        docker-compose config -q
        docker network prune -f && docker network inspect amazeeio-network >/dev/null || docker network create amazeeio-network
        COMPOSE_PROJECT_NAME=denpal docker-compose down
        COMPOSE_PROJECT_NAME=denpal docker-compose up -d --build "$@"
        '''
      }
    }
    stage('Waiting 10 seconds') {
      steps {
        sh """
        sleep 10s
        """
      }
    }
    stage('Debug') {
      when { expression { return params.debug } }
      steps {
        sh """
        export COMPOSE_PROJECT_NAME="denpal-${BUILD_ID}"
        docker-compose ps
        docker network list
        docker ps | head
        docker images | head
        docker-compose ps
        docker logs denpal_cli_1
        docker logs denpal_mariadb_1
        docker inspect denpal_php_1  | grep -i DB_
        docker exec denpal_cli_1 mysql -hmariadb -udrupal -pdrupal
        docker-compose logs
        """
      }
    }
    stage('Verification') {
      steps {
        sh '''
        export COMPOSE_PROJECT_NAME="denpal-${BUILD_ID}"
        docker-compose exec -T cli drush status
        docker-compose exec -T cli curl http://nginx:8080 -v
        curl -v http://localhost:10000/
        curl -v http://localhost:10001/
        if [ $? -eq 0 ]; then
          echo "OK!"
        else
          echo "FAIL"
          /bin/false
        fi
        '''
      }
    }
    stage('Docker Push') {
      steps {
        sh '''
        tag=$(git describe --abbrev=0 --tags)
        branch=$(git describe --all --contains --abbrev=4)
        echo "Branch: "
        echo $branch

        docker tag denpal:latest amazeeiodevelopment/denpal:latest
        docker tag denpal:latest amazeeiodevelopment/denpal:$tag
        docker push amazeeiodevelopment/denpal:latest
        docker push amazeeiodevelopment/denpal:$tag

        docker tag denpal_nginx:latest amazeeiodevelopment/denpal_nginx:latest
        docker tag denpal_nginx:latest amazeeiodevelopment/denpal_nginx:$tag
        docker push amazeeiodevelopment/denpal_nginx:latest
        docker push amazeeiodevelopment/denpal_nginx:$tag

        docker tag denpal_php:latest amazeeiodevelopment/denpal_php:latest
        docker tag denpal_php:latest amazeeiodevelopment/denpal_php:$tag
        docker push amazeeiodevelopment/denpal_php:latest
        docker push amazeeiodevelopment/denpal_php:$tag
        '''
      }
    }
  }
  post {
    always {
      script {
        currentBuild.setDescription("CLI: ${params.cli} - NGINX: ${params.nginx} - PHP: ${params.php}")
      }
      echo 'I will always say Hello again!'
    }
  }
}
