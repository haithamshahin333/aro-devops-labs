# ARO DevOps Labs

## Lab 4: Deploy SonarQube

### Deploy SonarQube

1. Run `helm install sonarqube-lab-3 redhat-cop/sonarqube` to install an instance of sonarqube using the default values.

2. Once the app is up and running, login using the default credentials and you'll be prompted to change your password.

### Add SonarQube to GitHub Actions Workflow

1. First, we will provision a token in SonarQube token 