Setup required before this runs

Install the plugin — Manage Jenkins → Plugins → SonarQube Scanner for Jenkins.
Create a token in SonarQube — My Account → Security → Generate Token (type: Global Analysis Token).
Add it to Jenkins — Manage Jenkins → Credentials → Add → kind Secret text, paste the token, ID something like sonar-token.
Register the server — Manage Jenkins → System → SonarQube servers → Add. Name it exactly sonar-server (must match the string in withSonarQubeEnv), URL e.g. http://<sonar-host>:9000, and pick the credential from step 3. Tick "Environment variables".
Add the webhook — this is what makes waitForQualityGate work. In SonarQube: Administration → Configuration → Webhooks → Create, URL http://<jenkins-host>:8080/sonarqube-webhook/ (trailing slash matters). Without it the Quality Gate stage just hangs until the timeout.


Install Nexus Artifact Uploader from the plugin manager, and add a Username with password credential with ID nexus-creds. how to do installed plugin next

Plugin's installed. Two things left: create the credential, then add the stage.

Create the credential
Manage Jenkins → Credentials → System → Global credentials (unrestricted) → Add Credentials

Kind: Username with password
Username: your Nexus login (usually admin)
Password: your Nexus password
ID: nexus-creds — type this in yourself, don't leave it blank, because the pipeline references it by this exact string
Description: anything, e.g. "Nexus deploy account"


sudo apt-get install -y wget apt-transport-https gnupg
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb generic main" | sudo tee /etc/apt/sources.list.d/trivy.list
sudo apt-get update && sudo apt-get install -y trivy
trivy --version


sudo usermod -aG docker jenkins
sudo systemctl restart jenkins

sudo -u jenkins docker ps


kubectl create secret docker-registry acr-secret \
  --docker-server=democontainerregi.azurecr.io \
  --docker-username=<acr-username> \
  --docker-password=<acr-password>

  stage('Create ACR Secret') {
    steps {
        withCredentials([usernamePassword(credentialsId: 'acr-creds',
                                          usernameVariable: 'ACR_USER',
                                          passwordVariable: 'ACR_PASS'),
                         file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
            sh '''
                kubectl create secret docker-registry acr-secret \
                  --docker-server=$ACR_SERVER \
                  --docker-username="$ACR_USER" \
                  --docker-password="$ACR_PASS" \
                  --dry-run=client -o yaml | kubectl apply -f -
            '''
        }
    }
}