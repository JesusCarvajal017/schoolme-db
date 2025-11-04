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

          echo "Branch: ${env.BRANCH_NAME} -> Dir: ${env.ENV_DIR}, Servicio: ${env.SERVICE}"
        }
      }
    }

    stage('Deploy DB') {
      steps {
        dir("${env.ENV_DIR}") {
          script {
            if (params.RESET_VOLUMES) {
              sh '''#!/bin/bash
set -e
docker compose -f "$COMPOSE_Y" --env-file "$ENV_FILE" down -v || true
'''
            }

            sh '''#!/bin/bash
set -euxo pipefail
docker compose -f "$COMPOSE_Y" --env-file "$ENV_FILE" pull "$SERVICE" || true
docker compose -f "$COMPOSE_Y" --env-file "$ENV_FILE" up -d "$SERVICE"
'''

            // Esperar healthcheck usando el nombre EXACTO del contenedor ($SERVICE)
            sh '''#!/bin/bash
set -euo pipefail
echo "Esperando a que $SERVICE esté healthy..."
for i in {1..30}; do
  state=$(docker inspect -f '{{json .State.Health.Status}}' "$SERVICE" 2>/dev/null || echo '"starting"')
  echo "Intento $i - Estado: $state"
  [[ "$state" == "\"healthy\"" ]] && break
  sleep 2
done
docker inspect -f '{{.State.Health.Status}}' "$SERVICE"
'''
          }
        }
      }
    }

    stage('Status & Logs') {
      when { expression { return params.SHOW_LOGS } }
      steps {
        dir("${env.ENV_DIR}") {
          sh '''#!/bin/bash
set -e
docker compose -f "$COMPOSE_Y" --env-file "$ENV_FILE" ps
echo "---- Logs ($SERVICE) ----"
docker compose -f "$COMPOSE_Y" --env-file "$ENV_FILE" logs --tail=80 "$SERVICE" || true
'''
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
