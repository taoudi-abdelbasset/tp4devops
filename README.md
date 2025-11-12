# **Compte Rendu TP4 – CI/CD with Jenkins & Docker**  
**Student**: Abdelbasset Taoudi  
**Course**: Ingénierie des Infrastructures Bigdata et Cloud 2025/2026  
**Professor**: Pr. Kamal EL GUEMMAT  

---

## **Part 1: Setup & Freestyle Job**

### 1. **Jenkins Installation**
- Installed Jenkins on Ubuntu using official guide  
- Accessed via `http://localhost:8080`  
- Unlocked with initial admin password  

![alt text](image.png)

---

### 2. **Project Setup (`tp4devops`)**
```html
<!-- index.html -->
<h1>Welcome BDCC</h1>
```
```dockerfile
# Dockerfile
From nginx
COPY index.html /usr/share/nginx/html
EXPOSE 80
```

![alt text](image-1.png)

---

### 3. **GitHub Repository**
- Created: `https://github.com/taoudi-abdelbasset/tp4devops`  
- Commands:
```bash
git init
git add .
git commit -m "tp4 v1"
git remote add origin https://github.com/taoudi-abdelbasset/tp4devops.git
git push origin master
```

![alt text](image-2.png)

---

### 4. **Freestyle Job: `job01 tp4devops`**
- **Source Code Management**: Git → URL + branch `master`  
- **Build Steps**:
  ```bash
  docker login -u taoudiabdelbasset -p <token>
  docker build -t taoudiabdelbasset/tp4devops .
  docker push taoudiabdelbasset/tp4devops
  ```
- **Deploy Script**:
  ```bash
  docker ps --filter "publish=8081" -q | xargs -r docker rm -f
  docker run -d -p 8081:80 taoudiabdelbasset/tp4devops
  ```
**Config**
![alt text](image-3.png)
![alt text](image-4.png)
- run every minute
![alt text](image-5.png)
- env
![alt text](image-6.png)
- docker image build
![alt text](image-7.png)
![alt text](image-8.png)

---

### 5. **Automatic Trigger on Push**
- Changed `index.html` → `git commit -m "changed index"` → `git push`  
- **Jenkins auto-triggered** → New image → Deployed on `localhost:8081`

![alt text](<Screenshot from 2025-11-12 22-39-04.png>)
![alt text](<Screenshot from 2025-11-12 22-41-47.png>)
![alt text](<Screenshot from 2025-11-12 22-51-21.png>)

---

## **Part 2: Pipeline Job (`job2tp4v2`)**

### **5 Stages Pipeline**

![alt text](<Screenshot from 2025-11-12 22-52-44.png>)

```groovy
pipeline {
    agent any

    environment {
        registry = 'taoudiabdelbasset/tp4devops'
        registryCredential = 'dockerhub'
        dockerImage = ''
    }

    stages {
        // STAGE 1: Cloning Git
        stage('Cloning Git') {
            steps {
                echo "Cloning from GitHub..."
                git branch: 'master',
                    url: 'https://github.com/taoudi-abdelbasset/tp4devops.git'
            }
        }

        // STAGE 2: Building Image
        stage('Building Image') {
            steps {
                script {
                    echo "Building image: ${registry}:${BUILD_NUMBER}"
                    dockerImage = docker.build("${registry}:${BUILD_NUMBER}", "--no-cache .")
                    sh "docker tag ${registry}:${BUILD_NUMBER} ${registry}:latest"
                }
            }
        }

        // STAGE 3: Test Image
        stage('Test Image') {
            steps {
                script {
                    echo "Running tests on image..."
                    def containerId = sh(
                        script: "docker run -d -p 8083:80 ${registry}:${BUILD_NUMBER}",
                        returnStdout: true
                    ).trim()

                    // Wait for container to start
                    sleep 5

                    // Simple test: check if HTTP 200
                    def status = sh(
                        script: "curl -s -o /dev/null -w '%{http_code}' http://localhost:8083",
                        returnStdout: true
                    ).trim()

                    if (status == "200") {
                        echo "Test PASSED: Web server is running!"
                    } else {
                        error "Test FAILED: HTTP status = ${status}"
                    }

                    // Clean up test container
                    sh "docker rm -f ${containerId}"
                }
            }
        }

        // STAGE 4: Publish Image
        stage('Publish Image') {
            steps {
                script {
                    echo "Pushing to Docker Hub..."
                    docker.withRegistry('https://index.docker.io/v1/', registryCredential) {
                        dockerImage.push()
                        dockerImage.push('latest')
                    }
                }
            }
        }

        // STAGE 5: Deploy Image
        stage('Deploy Image') {
            steps {
                echo "Deploying to production: http://localhost:8082"
                sh '''
                    # Remove old container on port 8082
                    docker ps -a --filter "publish=8082" -q | xargs -r docker rm -f
                    
                    # Run final image
                    docker run -d -p 8082:80 ${registry}:${BUILD_NUMBER}
                '''
            }
        }
    }

    post {
        success {
            echo "PIPELINE SUCCESS! 5 Stages Completed."
            echo "App is live at: http://localhost:8082"
        }
        failure {
            echo "PIPELINE FAILED. Check logs."
        }
        always {
            echo "Cleaning up temporary test containers..."
            sh 'docker ps -a --filterAncestor=${registry}:${BUILD_NUMBER} -q | xargs -r docker rm -f || true'
        }
    }
}
```

![alt text](<Screenshot from 2025-11-12 23-13-31.png>)

---

### **Test Image Stage**
```groovy
def containerId = sh(script: "docker run -d -p 8083:80 image:7", returnStdout: true)
sleep 5
def status = sh(script: "curl -f http://localhost:8083", returnStatus: true)
```
→ **Test PASSED**

![alt text](image-9.png)

---

### **Deploy Stage**
- Removed old container on **port 8082**  
- Ran new image: `taoudiabdelbasset/tp4devops:7`  
- App live at: **http://localhost:8082**

![alt text](<Screenshot from 2025-11-12 23-19-04.png>)

---

## **Docker Hub**
- Image: `taoudiabdelbasset/tp4devops:7`  
- Tag: `latest` updated  
- All layers pushed successfully

![alt text](image-10.png)

---

## **Conclusion**

| Objective | Status |
|---------|--------|
| Jenkins + Docker Pipeline | Success |
| Automatic build on Git push | Success |
| 5-stage CI/CD | Success |
| Image testing & deployment | Success |
| Web app live & updated | Success |

**Full CI/CD pipeline achieved** — from code change to live deployment in seconds.