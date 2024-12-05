Vul onderstaande aan met de antwoorden op de vragen uit de readme.md file. Wil je de oplossingen file van opmaak voorzien? Gebruik dan [deze link](https://github.com/adam-p/markdown-here/wiki/Markdown-Cheatsheet) om informatie te krijgen over
opmaak met Markdown.

a)

Testserver:
  ubuntu en jenkins user toegang geven tot docker zonder sudo:
    sudo usermod -aG docker jenkins
    sudo usermod -aG docker ubuntu (in mijn geval: sudo usermod -aG docker student voor de ubuntu gebruiker)
    
Productieserver:
  ubuntu user toegang tot docker zonder sudo:
    sudo usermod -aG docker ubuntu


andere aanpassingen :
  1.
  dockerhub-credentials aangemaakt "dockerhub-credentials"
  ![image](https://github.com/user-attachments/assets/7a28cd22-8b04-4738-863d-f3f651856c4a)
  credentials voor authorisatie bij het pushen van de gemaakte docker file in de pipeline naar Docker Hub

  2.
  de file (onder ~/.ssh/key-opdracht6.pem) waarin mijn private key is opgeslaan van rechten veranderd:
  chmod 400 "mijn-file" deze heeft nu enkel lees rechten: -r-------- 1 student student 

  3.
  jenkins credentials aangemaakt "key-opdracht6":
    deze bevat de private key van de ec2 instance in aws
    ![image](https://github.com/user-attachments/assets/b7e327b9-92f6-46dc-a619-f95fcab8d203)

  4.
  docker plugin en de SSH Agent plugin geinstalleerd.


b)


Test deployment: test.jenkinsfile
Deze pipeline voert het volledige CI/CD-proces uit en publiceert de artifact naar Docker Hub. 

1. Cleanup:

  Verwijdert de huidige werkdirectory:
  deleteDir()

  
2. Clone Repository

  Cloont de calculator-app repository van GitHub:
  git branch: 'main', url: 'https://github.com/TimoHubner444/calculator-app-finished.git'

  
3. Install Dependencies

  Installeert NodeJS dependencies met npm install:
  sh 'npm install'

  
4. Build Artifact

  Bouwt een Docker-container voor de Node.js-applicatie:
  sh "docker build -t ${DOCKER_IMAGE_NAME}:${DOCKER_TAG} ."

  
5. Login to Docker Hub

  Logt in op Docker Hub met behulp van Jenkins credentials:
  withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
      sh "echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin"
  }

  
6. Push Artifact

  Tagt en pusht de Docker-image naar Docker Hub:
  sh "docker tag ${DOCKER_IMAGE_NAME}:${DOCKER_TAG} ${DOCKER_IMAGE_NAME}:${DOCKER_TAG}"
  sh "docker push ${DOCKER_IMAGE_NAME}:${DOCKER_TAG}"
  
7. Deployment

  Cleanup van oude containers en start een nieuwe container op poort 3000:
    echo "Checking for existing containers using port 3000..."
    sh '''
        CONTAINER_ID=$(docker ps -q --filter "publish=3000")
        if [ ! -z "$CONTAINER_ID" ]; then
            echo "Stopping container with ID $CONTAINER_ID"
            docker stop $CONTAINER_ID
            docker rm $CONTAINER_ID
        else
            echo "No containers using port 3000 found."
        fi
    '''
    
    
    echo "Running the Docker container from Docker Hub..."
    sh "docker run -d -p 3000:3000 ${DOCKER_IMAGE_NAME}:${DOCKER_TAG}"



productie deployment: production.jenkinsfile

1. Cleanup

  Verwijdert oude containers, netwerken, en ongebruikte Docker-images op de Productieserver via SSH:
  sshagent(['key-opdracht6']) {
      sh """
          ssh ${PROD_SERVER} 'docker rm -f ${CONTAINER_NAME} || true'
          ssh ${PROD_SERVER} 'docker image prune -af'
          ssh ${PROD_SERVER} 'docker volume prune -f'
      """
  }


2. Deploy Prod

Verwijdert de oude container (indien aanwezig) en downloadt de nieuwste versie van de Docker-image van Docker Hub:
  sshagent(['key-opdracht6']) {
      sh """
          ssh ${PROD_SERVER} 'docker rm -f ${CONTAINER_NAME} || true'
          ssh ${PROD_SERVER} 'docker pull ${DOCKER_IMAGE}'
      """
  }
3. Start Prod

Start de nieuwe container op de Productieserver op poort 80:
  sshagent(['key-opdracht6']) {
      sh """
          ssh ${PROD_SERVER} 'docker run -d --name ${CONTAINER_NAME} -p ${REMOTE_PORT}:3000 ${DOCKER_IMAGE}'
      """
  }
  
4. Test Prod

Controleert of de applicatie draait door een HTTP-statuscode 200 te valideren:
  def response = sh(script: "curl -s -o /dev/null -w '%{http_code}' http://3.91.218.68:${REMOTE_PORT}", returnStdout: true).trim()
  if (response == '200') {
      echo 'Applicatie draait succesvol op productie server.'
  } else {
      error "Applicatie is niet bereikbaar op de productie server. HTTP status: ${response}"
  }




succesvolle uitvoering test deployment (pipeline):
![image](https://github.com/user-attachments/assets/7d6a5d9b-64a9-4910-8f4d-42d050f99573)
![image](https://github.com/user-attachments/assets/4b57ed64-487a-4271-89f8-8bbc846246c2)
![image](https://github.com/user-attachments/assets/7cfcea1f-ce54-49cf-aff6-9f0ccdee2290)
![image](https://github.com/user-attachments/assets/2d99f971-8ecb-426e-87cf-0f94c811f6c8)
![image](https://github.com/user-attachments/assets/97778c09-4850-47fd-9a15-fd13fdc89729)



succesvolle uitvoering production deployment (pipeline):

![image](https://github.com/user-attachments/assets/1caf4e99-4dd6-4867-8d1b-ade16838861e)
![image](https://github.com/user-attachments/assets/ba47b239-4305-4688-b9cc-f0745f2e41b7)
![image](https://github.com/user-attachments/assets/ce62079c-dba5-46bb-b7e4-008b88d68296)
![image](https://github.com/user-attachments/assets/55fbacb4-1615-43b1-a259-35b856478303)
![image](https://github.com/user-attachments/assets/1f80c62d-8702-4ce9-8a83-0f9dbab4d864)






container status testserver:
![image](https://github.com/user-attachments/assets/074e0c66-473c-4ebb-aa93-c48dc8c42928)



container status productieserver:
![image](https://github.com/user-attachments/assets/f85497eb-d3c2-4e13-bc66-14f6d4335f59)


screenschot calculator-app op poort 3000(testserver):
![image](https://github.com/user-attachments/assets/acbce384-ac45-41fe-98b5-2b4c73456866)



screenshot van de calculator-app op poort 80 (Productieserver):
![image](https://github.com/user-attachments/assets/c818d4b0-6b56-4072-92ad-9bb83e697aac)

