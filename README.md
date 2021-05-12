# ARO DevOps Labs

## Lab 4: Deploy SonarQube

### Deploy SonarQube

1. Run `helm install sonarqube-lab-3 redhat-cop/sonarqube` to install an instance of sonarqube using the default values.

2. Once the app is up and running, login using the default credentials and you'll be prompted to change your password.

### Add SonarQube to GitHub Actions Workflow

1. First, we will provision a token in SonarQube token for the GitHub Actions Workflow. View https://docs.sonarqube.org/latest/user-guide/user-token/ for instructions on the token.

2. Add a `SONAR_TOKEN` secret to the repo as well as a `SONAR_ROUTE` secret that points to the sonarqube instance deployed.

3. Push up to `lab-4` and watch the pipeline execute. After it's complete, navigate to sonarqube and you should see data that came back from the scan.