pipeline {
    agent any
    
    triggers {
        // En producción se corre diario, en otras ramas puedes quitarlo o mantenerlo
        cron('H H * * *') 
    }
    
    environment {
        AWS_CREDENTIALS = credentials('mis-credenciales-aws')
        S3_BUCKET       = 's3://tu-bucket-backups-jenkins'
        BACKUP_DIR      = '/var/jenkins_home/local_backups'
        IMAGE_NAME      = "jenkins-backup-tool:${BUILD_ID}"
        
        // 💡 CONCEPTO CLAVE: Detectamos la rama actual automáticamente
        // env.BRANCH_NAME nos da 'int', 'pre' o 'pro' directamente en pipelines multirama
        ENV_BRANCH      = "${env.BRANCH_NAME}"
        FILE_NAME       = "backup_jenkins_${env.BRANCH_NAME}_build_${BUILD_ID}.tar.gz"
    }
    
    stages {
        stage('Validar Entorno Real') {
            steps {
                // Validación de seguridad para simular flujos reales
                echo "🚀 Iniciando proceso de respaldo automatizado"
                echo "🌍 Entorno detectado: [${ENV_BRANCH.toUpperCase()}]"
            }
        }
        
        stage('Pre-bake Backup Image') {
            steps {
                sh "docker build -t ${IMAGE_NAME} -f Dockerfile.backup ."
            }
        }
        
        stage('Execute Local Backup') {
            steps {
                sh "mkdir -p ${BACKUP_DIR}/${ENV_BRANCH}"
                sh """
                    docker run --rm \
                    -v jenkins_home:/var/jenkins_home \
                    -v ${BACKUP_DIR}/${ENV_BRANCH}:/backup_destination \
                    ${IMAGE_NAME} \
                    tar -czf /backup_destination/${FILE_NAME} -C /var/jenkins_home jobs/ users/ plugins/ config.xml
                """
            }
        }
        
        stage('Upload to AWS S3') {
            steps {
                // 💡 SQUEEZE DE AWS: Subimos el archivo a la subcarpeta del entorno correspondiente
                // Ejemplo: s3://tu-bucket-backups-jenkins/pro/archivo.tar.gz
                sh """
                    docker run --rm \
                    -v ${BACKUP_DIR}/${ENV_BRANCH}:/backup_source \
                    -e AWS_ACCESS_KEY_ID=${AWS_CREDENTIALS_USR} \
                    -e AWS_SECRET_ACCESS_KEY=${AWS_CREDENTIALS_PSW} \
                    ${IMAGE_NAME} \
                    aws s3 cp /backup_source/${FILE_NAME} ${S3_BUCKET}/${ENV_BRANCH}/
                """
            }
        }
        
        stage('Cleanup Images') {
            steps {
                sh "docker rmi ${IMAGE_NAME}"
            }
        }
    }
}
