## Prisma Cloud Docker Image Scan GitHub Action

This GitHub Actions workflow performs a Prisma Cloud scan on a Docker image built from a GitHub pull request. The workflow consists of the following steps:

1. **Triggering the workflow:** The workflow is triggered on every pull request made against the `main` branch of the repository.

```
on:
  pull_request:
    branches:
      - 'main'
```

2. **Defining the job:** The `Docker-Image-Scan` job is defined and runs on an `ubuntu-latest` virtual machine.

```
jobs:
  Docker-Image-Scan:
    runs-on: ubuntu-latest
```    

3. **Checking out the repository:** The repository is checked out using the `actions/checkout@v3` action.

```
steps:
  - uses: actions/checkout@v3
```

4. **Logging in to the Docker registry:** The `docker/login-action@v2` action is used to log in to the Docker registry using the `DOCKERHUB_TOKEN` secret.

```
- name: Docker login
  uses: docker/login-action@v2
  with:
    username: ultimatetestdrive
    password: ${{ secrets.DOCKERHUB_TOKEN }}
```

5. **Building the Docker image:** The `docker/build-push-action@v4` action is used to build the Docker image and tag it with the current `github.run_number`.

```
- name: Docker Build
  uses: docker/build-push-action@v4
  with:
    push: false
    tags: ultimatetestdrive/ubuntu:${{ github.run_number }}
```

6. **Running the Prisma Cloud scan:** The `PaloAltoNetworks/prisma-cloud-scan@v1` action is used to run the Prisma Cloud scan on the Docker image.

```
- name: Docker Image Scan
  id: scan
  uses: PaloAltoNetworks/prisma-cloud-scan@v1
  with:
    pcc_console_url: ${{ secrets.PCC_CONSOLE_URL }}
    pcc_user: ${{ secrets.PCC_ACCESS_KEY_ID }}
    pcc_pass: ${{ secrets.PCC_SECRET_ACCESS_KEY }}
    image_name: ultimatetestdrive/ubuntu:${{ github.run_number }}
```

The `pcc_console_url`, `pcc_user`, and `pcc_pass` inputs are configured using GitHub secrets to authenticate with the Prisma Cloud console.

7. **Running the twistcli sandbox:** The `twistcli` sandbox is run using a `curl` command to scan the Docker image.

```
- name: Docker Image Sandbox Scan
  run: |
    curl -s https://utd-packages.s3.amazonaws.com/twistcli --output twistcli
    chmod +x twistcli
    sudo ./twistcli sandbox --address ${{ secrets.PCC_CONSOLE_URL }} --user ${{ secrets.PCC_ACCESS_KEY_ID }} --password ${{ secrets.PCC_SECRET_ACCESS_KEY }} --analysis-duration 2m ultimatetestdrive/ubuntu:${{ github.run_number }}
```

8. **Pushing the Docker image:** If all the previous steps pass, the Docker image is pushed to the Docker registry.

```
- name: Docker Push
  run: |
    docker push ultimatetestdrive/ubuntu:${{ github.run_number }}
```
### Required Workflow Secrets
Below are the secrets that are passed as variables to this Github Action:
- **DOCKERHUB_TOKEN:** Dockerhub personal access token
- **PCC_CONSOLE_URL:** Prisma Cloud Compute URL
- **PCC_ACCESS_KEY_ID:** Prisma Cloud Access Key ID
- **PCC_SECRET_ACCESS_KEY:** Prisma Cloud Secret Access Key
- **PC_API_KEY:** Prsima Cloud Access Key ID and Secret Key in the format `<PCC_SECRET_ACCESS_KEY>::<PC_API_KEY>`

## Conclusion

- Prisma Cloud provides an effective way to scan Docker images for vulnerabilities and misconfigurations. 
- The GitHub Actions workflow we've demonstrated automates this process, allowing developers to identify and fix issues early in the software development lifecycle. 
- By integrating Prisma Cloud scans into your CI/CD pipeline, you can ensure that your Docker images are secure and compliant before they are deployed. 
- With the use of this GitHub Actions workflow and Prisma Cloud scans, you can have greater confidence in the security of your applications and reduce the risk of potential attacks.

## References
- https://github.com/PaloAltoNetworks/prisma-cloud-scan
- https://github.com/docker/login-action
- https://github.com/docker/build-push-action

