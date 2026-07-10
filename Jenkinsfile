pipeline {
    agent any
    
    triggers {
        cron('H H * * *') 
    }
    
    environment {
        AWS_ACCESS_KEY_ID     = credentials('aws-access-key-id')
        AWS_SECRET_ACCESS_KEY = credentials('aws-secret-access-key')
        
        S3_BUCKET       = 's3://backup-jenkinsbucket'
        BACKUP_DIR      = '/var/jenkins_home/local_backups'
        IMAGE_NAME      = "jenkins-backup-tool:${BUILD_ID}"
        ENV_BRANCH      = "${env.BRANCH_NAME}"
        FILE_NAME       = "backup_jenkins_${env.BRANCH_NAME}_build_${BUILD_ID}.tar.gz"
    }
    
    stages {
        stage('Instalar Docker CLI Portable') {
            steps {
                echo "📦 Configurando Docker CLI estático y portátil..."
                sh '''
                    mkdir -p bin
                    if [ ! -f bin/docker ]; then
                        echo "⬇️ Descargando binario oficial de Docker..."
                        curl -fsSL https://download.docker.com/linux/static/stable/x86_64/docker-24.0.7.tgz | tar -xz --strip-components=1 -C bin docker/docker
                        chmod +x bin/docker
                    fi
                '''
            }
        }

        stage('Validar Entorno Real') {
            steps {
                echo "🚀 Iniciando proceso de respaldo automatizado"
                echo "🌍 Entorno detectado: [${ENV_BRANCH.toUpperCase()}]"
                // Verificación de seguridad para confirmar que el binario responde
                sh "bin/docker --version"
            }
        }
        
        stage('Pre-bake Backup Image') {
            steps {
                sh "bin/docker build -t ${IMAGE_NAME} -f Dockerfile.backup ."
            }
        }
        
        stage('Execute Local Backup') {
            steps {
                sh "mkdir -p ${BACKUP_DIR}/${ENV_BRANCH}"
                sh """
                    bin/docker run --rm \
                    -v jenkins_home:/var/jenkins_home \
                    -v ${BACKUP_DIR}/${ENV_BRANCH}:/backup_destination \
                    ${IMAGE_NAME} \
                    tar -czf /backup_destination/${FILE_NAME} -C /var/jenkins_home jobs/ users/ plugins/ config.xml
                """
            }
        }
        
        stage('Upload to AWS S3') {
            steps {
                sh """
                    bin/docker run --rm \
                    -v ${BACKUP_DIR}/${ENV_BRANCH}:/backup_source \
                    -e AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID} \
                    -e AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY} \
                    ${IMAGE_NAME} \
                    aws s3 cp /backup_source/${FILE_NAME} ${S3_BUCKET}/${ENV_BRANCH}/
                """
            }
        }
        
        stage('Cleanup Images') {
            steps {
                sh "bin/docker rmi ${IMAGE_NAME}"
            }
        }
    }
}
