pipeline {
  agent any
  options { timestamps(); ansiColor('xterm') }

  parameters {
    booleanParam(name: 'RESET_VOLUMES', defaultValue: false, description: 'Borrar volúmenes del entorno antes de subir (DB en cero).')
    booleanParam(name: 'SHOW_LOGS', defaultValue: true, description: 'Mostrar últimos logs del contenedor.')
  }

  stages {
    stage('Checkout & Tools') {
      steps {
        checkout scm
        sh 'docker version && docker compose version'
      }
    }

    stage('Seleccionar entorno') {
      steps {
        script {
          def map = [
            'main'   : [dir: 'environments/main',   svc: 'schoolme-db-main',    envfile: '.env-main',    compose: 'docker-compose.yml'],
            'staging': [dir: 'environments/staging',svc: 'schoolme-db-staging', envfile: '.env-staging', compose: 'docker-compose.yml'],
            'qa'     : [dir: 'environments/qa',     svc: 'schoolme-db-qa',      envfile: '.env-qa',      compose: 'docker-compose.yml'],
            'dev'    : [dir: 'environments/dev',    svc: 'schoolme-db-dev',     envfile: '.env-dev',     compose: 'docker-compose.yml']
          ]
          if (!map.containsKey(env.BRANCH_NAME)) {
            error "Branch '${env.BRANCH_NAME}' no está mapeado. Usa main, staging, qa o dev."
          }
          env.ENV_DIR   = map[env.BRANCH_NAME].dir
          env.SERVICE   = map[env.BRANCH_NAME].svc
          env.ENV_FILE  = map[env.BRANCH_NAME].envfile
          env.COMPOSE_Y = map[env.BRANCH_NAME].compose
        }
      }
    }

    stage('Deploy DB') {
      steps {
        dir("${env.ENV_DIR}") {
          script {
            if (params.RESET_VOLUMES) {
              sh "docker compose -f ${env.COMPOSE_Y} --env-file ${env.ENV_FILE} down -v || true"
            }
            sh """
                set -euxo pipefail
                docker compose -f ${env.COMPOSE_Y} --env-file ${env.ENV_FILE} pull ${env.SERVICE} || true
                docker compose -f ${env.COMPOSE_Y} --env-file ${env.ENV_FILE} up -d ${env.SERVICE}
                """
            sh """
                echo "Esperando a que ${env.SERVICE} esté healthy..."
                for i in {1..30}; do
                state=$(docker inspect -f '{{json .State.Health.Status}}' ${env.SERVICE} 2>/dev/null || echo '\"starting\"')
                echo "Intento $i - Estado: $state"
                [ "$state" = "\"healthy\"" ] && break
                sleep 2
                done
                docker inspect -f '{{.State.Health.Status}}' ${env.SERVICE}
                """
          }
        }
      }
    }

    stage('Status & Logs') {
      when { expression { return params.SHOW_LOGS } }
      steps {
        dir("${env.ENV_DIR}") {
          sh """
            docker compose -f ${env.COMPOSE_Y} --env-file ${env.ENV_FILE} ps
            echo "---- Logs (${env.SERVICE}) ----"
            docker compose -f ${env.COMPOSE_Y} --env-file ${env.ENV_FILE} logs --tail=80 ${env.SERVICE} || true
            """
        }
      }
    }
  }

  post {
    always {
      echo "Listo: ${env.BRANCH_NAME} -> ${env.SERVICE}"
    }
  }
}
