# Container Security

### Security Recommendations

1. Make images as small as possible
    - It reduces your attack surface by introducing only required packages, tools, and plugins necessary to run the application. 

2. Run containers as non-root. Set a user.
    - For example: USER node or USER 1001 in one of your layers.
    - If a dockerfile does not specify a USER then everything runs as root which means an attacker has full control of a  compromised container.
    - This may introduce access denied issues (if a file is only accessible or was created as root, then this user does not have permission to it) so use the RUN chmod command to change the ownership of the working directory.
    - For example:
      - WORKDIR /app
      - USER 1001
      - RUN chown 1001:1001 /app
    - Kubernetes may have a layer under "spec" to allow or deny root access"
      - For example:
        - apiVersion: v1
        - kind: Pod
        - metadata:
          - name: java-app
        - spec:
          - securityContext:
            - runAsNonRoot: False 
              - ***This should say True***
    - Container runtime (i.e., Docker daemon) with non-root
      - [Docker](https://docs.docker.com/engine/security/rootless)
      - [Podman](https://github.com/containers/podman#rootless)
      - [Kubernetes](https://kubernetes.io/docs/tasks/administer-cluster/kubelet-in-userns)

3. Pinning a version
    - Try to pin down a version of the image you need. For example, don't use node:latest, use node:16.14.2 instead. This prevents development efforts having different versions of the base image (depending on when someone downloads the "latest")
    - Rebuild app/update periodically to get the latest packages from base distro as leaving the original pinned version may introduce vulnerabilities later.

4. COPY VS ADD
    - COPY is more explicit and will only copy exactly what is supposed to versus ADD which copies everything in a directory. This avoids copying environment variables, sensitive files, etc. Still be careful with this command as you can still copy and entire local directory to the container which may include additional unnecessary files or secrets that can introduce additional risk.
    - When building images, use a clean build context (i.e., copying all the required files to a new directory) and build it from there to avoid sensitive or unnecessary files ending up in the image.
    - Using a .dockerignore could also help avoid this issue

5. Vulnerabilities
    - Ensure you are scanning for code vulnerabilities and remediating them.
    - Scan all images at every stage (Build, Publish, Pull and Run stages.)
    - Fix vulnerabilities based on probability of exploitation and risk to avoid overload.

6. Avoid using base OS images (i.e., Ubuntu)
    - By using a large base OS image, you introduce unnecessary attack vectors since the base OS images usually have a lot of tools, packages, and extensions pre-installed that you may not even need for your application.
    - Try to use only the absolute required images needed to run your application.

7. Avoid embedding secrets
    - Avoid using (i.e., ENV API_TOKEN=12345) or copying secrets (i.e. COPY aws_credentials /app/.aws/credentials) in your dockerfile/docker-compose. Even when they are removed later within the same file (i.e., RUN rm /app/.aws/credentials), the previous COPY layer is still recorded and by doing an image copy, inspecting the image and the index someone may be able to extract the exact layer (the COPY layer) SHA, and decompressing the tar with the correct SHA which contains the secrets in plain text.

8. Close Unused Network Ports
    - Avoid exposing or mapping ports that are not needed by your application.

9. Use trusted registries
    - Avoid using custom images from unknown or little-known publishers and maintainers. This introduces risks as it is unknown whether the maintainers are keeping the images up to date, remediating bugs or if they have embedded malware into their images which compromises your container. Also, there's no way to tell if the image in-fact comes from the public dockerfile which means they may embed things in the image that you may not want or are malicious.
    - If you need to use such custom image, build your own dockerfile (referencing the publisher's dockerfile) which builds your own image and ensures only the code you have reviewed, and trust is the only thing being built in the container. Essentially copy the publisher's dockerfile code into your own dockerfile. 

10. Protect registry access
    - Use a private registry as much as possible and do not expose it to the internet.
    - Lock down access to push/publish images into the registry to avoid malicious code being injected into images, and later being pulled and executed.

11. Container runtime
    - When possible, employ a container runtime/configuration drift technology to monitor for malicious/unexpected behavior (i.e., an ngnix container performing non web front end action, additional and non-standard ports being exposed, etc.)

12. Security in workflows
    - Embed security controls/tools starting in the early stages of development to reduce risk, remediate security problems earlier and help development continue as the pace necessary but in a secure manner.
  
``` mermaid
    graph LR
        1[Developer merges code to Git repo. Jenkins Job initiated.] --> 2(Synk's Jenkins Plugin finds Issues and breaks build?)
        2 --> |NO| 3{Jenkins job successful}
        3 --> 4[Snyk's continues monitoring last build run]
        4 --> 5(Snyk finds new issues and alerts using email/slack)
        5 --> 1
        2 --> |YES| 6[Jenkins sends notification to dev about failed build]
        6 --> 7[Dev looks at Snyk artifact in Jenkins]
        7 --> 8{Synk report includes remediation advice?}
        8 --> |NO| 9(Security determines best course of action. i.e., suggest alternative library)
        8 --> |YES| 10[Dev fixes issue, merges to git repo and triggers a Jenkins job]
        10 --> 3
```
