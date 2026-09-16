Setup required before this runs

Install the plugin — Manage Jenkins → Plugins → SonarQube Scanner for Jenkins.
Create a token in SonarQube — My Account → Security → Generate Token (type: Global Analysis Token).
Add it to Jenkins — Manage Jenkins → Credentials → Add → kind Secret text, paste the token, ID something like sonar-token.
Register the server — Manage Jenkins → System → SonarQube servers → Add. Name it exactly sonar-server (must match the string in withSonarQubeEnv), URL e.g. http://<sonar-host>:9000, and pick the credential from step 3. Tick "Environment variables".
Add the webhook — this is what makes waitForQualityGate work. In SonarQube: Administration → Configuration → Webhooks → Create, URL http://<jenkins-host>:8080/sonarqube-webhook/ (trailing slash matters). Without it the Quality Gate stage just hangs until the timeout.