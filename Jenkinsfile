properties([buildDiscarder(logRotator(artifactNumToKeepStr: '5', numToKeepStr: '5'))])


node {
    
    def commitId
    def nomDeLimage = 'g1-rockpedia'

    stage('checkout') {
        git branch: 'master', url: 'https://github.com/ALudwig57/rockpedia.git'
        commitId = sh(returnStdout: true, script: 'git rev-parse HEAD').trim()
    }

    // si jenkins est dans un docker, il faut rajouter -u root
    docker.image('openjdk:8').inside('-u root -e MAVEN_OPTS="-Duser.home=./"') {
       
       stage('check tools') {
            parallel(
                    java: {
                        sh "java -version"
                    },
                    maven: {
                        sh "chmod +x mvnw"
                        sh "./mvnw -version"
                    }
            )
        }

      stage('clean') {
            sh "./mvnw clean"
        }

        stage('backend tests') {
            try {
                sh "./mvnw verify -Psurefire"
            } catch (err) {
                throw err
            } finally {
                junit '**/target/surefire-reports/TEST-*.xml'
            }
        }

        stage('packaging') {
            sh "./mvnw verify -DskipTests"
            archiveArtifacts artifacts: '**/target/*.jar', fingerprint: true
        }

        stage('preparing docker') {
            sh "cp -R src/main/docker/* target/"
        }
        
        stage('Analyse SonarQube') {
            withSonarQubeEnv('sonarqube') {
                sh "./mvnw sonar:sonar"
            }
        }
    }


    
    stage('build docker') {
        try {
            def dockerImage = docker.build("m2gi/${nomDeLimage}:${commitId}", 'target')
        } catch (err) {
            throw err
        } finally {
            sh "docker tag m2gi/${nomDeLimage}:${commitId} m2gi/${nomDeLimage}:latest"
        }
    }

    stage('redeploy'){
        // remplacement de la version par le numéro de commit
        sh "sed -i -E 's=m2gi/${nomDeLimage}:[0-9]+\\.[0-9]+\\.[0-9]+(-SNAPSHOT)?=m2gi/${nomDeLimage}:${commitId}=g' src/main/docker/app.yml"
        sh "docker stack deploy -c src/main/docker/app.yml ${nomDeLimage}"
    }
}
