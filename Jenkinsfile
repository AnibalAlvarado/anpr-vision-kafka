pipeline {
    agent any

    environment {
        DOCKER_CLI_HINTS = "off"
    }

    stages {

        // =====================================================
        // 1️⃣ Leer entorno desde .env raíz
        // =====================================================
        stage('Leer entorno desde .env raíz') {
            steps {
                sh '''
                    echo "📂 Leyendo entorno desde .env raíz..."

                    DEPLOY_ENV=$(grep '^DEPLOY_ENV=' .env | cut -d '=' -f2 | tr -d '\\r\\n')

                    if [ -z "$DEPLOY_ENV" ]; then
                        echo "❌ No se encontró DEPLOY_ENV en .env"
                        exit 1
                    fi

                    echo "✅ Entorno detectado: $DEPLOY_ENV"
                    echo "DEPLOY_ENV=$DEPLOY_ENV" > env.properties
                    echo "ENV_DIR=DevOps/$DEPLOY_ENV" >> env.properties
                    echo "COMPOSE_FILE=DevOps/$DEPLOY_ENV/docker-compose.yml" >> env.properties
                    echo "ENV_FILE=DevOps/$DEPLOY_ENV/.env" >> env.properties
                '''

                script {
                    def props = readProperties file: 'env.properties'
                    env.DEPLOY_ENV = props['DEPLOY_ENV']
                    env.ENV_DIR = props['ENV_DIR']
                    env.COMPOSE_FILE = props['COMPOSE_FILE']
                    env.ENV_FILE = props['ENV_FILE']

                    echo """
                    ✅ Entorno detectado: ${env.DEPLOY_ENV}
                    📄 Compose: ${env.COMPOSE_FILE}
                    📁 Env file: ${env.ENV_FILE}
                    """
                }
            }
        }

        // =====================================================
        // 2️⃣ Limpiar imágenes y preparar entorno
        // =====================================================
        stage('Preparar entorno Docker') {
            steps {
                sh '''
                    echo "🧹 Limpiando imágenes no utilizadas..."
                    docker image prune -f || true

                    echo "🌐 Verificando red anpr-net-${DEPLOY_ENV} ..."
                    docker network create anpr-net-${DEPLOY_ENV} || echo "✅ Red ya existente"
                '''
            }
        }

        // =====================================================
        // 3️⃣ Desplegar stack de Kafka
        // =====================================================
        stage('Desplegar Kafka Stack') {
            steps {
                script {
                    if (env.DEPLOY_ENV == 'prod') {
                        echo "🚀 Despliegue remoto en AWS (Kafka - Producción)"

                        withCredentials([
                            sshUserPrivateKey(credentialsId: 'aws_ssh_key', keyFileVariable: 'SSH_KEY'),
                            string(credentialsId: 'aws_kafka_ip', variable: 'KAFKA_PROD_IP')
                        ]) {
                            sh '''
                                echo "🌍 Conectando al servidor Kafka en $KAFKA_PROD_IP"
                                ssh -o StrictHostKeyChecking=no -i $SSH_KEY ubuntu@$KAFKA_PROD_IP "
                                    set -e
                                    echo '📦 Actualizando repositorio Kafka...'
                                    cd /srv/anpr-vision-kafka || exit 1
                                    git pull

                                    echo '🐳 Desplegando Kafka stack en red anpr-net-prod...'
                                    docker network create anpr-net-prod || echo '🟡 Red ya existente'
                                    docker compose -f DevOps/prod/docker-compose.yml --env-file DevOps/prod/.env up -d --build --force-recreate --remove-orphans
                                "
                            '''
                        }
                    } else {
                        echo "🚀 Despliegue local (${env.DEPLOY_ENV})"
                        sh '''
                            cd $ENV_DIR
                            echo "🐳 Desplegando Kafka stack local (rebuild forzado)..."
                            docker compose --env-file .env -f docker-compose.yml up -d --build --force-recreate --remove-orphans
                        '''
                    }
                }
            }
        }
    }

    // =========================================================
    // Post actions
    // =========================================================
    post {
        success {
            echo "✅ Despliegue de Kafka completado correctamente (${env.DEPLOY_ENV})"
        }
        failure {
            echo "💥 Error durante el despliegue de Kafka (${env.DEPLOY_ENV})"
        }
    }
}

