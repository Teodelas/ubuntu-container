# Ubuntu

- This repository contains source code to build a sample Ubuntu based docker image that runns nginx server on port 80. 
- The Dockerfile uses a base image that contains `entrypoint.sh` shell script.
- The script `entrypoint.sh` in the base image executes as soon as the container starts but the evil function inside the script sleeps for a predetermined amount of time before it executes to avoid detection by traditional image scanning.
- The evil function downloads and executes malware samples, cryptominer and also performs lateral movement.
- Using this example, we can demonstrate how Prisma Cloud Image Analysis Sandbox can detect and block this.
