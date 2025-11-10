pipeline {
  agent any
  options {
    skipDefaultCheckout(true)   // evita el checkout anónimo por defecto
    timestamps()
  }

  environment {
    REPO_URL       = 'https://github.com/JesusCarvajal017/schoolme-db'
    GIT_CREDENTIAL = 'validation_user' // ID de credencial Global (Secret text con tu PAT)
    STACK_NAME     = 'schoolme-db'
  }

  parameters {
    booleanParam(name: 'RESET_VOLUMES', defaultValue: false, description: 'Borrar volúmenes del entorno antes de subir (DB en cero).')
    booleanParam(name: 'SHOW_LOGS',     defaultValue: true,  description: 'Mostrar últimos logs del contenedor.')
    booleanParam(name: 'RUN_ALL',       defaultValue: false, description: 'Levantar las 4 bases (main, staging, qa, dev) en una sola stack.')
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

          if (!params.RUN_ALL) {
            if (!map.containsKey(env.BRANCH_NAME)) {
              error "Branch '${env.BRANCH_NAME}' no está mapeado. Usa main, staging, qa o dev."
            }
            env.ENV_DIR   = map[env.BRANCH_NAME].dir
            env.SERVICE   = map[env.BRANCH_NAME].svc
            env.ENV_FILE  = map[env.BRANCH_NAME].envfile
            env.COMPOSE_Y = map[env.BRANCH_NAME].compose
            echo "Branch: ${env.BRANCH_NAME} -> Dir: ${env.ENV_DIR}, Servicio: ${env.SERVICE}"
          } else {
            // modo stack
            env.ENV_DIR_ALL = 'environments'
            env.ALL_COMPOSE = 'docker-compose.all.yml'
            echo "RUN_ALL=true -> Se levantarán pg-main, pg-staging, pg-qa, pg-dev desde ${env.ENV_DIR_ALL}/${env.ALL_COMPOSE}"
          }
        }
      }
    }

    stage('Deploy DB') {
      steps {
        script {
          // --- MODO STACK: levantar TODOS a la vez ---
          if (params.RUN_ALL) {
            dir("${env.ENV_DIR_ALL}") {
              if (params.RESET_VOLUMES) {
                sh '''#!/usr/bin/env bash
                  set -e
                  ${COMPOSE_CMD} -p "$STACK_NAME" -f "$ALL_COMPOSE" down -v || true
                '''
              }

              sh '''#!/usr/bin/env bash
                set -euxo pipefail
                ${COMPOSE_CMD} -p "$STACK_NAME" -f "$ALL_COMPOSE" pull pg-main pg-staging pg-qa pg-dev || true
                ${COMPOSE_CMD} -p "$STACK_NAME" -f "$ALL_COMPOSE" up -d pg-main pg-staging pg-qa pg-dev
              '''

              // Esperar health de TODOS usando container ID real
              sh '''#!/usr/bin/env bash
                set -euo pipefail
                services=(pg-main pg-staging pg-qa pg-dev)

                for s in "${services[@]}"; do
                  cid=$(${COMPOSE_CMD} -p "$STACK_NAME" -f "$ALL_COMPOSE" ps -q "$s")
                  [ -n "$cid" ] || { echo "No se encontró contenedor para $s"; exit 1; }

                  echo "Esperando a que $s ($cid) esté healthy..."
                  ok=0
                  for i in {1..40}; do
                    state=$(docker inspect -f '{{.State.Health.Status}}' "$cid" 2>/dev/null || echo "starting")
                    echo "[$s] Intento $i - Estado: $state"
                    if [[ "$state" == "healthy" ]]; then ok=1; break; fi
                    sleep 2
                  done
                  docker inspect -f '{{.State.Health.Status}}' "$cid" || true
                  [[ "$ok" == "1" ]] || { echo "Timeout esperando healthcheck de $s"; exit 1; }
                done
              '''

              if (params.SHOW_LOGS) {
                sh '''#!/usr/bin/env bash
                  set -e
                  ${COMPOSE_CMD} -p "$STACK_NAME" -f "$ALL_COMPOSE" ps
                  echo "---- Logs (pg-main) ----";    ${COMPOSE_CMD} -p "$STACK_NAME" -f "$ALL_COMPOSE" logs --tail=60 pg-main    || true
                  echo "---- Logs (pg-staging) ----"; ${COMPOSE_CMD} -p "$STACK_NAME" -f "$ALL_COMPOSE" logs --tail=60 pg-staging || true
                  echo "---- Logs (pg-qa) ----";      ${COMPOSE_CMD} -p "$STACK_NAME" -f "$ALL_COMPOSE" logs --tail=60 pg-qa      || true
                  echo "---- Logs (pg-dev) ----";     ${COMPOSE_CMD} -p "$STACK_NAME" -f "$ALL_COMPOSE" logs --tail=60 pg-dev     || true
                '''
              }
            }
            return // no sigas con el flujo por servicio
          }
          // --- FIN MODO STACK ---

          // --- MODO POR RAMA (tal cual lo tenías) ---
          dir("${env.ENV_DIR}") {
            if (params.RESET_VOLUMES) {
              sh '''#!/bin/bash
                set -e
                ${COMPOSE_CMD} -p "$STACK_NAME" -f "$COMPOSE_Y" --env-file "$ENV_FILE" down -v || true
              '''
            }

            sh '''#!/bin/bash
              set -euxo pipefail
              ${COMPOSE_CMD} -p "$STACK_NAME" -f "$COMPOSE_Y" --env-file "$ENV_FILE" pull "$SERVICE" || true
              ${COMPOSE_CMD} -p "$STACK_NAME" -f "$COMPOSE_Y" --env-file "$ENV_FILE" up -d "$SERVICE"
            '''

            // Esperar healthcheck del contenedor (versión corregida)
            sh '''#!/usr/bin/env bash
              set -euo pipefail
              # Tomar el container ID si existe; si no, usar el nombre del servicio
              cid=$(${COMPOSE_CMD} -p "$STACK_NAME" -f "$COMPOSE_Y" --env-file "$ENV_FILE" ps -q "$SERVICE" || true)
              target="${cid:-$SERVICE}"

              echo "Esperando a que $target esté healthy..."
              ok=0
              for i in {1..30}; do
                state=$(docker inspect -f '{{.State.Health.Status}}' "$target" 2>/dev/null || echo "starting")
                echo "Intento $i - Estado: $state"
                if [[ "$state" == "healthy" ]]; then ok=1; break; fi
                sleep 2
              done

              docker inspect -f '{{.State.Health.Status}}' "$target" || true
              if [[ "$ok" != "1" ]]; then
                echo "Timeout esperando healthcheck de $target"
                exit 1
              fi
            '''
          }
        }
      }
    }

    stage('Status & Logs') {
      when { expression { return params.SHOW_LOGS } }
      steps {
        script {
          if (params.RUN_ALL) {
            dir('environments') {
              sh '''#!/usr/bin/env bash
                set -e
                ${COMPOSE_CMD} -p "$STACK_NAME" -f "$ALL_COMPOSE" ps
                echo "---- Logs (pg-main) ----";    ${COMPOSE_CMD} -p "$STACK_NAME" -f "$ALL_COMPOSE" logs --tail=120 pg-main    || true
                echo "---- Logs (pg-staging) ----"; ${COMPOSE_CMD} -p "$STACK_NAME" -f "$ALL_COMPOSE" logs --tail=120 pg-staging || true
                echo "---- Logs (pg-qa) ----";      ${COMPOSE_CMD} -p "$STACK_NAME" -f "$ALL_COMPOSE" logs --tail=120 pg-qa      || true
                echo "---- Logs (pg-dev) ----";     ${COMPOSE_CMD} -p "$STACK_NAME" -f "$ALL_COMPOSE" logs --tail=120 pg-dev     || true
              '''
            }
          } else {
            dir("${env.ENV_DIR}") {
              sh '''#!/bin/bash
                set -e
                ${COMPOSE_CMD} -p "$STACK_NAME" -f "$COMPOSE_Y" --env-file "$ENV_FILE" ps
                echo "---- Logs ($SERVICE) ----"
                ${COMPOSE_CMD} -p "$STACK_NAME" -f "$COMPOSE_Y" --env-file "$ENV_FILE" logs --tail=120 "$SERVICE" || true
              '''
            }
          }
        }
      }
    }
  }

  post {
    always {
      echo "Listo: ${env.BRANCH_NAME} -> ${ params.RUN_ALL ? 'STACK (pg-main, pg-staging, pg-qa, pg-dev)' : env.SERVICE }"
    }
  }
}

