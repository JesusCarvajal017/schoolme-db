pipeline {
  agent any
  options {
    skipDefaultCheckout(true)   // evita el checkout anónimo por defecto
    timestamps()
  }

  environment {
    REPO_URL       = 'https://github.com/JesusCarvajal017/schoolme-db'
    GIT_CREDENTIAL = 'validation_user' // ID de credencial Global (Secret text con tu PAT)
  }

  parameters {
    booleanParam(name: 'RESET_VOLUMES', defaultValue: false, description: 'Borrar volúmenes del entorno antes de subir (DB en cero).')
    booleanParam(name: 'SHOW_LOGS',     defaultValue: true,  description: 'Mostrar últimos logs del contenedor.')
  }

  stages {

    stage('Checkout & Tools') {
      steps {
        // 🔐 Checkout autenticado (evita rate limit)
        checkout([
          $class: 'GitSCM',
          branches: [[name: "*/${env.BRANCH_NAME}"]],
          userRemoteConfigs: [[
            url: "${env.REPO_URL}",
            credentialsId: "${env.GIT_CREDENTIAL}"
          ]]
        ])

        // Detectar docker compose (v2) o docker-compose (v1)
        script {
          env.COMPOSE_CMD = sh(
            script: '''
              if docker compose version >/dev/null 2>&1; then
                echo "docker compose"
              elif command -v docker-compose >/dev/null 2>&1; then
                echo "docker-compose"
              else
                echo ""
              fi
            ''',
            returnStdout: true
          ).trim()
          if (!env.COMPOSE_CMD) {
            error "Docker Compose no está instalado en el agente. Instala 'docker compose' (plugin v2) o 'docker-compose' (v1)."
          }
        }

        // Mostrar versiones
        sh '''
          set -e
          docker version
        '''
        sh '''
          set -e
          ${COMPOSE_CMD} version
        '''
      }
    }

    stage('Seleccionar entorno') {
      steps {
        script {
          def map = [
            'main'   : [dir: 'environments/main',    svc: 'pg-main',    envfile: '.env-main',    compose: 'docker-compose.yml'],
            'staging': [dir: 'environments/staging', svc: 'pg-staging', envfile: '.env-staging', compose: 'docker-compose.yml'],
            'qa'     : [dir: 'environments/qa',      svc: 'pg-qa',      envfile: '.env-qa',      compose: 'docker-compose.yml'],
            'dev'    : [dir: 'environments/dev',     svc: 'pg-dev',     envfile: '.env-dev',     compose: 'docker-compose.yml']
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
${COMPOSE_CMD} -f "$COMPOSE_Y" --env-file "$ENV_FILE" down -v || true
'''
            }

            sh '''#!/bin/bash
set -euxo pipefail
${COMPOSE_CMD} -f "$COMPOSE_Y" --env-file "$ENV_FILE" pull "$SERVICE" || true
${COMPOSE_CMD} -f "$COMPOSE_Y" --env-file "$ENV_FILE" up -d "$SERVICE"
'''

            // Esperar healthcheck del contenedor
            sh '''#!/bin/bash
set -euo pipefail
echo "Esperando a que $SERVICE esté healthy..."
ok=0
for i in {1..30}; do
  state=$(docker inspect -f '{{json .State.Health.Status}}' "$SERVICE" 2>/dev/null || echo '"starting"')
  echo "Intento $i - Estado: $state"
  if [[ "$state" == "\"healthy\"" ]]; then ok=1; break; fi
  sleep 2
done
docker inspect -f '{{.State.Health.Status}}' "$SERVICE" || true
[ "$ok" = "1" ] || { echo "Timeout esperando healthcheck de $SERVICE"; exit 1; }
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
${COMPOSE_CMD} -f "$COMPOSE_Y" --env-file "$ENV_FILE" ps
echo "---- Logs ($SERVICE) ----"
${COMPOSE_CMD} -f "$COMPOSE_Y" --env-file "$ENV_FILE" logs --tail=120 "$SERVICE" || true
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
