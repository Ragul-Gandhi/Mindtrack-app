# Brain Task App: Docker, Amazon EKS and AWS CodePipeline / CodeBuild

A static React (Vite) app, "Brain Task", containerized with Nginx, pushed to Docker Hub and deployed to an **Amazon EKS** cluster. A CI/CD pipeline built with **AWS CodePipeline + CodeBuild** rebuilds the image and rolls it out to the cluster on every commit.

- **App source (original):** https://github.com/Vennilavanguvi/Brain-Tasks-App
- **Pipeline repo:** https://github.com/Ragul-Gandhi/Mindtrack-sample-app
- **Docker Hub image:** `ragulgandhi/brain-app1`
- **Region / cluster:** `ap-south-1` / `ragul`

---

## Architecture

```
 GitHub  ──►  CodePipeline  ──►  CodeBuild  ──►  Docker Hub
 (source)      (orchestrates)     │  1. docker build              (ragulgandhi/brain-app1)
                                  │  2. docker push                       │
                                  │  3. kubectl apply / set image         │ image pull
                                  ▼                                       ▼
                             Amazon EKS cluster "ragul" (2 nodes)  ◄──────┘
                             ├── Deployment: brain-app-deploy (2 replicas)
                             └── Service: brain-app-service (LoadBalancer :3000 → pod :80)
                                                │
                                                ▼
                                    http://<ELB-hostname>:3000
```

CodeDeploy is **not** used. CodeDeploy agents target EC2 and on-premises servers, not Kubernetes pods. CodeBuild deploys to EKS directly with `kubectl`.

---

## Repository contents

```
.
├── Dockerfile        # Nginx image serving the dist/ folder
├── buildspec.yml     # CodeBuild: build → push → deploy to EKS
├── brain-app.yml     # Kubernetes Service + Deployment
├── nginx.conf        # (optional) SPA fallback config
└── dist/             # Production build output (HTML, JS, CSS, assets)
```

---

## Prerequisites

- An AWS account and a Docker Hub account
- A Docker Hub **access token** (Docker Hub → Account settings → Personal access tokens)
- Basic familiarity with the Linux terminal

---

## Step 1: Prepare the EC2 workstation

This instance is only used to build, test and create the cluster by hand. The pipeline does not depend on it afterwards.

1. Launch an Ubuntu EC2 instance, type **t2.medium** (micro and nano are too small).
2. Create an IAM role for EC2 and attach it to the instance:
   - **IAM → Roles → Create role → AWS service → EC2 → AdministratorAccess → name it `ec2-role`**
   - **EC2 → Actions → Security → Modify IAM role → `ec2-role` → Update**

   > Admin access is convenient for a practice project, because `eksctl` needs to create many resource types. For anything real, use a least-privilege policy.
3. Update the packages:

   ```bash
   sudo apt update
   ```

---

## Step 2: Install Docker and containerize the app

```bash
sudo apt install docker.io -y
sudo usermod -aG docker $USER
newgrp docker                    # apply the group without logging out
docker run hello-world           # verify Docker works without sudo
```

Clone the application:

```bash
git clone https://github.com/Vennilavanguvi/Brain-Tasks-App.git
cd Brain-Tasks-App
```

### Dockerfile

The repo holds the prebuilt `dist/` folder, so only Nginx is needed. There is no Node.js or build step.

```dockerfile
FROM nginx:alpine
COPY dist/ /usr/share/nginx/html/
EXPOSE 80
```

If the app has client-side routes, add an SPA fallback so refreshing `/some-route` doesn't return a 404:

```nginx
# nginx.conf
server {
    listen 80;
    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

```dockerfile
FROM nginx:alpine
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY dist/ /usr/share/nginx/html/
EXPOSE 80
```

### Build, run and test locally

```bash
docker build -t ragulgandhi/brain-app1:latest .
docker run -d -p 8080:80 --name brain-app ragulgandhi/brain-app1:latest
```

Open `http://<EC2-public-IP>:8080`. The EC2 security group must allow inbound TCP 8080.

### Push to Docker Hub

```bash
docker login -u ragulgandhi       # use your access token as the password
docker push ragulgandhi/brain-app1:latest
```

> The image name must start with **your Docker Hub username**. Otherwise the push fails with `insufficient_scope: authorization failed`.

---

## Step 3: Install Kubernetes tooling

**kubectl**

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
file kubectl                      # should say "ELF 64-bit executable"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
kubectl version --client
```

**eksctl**

```bash
curl --silent --location "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
```

**AWS CLI v2** (required by `eksctl`)

```bash
sudo apt install zip -y
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o awscliv2.zip
unzip awscliv2.zip
sudo ./aws/install
aws --version
```

---

## Step 4: Create the EKS cluster

```bash
eksctl create cluster \
  --name ragul \
  --region ap-south-1 \
  --node-type m7i-flex.large \
  --nodes 2
```

This takes roughly 15–20 minutes. Then connect `kubectl` to the cluster and verify:

```bash
aws eks update-kubeconfig --name ragul --region ap-south-1
eksctl get cluster --region ap-south-1
kubectl get nodes -o wide
```

You should see 2 nodes in `Ready` state.

---

## Step 5: Kubernetes manifest

`brain-app.yml` combines the Service and Deployment in one file. The Service exposes the app on port **3000** and forwards to the Nginx container on port 80.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: brain-app-service
spec:
  selector:
    app: brain-app
  ports:
    - protocol: TCP
      port: 3000
      targetPort: 80
  type: LoadBalancer
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: brain-app-deploy
spec:
  replicas: 2
  selector:
    matchLabels:
      app: brain-app
  template:
    metadata:
      labels:
        app: brain-app
    spec:
      containers:
        - name: project-1-app
          image: ragulgandhi/brain-app1:latest
          ports:
            - containerPort: 80
```

Deploy manually once to confirm everything works:

```bash
kubectl apply -f brain-app.yml
kubectl get pods
kubectl get svc brain-app-service
```

Wait until the pods show `Running` and the service shows an `EXTERNAL-IP` (an AWS ELB hostname). This can take a couple of minutes. Then open:

```
http://<ELB-hostname>:3000
```

---

## Step 6: CI/CD with CodePipeline and CodeBuild

### 6.1 Store Docker Hub credentials in SSM Parameter Store

```bash
aws ssm put-parameter \
  --name "/brain-app/dockerhub/username" \
  --type String \
  --value "ragulgandhi" \
  --region ap-south-1

aws ssm put-parameter \
  --name "/brain-app/dockerhub/password" \
  --type SecureString \
  --value "<your-docker-hub-access-token>" \
  --region ap-south-1
```

### 6.2 Create the CodeBuild project

| Setting | Value |
|---|---|
| Project name | `MindTrack` |
| Source | GitHub repo `Mindtrack-sample-app` (via CodeConnections) |
| Environment image | Managed image, Ubuntu, `aws/codebuild/standard:7.0` |
| **Privileged** | **Enabled**. This is required to run Docker builds. |
| Service role | `codebuild-MindTrack-service-role` (auto-created) |
| Buildspec | Use a buildspec file: `buildspec.yml` |

### 6.3 Give the CodeBuild role the permissions it needs

**a) Read the Docker Hub parameters.** Add this inline policy to the CodeBuild service role:

```json
{
  "Effect": "Allow",
  "Action": ["ssm:GetParameters", "ssm:GetParameter"],
  "Resource": "arn:aws:ssm:ap-south-1:<ACCOUNT_ID>:parameter/brain-app/dockerhub/*"
}
```

**b) Describe the cluster.** `aws eks update-kubeconfig` needs this:

```json
{
  "Effect": "Allow",
  "Action": "eks:DescribeCluster",
  "Resource": "arn:aws:eks:ap-south-1:<ACCOUNT_ID>:cluster/ragul"
}
```

**c) Allow the role to use `kubectl` inside the cluster.** IAM permissions alone are not enough, so create an EKS access entry:

```bash
aws eks create-access-entry --cluster-name ragul --region ap-south-1 \
  --principal-arn arn:aws:iam::<ACCOUNT_ID>:role/service-role/codebuild-MindTrack-service-role

aws eks associate-access-policy --cluster-name ragul --region ap-south-1 \
  --principal-arn arn:aws:iam::<ACCOUNT_ID>:role/service-role/codebuild-MindTrack-service-role \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy \
  --access-scope type=cluster
```

> `AmazonEKSClusterAdminPolicy` is broad. For tighter access, use `AmazonEKSEditPolicy` scoped to the `default` namespace:
> `--policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSEditPolicy --access-scope type=namespace,namespaces=default`

### 6.4 buildspec.yml

Commit this file to the root of the repo.

```yaml
version: 0.2

env:
  variables:
    IMAGE_REPO: "ragulgandhi/brain-app1"
    CLUSTER_NAME: "ragul"
    AWS_REGION: "ap-south-1"
    K8S_NAMESPACE: "default"
    DEPLOYMENT_NAME: "brain-app-deploy"   # must match metadata.name in brain-app.yml
    CONTAINER_NAME: "project-1-app"       # must match containers[].name in brain-app.yml
    KUBECTL_VERSION: "v1.31.0"            # keep within one minor version of the EKS version
  parameter-store:
    DOCKERHUB_USER: "/brain-app/dockerhub/username"
    DOCKERHUB_PASS: "/brain-app/dockerhub/password"

phases:
  install:
    commands:
      - curl -fsSLo kubectl "https://dl.k8s.io/release/${KUBECTL_VERSION}/bin/linux/amd64/kubectl"
      - file kubectl
      - chmod +x kubectl && mv kubectl /usr/local/bin/kubectl
      - kubectl version --client

  pre_build:
    commands:
      - echo "$DOCKERHUB_PASS" | docker login -u "$DOCKERHUB_USER" --password-stdin
      - IMAGE_TAG=$(echo "$CODEBUILD_RESOLVED_SOURCE_VERSION" | cut -c 1-7)
      - echo "Image tag will be $IMAGE_TAG"

  build:
    commands:
      - docker build --pull -t $IMAGE_REPO:latest -t $IMAGE_REPO:$IMAGE_TAG .

  post_build:
    commands:
      # post_build runs even when build fails, so only push and deploy on success
      - |
        if [ "$CODEBUILD_BUILD_SUCCEEDING" = "1" ]; then
          docker push $IMAGE_REPO:latest
          docker push $IMAGE_REPO:$IMAGE_TAG

          aws eks update-kubeconfig --name "$CLUSTER_NAME" --region "$AWS_REGION"
          kubectl get nodes
          kubectl apply -n "$K8S_NAMESPACE" -f brain-app.yml
          kubectl set image deployment/"$DEPLOYMENT_NAME" "$CONTAINER_NAME"="$IMAGE_REPO:$IMAGE_TAG" -n "$K8S_NAMESPACE"
          kubectl rollout status deployment/"$DEPLOYMENT_NAME" -n "$K8S_NAMESPACE" --timeout=180s
          kubectl get svc brain-app-service -n "$K8S_NAMESPACE"
        else
          echo "Build failed, skipping push and deploy"
        fi

artifacts:
  files:
    - brain-app.yml
```

**How the deploy works**

1. The image is built and pushed with two tags: `latest` and the 7-character commit hash.
2. `kubectl apply` creates or updates the Service and Deployment.
3. `kubectl set image` pins the Deployment to the commit-tagged image. This forces a fresh rollout every time, which would not happen with `:latest` alone, since that tag never changes.
4. `kubectl rollout status` fails the build if the new pods don't become healthy within 3 minutes.

### 6.5 Create the pipeline in CodePipeline

1. **CodePipeline → Create pipeline**
2. **Source stage:** GitHub (via CodeConnections), repo `Mindtrack-sample-app`, branch `main`
3. **Build stage:** AWS CodeBuild, project `MindTrack`
4. **Deploy stage:** skip it. The deployment happens inside the CodeBuild project.
5. Create the pipeline and push a commit to confirm it runs end to end.

---

## Verify the deployment

```bash
kubectl get pods
kubectl get deployment brain-app-deploy
kubectl get svc brain-app-service
kubectl describe deployment brain-app-deploy | grep Image
```

The `Image` line should show the latest commit tag, for example `ragulgandhi/brain-app1:a1b2c3d`.

Open `http://<ELB-hostname>:3000` in a browser.

CodeBuild logs are available in the console under **CodeBuild → Build history → the build → Build logs**, and in **CloudWatch Logs** under `/aws/codebuild/MindTrack`.

---

## Troubleshooting

| Problem | Cause and fix |
|---|---|
| `unable to evaluate symlinks in Dockerfile path` | The file name is case-sensitive. It must be `Dockerfile`, not `DockerFile`. Use `mv DockerFile Dockerfile`. |
| `error reading .dockerignore: is a directory` | `.dockerignore` was created as a folder. Remove it and recreate it as a file: `rm -rf .dockerignore && printf ".git\nREADME.md\n" > .dockerignore`. |
| `push access denied ... insufficient_scope` | The image name doesn't start with your Docker Hub username, or you ran `sudo docker push` after logging in without `sudo`. Retag with `docker tag` and push without `sudo`. |
| `kubectl: syntax error near unexpected token newline` | The download saved an XML error page as `kubectl`. Remove it, check `echo $KVER` and `file kubectl`, then reinstall. |
| CodeBuild: `Cannot connect to the Docker daemon` | **Privileged** mode is not enabled on the CodeBuild project. |
| CodeBuild: `AccessDeniedException` on `ssm:GetParameters` | The inline SSM policy is missing or the parameter path doesn't match. |
| CodeBuild: `error: You must be logged in to the server (Unauthorized)` | The EKS access entry / policy association is missing for the CodeBuild role. |
| `deployments.apps "..." not found` or `unable to find container` | `DEPLOYMENT_NAME` or `CONTAINER_NAME` in the buildspec doesn't match `brain-app.yml`. |
| Pods stuck in `ImagePullBackOff` | The image name or tag is wrong, or the Docker Hub repo is private. For a private repo, add an `imagePullSecret`. |
| Browser can't reach the app | Use port **3000** (the Service port), not 80, and allow a few minutes for the ELB DNS to resolve. |

---

## Clean up (avoid ongoing AWS charges)

An EKS cluster and its load balancer incur hourly costs, so delete them when you're done:

```bash
kubectl delete -f brain-app.yml      # removes the ELB created by the Service
eksctl delete cluster --name ragul --region ap-south-1
```

Also stop or terminate the EC2 workstation, and delete the CodePipeline, CodeBuild project and SSM parameters if you no longer need them.

---

## Security notes

- Keep Docker Hub credentials only in SSM Parameter Store. Never commit them to the repo.
- Use a Docker Hub **access token**, not your account password.
- Replace the AdministratorAccess role and the cluster-admin policy with least-privilege equivalents outside of practice environments.
