

# Git Flow and Deployment Pipeline Diagram

### Various Strategy:

Git branching strategy 1:

```plaintext
                      ┌───────────────┐
                      │               │
                      │    Main Branch│
                      │     (main)    │
                      │               │
                      └───────┬───────┘
                              │
                              ▼
                ┌──────────────────────────┐
                │ CodePipeline: Source Stage│
                │ - Watches GitHub/Repo    │
                │ - Pulls code from main   │
                └──────────────────────────┘
                              │
                              ▼
                     ┌────────────────┐
                     │                │
                     │  Build Stage   │
                     │ (CodeBuild)    │
                     │ - Build image  │
                     │ - Push image   │
                     │ into Amazon ECR│
                     └────────────────┘
                              │
   ┌──────────────────────────┼───────────────────────────┐
   ▼                          ▼                           ▼
┌──────────────┐      ┌──────────────┐          ┌──────────────┐   
│ Alpha Branch │----▶ │ Beta Branch  │ --------> │ UAT Branch │ 
      │                  │                  └────┐ │-------
 │────------------------------────────────--------→---------
    ┌──────────────       






Git branching strategy 2:

```markdown
# Git Branching Strategy

```plaintext
                        ┌────────────────┐
                        │ Feature Branch │
                        │   feature/*    │
                        └────────┬───────┘
                                 │
                        ┌────────▼────────┐
                        │     Main        │
                        │   Branch (main) │
                        └────────┬────────┘
                                 │
       ┌─────────────────────────┼─────────────────────────┐
       ▼                         ▼                         ▼
┌────────────┐           ┌────────────┐           ┌────────────┐
│ Alpha Branch│           │ Beta Branch │           │ UAT Branch │ 
--

 Git branching strategy:

```markdown
# Git Branching Strategy

```plaintext
                       ┌───────────────┐
                       │ Feature Branch│
                       │  feature/*    │
                       └───────┬───────┘
                               │
                               ▼
                       ┌───────────────┐
                       │   Main Branch │
                       │     (main)    │
                       └───────┬───────┘
                               │
         ┌─────────────────────┼─────────────────────┐
         ▼                     ▼                     ▼
    ┌────────────┐       ┌────────────┐       ┌────────────┐
    │ Alpha Branch│       │ Beta Branch │       │  UAT Branch │
    │   (alpha)   │       │   (beta)    │       │   (uat)    │
    └───────┬─────┘       └───────┬─────┘       └───────┬─────┘
            │                     │                     │
            ▼                     ▼                     ▼
    Deploy to Alpha       Deploy to Beta       Deploy to UAT

    ┌───────────────────────────────────────────────────────┐
    │                        Prod Branch                   │
    │                          (prod)                      │
    └───────────────────────────────────────────────────────┘
                               │
                               ▼
                       Deploy to Production
```

### Explanation:
1. **Feature Branches (`feature/*`):**
    - Developers work on features in isolated branches.
    - Merged into the `main` branch upon completion.

2. **Main Branch (`main`):**
    - Holds the latest stable code ready for deployment to environments.

3. **Environment Branches (`alpha`, `beta`, `uat`):**
    - Used for deploying to specific environments (testing, staging, UAT).

4. **Production Branch (`prod`):**
    - Final production-ready code after validation in other environments.

This branching strategy ensures stability and enables controlled deployments to different environments.



To create a **build-once-run-everywhere** deployment model for an Amazon EKS environment with multiple 
stages like `alpha`, `beta`, `uat`, and `prod`, 
follow a Git branching strategy paired with a containerized deployment pipeline. 
This approach ensures you build your container image once and deploy 
it across all environments with environment-specific configurations.

---

### **1. Git Branching Model**
Use a **Gitflow-inspired model** with branches representing environments:

- **Main Branches:**
    - `main`: Production-ready code.
    - `develop`: Integrates features and prepares for the next release.

- **Environment Branches:**  
  Each branch corresponds to a deployment environment:
    - `alpha`: Testing environment for developers.
    - `beta`: Staging environment for QA.
    - `uat`: Pre-production environment for user acceptance testing.
    - `prod`: Production-ready environment.

- **Feature and Hotfix Branches:**
    - Create short-lived `feature/` branches for new features.
    - Create `hotfix/` branches for urgent fixes, merging them into `main` and environment branches as needed.

---

### **2. CI/CD Pipeline**
Use a CI/CD tool like **GitHub Actions**, **GitLab CI/CD**, **Jenkins**, or **AWS CodePipeline** to implement the pipeline. The pipeline should:

1. Build the container image once when code is merged into `develop` or `main`.
2. Push the image to a container registry (e.g., Amazon Elastic Container Registry (ECR)).
3. Deploy the image to the appropriate environment (EKS cluster) based on the branch.

---

#### **Pipeline Stages**
1. **Build Stage:**
    - Triggered on merging to `develop` or `main`.
    - Build the container image and tag it with a unique version (e.g., Git commit SHA or semantic version).
    - Push the image to Amazon ECR.

2. **Deploy Stage:**
    - Triggered on changes to environment branches (`alpha`, `beta`, `uat`, `prod`).
    - Deploy the prebuilt image to the corresponding EKS environment.
    - Apply environment-specific Kubernetes manifests or configurations using tools like `kubectl` or `Helm`.

---

### **3. Kubernetes Manifests**
Use the **Helm** package manager or `kustomize` for environment-specific configurations.

#### Example Helm Chart (`values.yaml`):
```yaml
# Common settings for all environments
replicaCount: 2
image:
  repository: <your-ecr-repository>
  pullPolicy: Always
  tag: "" # Set dynamically during deployment
service:
  type: ClusterIP
  port: 8080
env:
  AWS_REGION: us-west-2
  XRAY_LOG_LEVEL: info

# Environment-specific values (overrides)
environments:
  alpha:
    replicaCount: 1
    env:
      SERVICE_NAME: service-alpha
  beta:
    replicaCount: 2
    env:
      SERVICE_NAME: service-beta
  uat:
    replicaCount: 3
    env:
      SERVICE_NAME: service-uat
  prod:
    replicaCount: 5
    env:
      SERVICE_NAME: service-prod
```

---

### **4. Example GitHub Actions Workflow**
Here’s an example of a GitHub Actions workflow for build-once-run-everywhere:

#### **`.github/workflows/eks-deploy.yaml`:**
```yaml
name: EKS Deployment

on:
  push:
    branches:
      - develop
      - main
      - alpha
      - beta
      - uat
      - prod

jobs:
  build:
    if: github.ref == 'refs/heads/develop' || github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v3

      - name: Authenticate to AWS ECR
        run: aws ecr get-login-password --region us-west-2 | docker login --username AWS --password-stdin <your-account-id>.dkr.ecr.us-west-2.amazonaws.com

      - name: Build Docker Image
        run: docker build -t <your-service-name>:${{ github.sha }} .

      - name: Push Docker Image
        run: |
          docker tag <your-service-name>:${{ github.sha }} <your-account-id>.dkr.ecr.us-west-2.amazonaws.com/<your-service-name>:${{ github.sha }}
          docker push <your-account-id>.dkr.ecr.us-west-2.amazonaws.com/<your-service-name>:${{ github.sha }}

  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v3

      - name: Configure AWS CLI
        run: aws configure set region us-west-2

      - name: Deploy to EKS
        run: |
          IMAGE_TAG=${{ github.sha }}
          ENV=${{ github.ref_name }}
          helm upgrade --install my-service ./helm-chart \
            --set image.tag=$IMAGE_TAG \
            --set environment=$ENV
```

---

### **5. Key Concepts**
1. **Image Tagging:**
    - Use a consistent image tag (e.g., Git commit SHA) to identify the same build across environments.

2. **Environment-specific Configurations:**
    - Use Helm, `kustomize`, or parameterized Kubernetes manifests to inject environment-specific variables like replica count, service names, and database connections.

3. **Permissions:**
    - Use IAM roles and service accounts for secure access to ECR and EKS.

4. **Rollbacks:**
    - Include rollback capabilities in the pipeline using Helm’s `rollback` command or Kubernetes deployment strategies.

5. **Monitoring and Alerts:**
    - Integrate tools like **Prometheus**, **Grafana**, or **Datadog** to monitor deployments and application performance.

---

This strategy ensures that the same artifact is tested and promoted through `alpha`, `beta`, `uat`, and finally `prod`, with minimal duplication and strong consistency. Let me know if you'd like help customizing any part!

## Other option with only  main branch and no develop branch:

If your Git branching model doesn’t include a `develop` branch and you want to simplify 
the process for a Go project, you can adapt the build-once-run-everywhere model as follows. 
We'll directly use environment branches (`alpha`, `beta`, `uat`, `prod`) 
alongside the `main` branch.

---

### **1. Git Branching Strategy**
- **Main Branch (`main`)**:
    - Contains production-ready code.
    - Merges into `alpha`, `beta`, `uat`, or `prod` trigger deployments to those environments.

- **Environment Branches:**
    - **`alpha`**: Early-stage environment for testing.
    - **`beta`**: Staging environment for QA testing.
    - **`uat`**: Pre-production environment for user acceptance testing.
    - **`prod`**: Production environment for live deployments.

- **Feature Branches (`feature/*`)**:
    - Create for individual features or fixes. Merge into `main` for testing in the pipeline.

---

### **2. CI/CD Workflow**

Here’s an updated **GitHub Actions workflow** for a Go project that builds once on `main` 
and deploys to different environments (`alpha`, `beta`, `uat`, `prod`).

---

#### **GitHub Actions Workflow**

**File:** `.github/workflows/go-eks-deploy.yaml`

```yaml
name: EKS Deployment Pipeline

on:
  push:
    branches:
      - main
      - alpha
      - beta
      - uat
      - prod

jobs:
  build:
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v3

      - name: Set up Go
        uses: actions/setup-go@v4
        with:
          go-version: 1.20

      - name: Run Tests
        run: go test ./...

      - name: Build Binary
        run: go build -o app ./cmd/main.go

      - name: Build Docker Image
        run: docker build -t <your-service-name>:${{ github.sha }} .

      - name: Authenticate to AWS ECR
        run: aws ecr get-login-password --region us-west-2 | docker login --username AWS --password-stdin <your-account-id>.dkr.ecr.us-west-2.amazonaws.com

      - name: Push Docker Image to ECR
        run: |
          docker tag <your-service-name>:${{ github.sha }} <your-account-id>.dkr.ecr.us-west-2.amazonaws.com/<your-service-name>:${{ github.sha }}
          docker push <your-account-id>.dkr.ecr.us-west-2.amazonaws.com/<your-service-name>:${{ github.sha }}

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v3

      - name: Configure AWS CLI
        run: aws configure set region us-west-2

      - name: Deploy to EKS
        env:
          IMAGE_TAG: ${{ github.sha }}
          ENVIRONMENT: ${{ github.ref_name }}
        run: |
          echo "Deploying to $ENVIRONMENT environment"
          helm upgrade --install my-service ./helm-chart \
            --set image.repository=<your-account-id>.dkr.ecr.us-west-2.amazonaws.com/<your-service-name> \
            --set image.tag=$IMAGE_TAG \
            --set environment=$ENVIRONMENT
```

---

### **3. Kubernetes Configuration for Go Microservice**

Use **Helm** for environment-specific configurations.

#### **`helm-chart/values.yaml` (Base Configuration)**
```yaml
# Common configuration for all environments
replicaCount: 2
image:
  repository: ""
  pullPolicy: Always
  tag: ""
service:
  type: ClusterIP
  port: 8080
env:
  AWS_REGION: us-west-2
  LOG_LEVEL: info
```

#### **`helm-chart/values-alpha.yaml`**
```yaml
replicaCount: 1
env:
  SERVICE_NAME: service-alpha
  LOG_LEVEL: debug
```

#### **`helm-chart/values-prod.yaml`**
```yaml
replicaCount: 5
env:
  SERVICE_NAME: service-prod
  LOG_LEVEL: error
```

---

### **4. Go Project Structure**
Organize your Go project to align with this pipeline.

#### **Directory Structure**
```plaintext
.
├── cmd/
│   └── main.go            # Main application entry point
├── pkg/
│   └── handlers/          # Handlers for routes
├── Dockerfile             # Docker build file
├── helm-chart/            # Helm chart for Kubernetes deployment
├── go.mod                 # Go modules
├── go.sum
└── .github/
    └── workflows/
        └── go-eks-deploy.yaml
```

---

### **5. Dockerfile for Go Microservice**
Ensure your Dockerfile is optimized for the build-once-run-everywhere approach.

#### **`Dockerfile`**
```dockerfile
# Build stage
FROM golang:1.20 AS builder
WORKDIR /app
COPY . .
RUN go build -o app ./cmd/main.go

# Runtime stage
FROM alpine:latest
WORKDIR /root/
COPY --from=builder /app/app .
CMD ["./app"]
```

---

### **6. Process Workflow**

#### **Developer Workflow**
1. Developers create feature branches (`feature/*`) and merge them into `main` after code review.
2. The pipeline:
    - Builds the container image on `main`.
    - Pushes the image to ECR.

#### **Deployment Workflow**
1. Merge `main` into the desired environment branch (`alpha`, `beta`, `uat`, or `prod`).
2. The pipeline:
    - Deploys the prebuilt image from ECR to the respective EKS environment.
    - Uses Helm to apply environment-specific configurations.

---

### **7. Key Advantages**
1. **Consistency:** The same artifact (container image) is deployed across all environments.
2. **Reduced Overhead:** No need for redundant builds in each environment branch.
3. **Flexibility:** Environment-specific configurations (e.g., replicas, environment variables) are handled by Helm.
4. **Simplified Git Model:** Directly use environment branches without needing a `develop` branch.

---

This setup is tailored for Go projects, ensuring efficient builds, environment-specific deployments, and minimal duplication. Let me know if you need further customization!

Using **Kustomize** for environment-specific configurations instead of Helm simplifies Kubernetes deployments for a **build-once-run-everywhere** model. Here’s how to adapt the pipeline and configuration for a Go project with Kustomize:

---

### **1. Git Branching Strategy**

#### Branches:
- `main`: Production-ready code.
- `alpha`, `beta`, `uat`, `prod`: Environment-specific branches for deployments.

---

### **2. CI/CD Workflow**

#### Updated **GitHub Actions Workflow** for Kustomize

**File:** `.github/workflows/go-eks-kustomize.yaml`
```yaml
name: EKS Deployment Pipeline with Kustomize

on:
  push:
    branches:
      - main
      - alpha
      - beta
      - uat
      - prod

jobs:
  build:
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v3

      - name: Set up Go
        uses: actions/setup-go@v4
        with:
          go-version: 1.20

      - name: Run Tests
        run: go test ./...

      - name: Build Binary
        run: go build -o app ./cmd/main.go

      - name: Build Docker Image
        run: docker build -t <your-service-name>:${{ github.sha }} .

      - name: Authenticate to AWS ECR
        run: aws ecr get-login-password --region us-west-2 | docker login --username AWS --password-stdin <your-account-id>.dkr.ecr.us-west-2.amazonaws.com

      - name: Push Docker Image to ECR
        run: |
          docker tag <your-service-name>:${{ github.sha }} <your-account-id>.dkr.ecr.us-west-2.amazonaws.com/<your-service-name>:${{ github.sha }}
          docker push <your-account-id>.dkr.ecr.us-west-2.amazonaws.com/<your-service-name>:${{ github.sha }}

  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v3

      - name: Configure AWS CLI
        run: aws configure set region us-west-2

      - name: Deploy to EKS
        env:
          IMAGE_TAG: ${{ github.sha }}
          ENVIRONMENT: ${{ github.ref_name }}
        run: |
          echo "Deploying to $ENVIRONMENT environment"
          kubectl apply -k ./kustomize/overlays/$ENVIRONMENT
```

---

### **3. Kubernetes Configuration with Kustomize**

#### **Directory Structure**
Organize your Kubernetes manifests to use Kustomize overlays for each environment.

```plaintext
kustomize/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
├── overlays/
│   ├── alpha/
│   │   ├── kustomization.yaml
│   │   └── patch-deployment.yaml
│   ├── beta/
│   │   ├── kustomization.yaml
│   │   └── patch-deployment.yaml
│   ├── uat/
│   │   ├── kustomization.yaml
│   │   └── patch-deployment.yaml
│   └── prod/
│       ├── kustomization.yaml
│       └── patch-deployment.yaml
```

---

#### **Base Configuration**

##### **`kustomize/base/deployment.yaml`**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-service
  labels:
    app: my-service
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-service
  template:
    metadata:
      labels:
        app: my-service
    spec:
      containers:
        - name: my-service
          image: <your-ecr-repository>:latest
          ports:
            - containerPort: 8080
          env:
            - name: AWS_REGION
              value: us-west-2
            - name: LOG_LEVEL
              value: info
```

##### **`kustomize/base/service.yaml`**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: my-service
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: ClusterIP
```

##### **`kustomize/base/kustomization.yaml`**
```yaml
resources:
  - deployment.yaml
  - service.yaml
```

---

#### **Environment Overlays**

##### **`kustomize/overlays/alpha/kustomization.yaml`**
```yaml
resources:
  - ../../base

patchesStrategicMerge:
  - patch-deployment.yaml
```

##### **`kustomize/overlays/alpha/patch-deployment.yaml`**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-service
spec:
  replicas: 1
  template:
    spec:
      containers:
        - name: my-service
          image: <your-ecr-repository>:${IMAGE_TAG} # Replaced during deployment
          env:
            - name: LOG_LEVEL
              value: debug
```

Repeat this pattern for other environments (`beta`, `uat`, `prod`), adjusting `replicas` or `env` as needed.

---

### **4. Process Workflow**

#### **Build Stage**
- Triggered on `main` branch pushes.
- Builds the Docker image, tags it with the commit SHA, and pushes it to ECR.

#### **Deploy Stage**
- Triggered on `alpha`, `beta`, `uat`, or `prod` branch pushes.
- Deploys the prebuilt image to the appropriate EKS environment using Kustomize.

---

### **5. Benefits of Kustomize**
1. **Declarative Overlays:** Clean separation between base manifests and environment-specific configurations.
2. **Reusability:** Base manifests are shared, reducing duplication.
3. **Flexibility:** Overlays allow you to patch any part of the Kubernetes manifests.
4. **Ease of Maintenance:** Each environment-specific overlay is modular and easy to update.

---

This approach ensures a streamlined and reusable setup for deploying Go microservices to multiple EKS environments with environment-specific configurations using Kustomize. Let me know if you'd like further adjustments!


To use **AWS CodePipeline** for a **build-once-run-everywhere** deployment model with **Kustomize** and multiple environments (`alpha`, `beta`, `uat`, `prod`) for a Go microservice, follow the steps below. This setup leverages **AWS CodePipeline**, **CodeBuild**, and **EKS**.

---

### **1. CodePipeline Overview**
- **Source Stage:** Monitors a Git repository (e.g., CodeCommit, GitHub, or Bitbucket) for changes.
- **Build Stage:** Builds the Docker image using CodeBuild and pushes it to Amazon ECR.
- **Deploy Stage:** Uses CodeBuild to apply Kustomize overlays to the appropriate EKS environment.

---

### **2. AWS CodePipeline YAML Configuration**

Save this configuration as `pipeline.yml`.

```yaml
Version: "2010-09-09"
Resources:
  MyPipeline:
    Type: AWS::CodePipeline::Pipeline
    Properties:
      Name: GoMicroservicePipeline
      RoleArn: arn:aws:iam::<account-id>:role/CodePipelineServiceRole
      ArtifactStore:
        Type: S3
        Location: <your-s3-bucket>
      Stages:
        - Name: Source
          Actions:
            - Name: SourceAction
              ActionTypeId:
                Category: Source
                Owner: ThirdParty
                Provider: GitHub
                Version: 1
              OutputArtifacts:
                - Name: SourceOutput
              Configuration:
                RepositoryName: <your-repo-name>
                BranchName: main
                OAuthToken: <github-oauth-token>
              RunOrder: 1
        - Name: Build
          Actions:
            - Name: BuildAction
              ActionTypeId:
                Category: Build
                Owner: AWS
                Provider: CodeBuild
                Version: 1
              InputArtifacts:
                - Name: SourceOutput
              OutputArtifacts:
                - Name: BuildOutput
              Configuration:
                ProjectName: GoMicroserviceBuild
              RunOrder: 1
        - Name: DeployToAlpha
          Actions:
            - Name: DeployToAlphaAction
              ActionTypeId:
                Category: Build
                Owner: AWS
                Provider: CodeBuild
                Version: 1
              InputArtifacts:
                - Name: BuildOutput
              Configuration:
                ProjectName: DeployToAlpha
              RunOrder: 1
        - Name: DeployToBeta
          Actions:
            - Name: DeployToBetaAction
              ActionTypeId:
                Category: Build
                Owner: AWS
                Provider: CodeBuild
                Version: 1
              InputArtifacts:
                - Name: BuildOutput
              Configuration:
                ProjectName: DeployToBeta
              RunOrder: 1
        - Name: DeployToUAT
          Actions:
            - Name: DeployToUATAction
              ActionTypeId:
                Category: Build
                Owner: AWS
                Provider: CodeBuild
                Version: 1
              InputArtifacts:
                - Name: BuildOutput
              Configuration:
                ProjectName: DeployToUAT
              RunOrder: 1
        - Name: DeployToProd
          Actions:
            - Name: DeployToProdAction
              ActionTypeId:
                Category: Build
                Owner: AWS
                Provider: CodeBuild
                Version: 1
              InputArtifacts:
                - Name: BuildOutput
              Configuration:
                ProjectName: DeployToProd
              RunOrder: 1
```

---

### **3. CodeBuild Projects**

#### **Build Project (GoMicroserviceBuild)**

**BuildSpec for Building and Pushing Docker Image:** `buildspec-build.yml`
```yaml
version: 0.2
phases:
  install:
    runtime-versions:
      docker: latest
      golang: 1.20
  pre_build:
    commands:
      - echo "Logging in to Amazon ECR..."
      - aws ecr get-login-password --region us-west-2 | docker login --username AWS --password-stdin <your-account-id>.dkr.ecr.us-west-2.amazonaws.com
  build:
    commands:
      - echo "Building the Docker image..."
      - docker build -t <your-service-name> .
      - IMAGE_TAG=$(git rev-parse --short HEAD)
      - echo "Tagging the Docker image..."
      - docker tag <your-service-name>:latest <your-account-id>.dkr.ecr.us-west-2.amazonaws.com/<your-service-name>:${IMAGE_TAG}
  post_build:
    commands:
      - echo "Pushing the Docker image to Amazon ECR..."
      - docker push <your-account-id>.dkr.ecr.us-west-2.amazonaws.com/<your-service-name>:${IMAGE_TAG}
artifacts:
  files:
    - kustomize/**
    - buildspec-deploy.yml
  discard-paths: no
```

---

#### **Deploy Project (DeployTo[Alpha|Beta|UAT|Prod])**

**BuildSpec for Applying Kustomize Overlays:** `buildspec-deploy.yml`
```yaml
version: 0.2
phases:
  install:
    runtime-versions:
      kubectl: 1.27
  pre_build:
    commands:
      - echo "Logging in to Amazon EKS..."
      - aws eks update-kubeconfig --name <your-cluster-name> --region us-west-2
  build:
    commands:
      - echo "Deploying to environment: $ENVIRONMENT"
      - cd kustomize/overlays/$ENVIRONMENT
      - kubectl apply -k .
```

Set the environment variable `ENVIRONMENT` to the respective environment (`alpha`, `beta`, `uat`, `prod`) for each deploy project.

---

### **4. Kustomize Configuration**

Keep the **Kustomize** configuration as explained in the earlier example with overlays for `alpha`, `beta`, `uat`, and `prod`. Ensure `IMAGE_TAG` is dynamically replaced in the deployment manifest.

---

### **5. Deploy Pipeline**

1. **Build Phase:**
    - Builds the Docker image once from the `main` branch.
    - Pushes the image to Amazon ECR.
    - Copies the `kustomize/` directory and deploy build spec (`buildspec-deploy.yml`) as artifacts.

2. **Deploy Phase:**
    - For each environment (`alpha`, `beta`, `uat`, `prod`):
        - Deploys the prebuilt Docker image using `kubectl apply -k` and the appropriate overlay.

---

### **6. Setting Up**

1. **Pipeline IAM Roles:**
    - Ensure the CodePipeline and CodeBuild roles have access to:
        - Amazon ECR (for pushing/pulling images).
        - Amazon EKS (for deploying applications).

2. **Trigger Pipeline:**
    - On a commit to `main`, the pipeline builds the image and pushes it to ECR.
    - On a merge to `alpha`, `beta`, `uat`, or `prod`, the pipeline deploys the prebuilt image to the respective EKS environment.

---

### **Benefits**

1. **Build Once, Run Everywhere:** Docker images are built once and reused across environments.
2. **Environment-Specific Deployments:** Kustomize overlays allow configuration changes per environment.
3. **AWS Native Integration:** Fully utilizes AWS services for pipeline, registry, and deployment.
4. **Scalability:** Add new environments or services with minimal configuration changes.





To visualize the "Build Once, Run Everywhere" strategy in a Git branching model, here's a description of the key elements and a diagram outline for your Markdown file:

### Key Features of the "Build Once, Run Everywhere" Strategy:
1. **Main Branch**: This is the default branch where the stable version of the code always resides. It serves as the baseline for all environments (e.g., development, testing, staging, production).
2. **Feature Branches**: Developers work on features or fixes in isolated branches based off the main branch. Once features are ready, they are merged back into the main branch.
3. **CI/CD Pipeline**: Once merged into the main branch, the pipeline triggers automated tests and builds. The same artifact is deployed to all environments with minimal adjustments.
4. **Minimal Overhead**: By keeping the main branch consistent across all environments and using CI/CD pipelines for deployment, this strategy reduces overhead, ensuring smooth transitions between environments.

### Git Diagram for "Build Once, Run Everywhere" Strategy

```plaintext
    +--------------------+
    |      Main Branch   |
    |  (Stable Version)  |
    +--------------------+
             |
             v
    +--------------------+     +---------------------+
    |  Feature Branches  | --> |  CI/CD Pipeline     |
    | (Develop Features) |     |  (Automated Build,  |
    +--------------------+     |  Tests, Deployments)|
             |                 +---------------------+
             v
    +---------------------+
    |  All Environments   |
    |  (Alpha, Beta, Prod)|
    +---------------------+
```

### Steps in the Flow:
1. Developers create feature branches from the **Main Branch**.
2. Once the feature is complete, it is merged back into the **Main Branch**.
3. The **CI/CD Pipeline** automatically triggers the build, runs tests, and deploys the same artifact to all environments (Alpha, Beta, Production).
4. Since the same code and artifacts are used across all environments, overhead is minimal and the "Build Once, Run Everywhere" philosophy is maintained.




To update the Git diagram in the context of containerization, AWS EKS, and a Go project, we can integrate the following elements:

### Key Updates:
1. **Containerization**: The Go project is containerized (e.g., using Docker) so that the same container image is deployed across all environments.
2. **AWS EKS**: The containerized Go application is deployed to AWS EKS (Elastic Kubernetes Service) for orchestration.
3. **CI/CD with Container Registry**: The CI/CD pipeline builds the container image and pushes it to a container registry (like Amazon ECR). The Kubernetes deployment in AWS EKS then pulls this image to run the application across various environments.
4. **Minimal Overhead**: By using Kubernetes with EKS, the application is managed and scaled efficiently with minimal overhead, and the container image used across all environments ensures consistency.

### Updated Git Diagram with Containerization and AWS EKS

```plaintext
    +------------------------+
    |      Main Branch       |
    |  (Stable Version Code) |
    +------------------------+
             |
             v
    +------------------------+
    |  Feature Branches      |
    |  (Develop Features)    |
    +------------------------+
             |
             v
    +------------------------+
    |  CI/CD Pipeline        |
    |  (Build Docker Image,  |
    |   Push to Container    |
    |   Registry - ECR)      |
    +------------------------+
             |
             v
    +-------------------------+        +---------------------+
    |  Container Registry     | -----> |  EKS (Kubernetes)   |
    |   (Amazon ECR)          |        |   (All Environments)|
    +-------------------------+        +---------------------+
             |
             v
    +---------------------+
    |  Environments       |
    | (Alpha, Beta, Prod) |
    +---------------------+
```

### Steps in the Updated Flow:
1. Developers create feature branches from the **Main Branch** to work on the Go application features.
2. Once the feature is ready, it is merged back into the **Main Branch**.
3. The **CI/CD Pipeline** is triggered, which:
    - Builds the Docker image for the Go application.
    - Pushes the image to a **Container Registry** (e.g., Amazon ECR).
4. **AWS EKS (Kubernetes)** pulls the container image from the registry and deploys it across different environments (Alpha, Beta, Production).
5. Since the same container image is used in all environments, the application remains consistent with minimal overhead.

### Summary:
- **Containerization** ensures that the application runs in a consistent environment, independent of the underlying infrastructure.
- **AWS EKS** simplifies the management of these containers with auto-scaling and orchestration, while also ensuring that the same Docker image is deployed across all environments.
- **CI/CD** automates the build and deployment process, minimizing manual overhead.


## "Build Once, Run Everywhere"

Here's a simple Git workflow example with relevant Git commands to support the "Build Once, Run Everywhere" strategy using containerization (Docker) and AWS EKS for a Go project. This includes typical Git commands, containerization steps, and integration with CI/CD (like GitHub Actions or AWS CodePipeline).

### 1. **Initial Setup**

- Create a **Go project** repository with a simple `Dockerfile` and Kubernetes deployment files.
- Push the repository to GitHub or any Git service.

### 2. **Folder Structure Example**
```plaintext
my-go-project/
├── .github/
│   └── workflows/
│       └── ci-cd-pipeline.yml  (GitHub Actions file for CI/CD)
├── Dockerfile
├── k8s/
│   └── deployment.yaml       (Kubernetes deployment file)
├── main.go
└── README.md
```

### 3. **Main Branch and Feature Branch Workflow**
#### 3.1. Create a feature branch from the main branch:
```bash
git checkout main
git pull origin main
git checkout -b feature/add-login-functionality
```

#### 3.2. Implement the feature in the Go project:
- Add or modify Go code in `main.go` or create new files as needed.

#### 3.3. Commit and push the feature branch:
```bash
git add .
git commit -m "Add login functionality"
git push origin feature/add-login-functionality
```

#### 3.4. Create a pull request (PR) to merge the feature into the **main branch**:
- Open your Git repository in GitHub and create a PR to merge `feature/add-login-functionality` into the `main` branch.

#### 3.5. Once the PR is merged, checkout the main branch:
```bash
git checkout main
git pull origin main
```

### 4. **CI/CD Pipeline (GitHub Actions Example)**

In `.github/workflows/ci-cd-pipeline.yml`, define the CI/CD pipeline:

```yaml
name: CI/CD Pipeline for Go Project

on:
  push:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout Code
      uses: actions/checkout@v2

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v1

    - name: Log in to Amazon ECR
      uses: aws-actions/amazon-ecr-login@v1

    - name: Build and Push Docker Image
      run: |
        docker build -t ${{ secrets.ECR_REPO_URI }}:$GITHUB_SHA .
        docker push ${{ secrets.ECR_REPO_URI }}:$GITHUB_SHA

    - name: Deploy to EKS
      run: |
        kubectl apply -f k8s/deployment.yaml
```

### 5. **Dockerfile Example**
Create a `Dockerfile` to containerize your Go application:

```Dockerfile
# Use the official Go image as the base image
FROM golang:1.20 as builder

# Set the current working directory inside the container
WORKDIR /app

# Copy the Go modules manifests
COPY go.mod go.sum ./

# Download Go dependencies
RUN go mod tidy

# Copy the source code into the container
COPY . .

# Build the Go binary
RUN go build -o my-go-app .

# Create a smaller image for runtime
FROM alpine:latest

WORKDIR /root/

# Copy the binary from the build image
COPY --from=builder /app/my-go-app .

# Expose the port the app will run on
EXPOSE 8080

# Run the Go binary
CMD ["./my-go-app"]
```

### 6. **Kubernetes Deployment (EKS)**
Example of a Kubernetes deployment file (`k8s/deployment.yaml`) to deploy the Go container in AWS EKS:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-go-app-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-go-app
  template:
    metadata:
      labels:
        app: my-go-app
    spec:
      containers:
      - name: my-go-app
        image: <AWS_ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com/my-go-app:${GITHUB_SHA}
        ports:
        - containerPort: 8080
```

### 7. **Pushing to ECR**

#### 7.1. Login to Amazon ECR (with AWS CLI)
Make sure you have AWS CLI installed and configured, and your ECR repository is created.

```bash
aws ecr get-login-password --region us-west-2 | docker login --username AWS --password-stdin <AWS_ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com
```

#### 7.2. Push the Docker Image to ECR
After building the Docker image locally, push it to Amazon ECR:

```bash
docker tag my-go-app:latest <AWS_ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com/my-go-app:${GITHUB_SHA}
docker push <AWS_ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com/my-go-app:${GITHUB_SHA}
```

### 8. **EKS Deployment**

The **Kubernetes deployment file** (`k8s/deployment.yaml`) will ensure that your containerized Go application is deployed on AWS EKS. The `kubectl apply` command from the GitHub Actions pipeline will trigger the deployment to EKS.

### Summary of Steps:
1. **Develop features** in isolated feature branches.
2. **Merge** the feature branches into the `main` branch.
3. The **CI/CD pipeline** builds the Docker container and pushes it to ECR.
4. **EKS** automatically deploys the latest container image to the defined environments (e.g., Alpha, Beta, Production).

This is the general workflow, and you can modify it based on your specific needs (e.g., use AWS CodePipeline instead of GitHub Actions, customize deployment files, etc.).


## Build Once, Run Everywhere 2

To adapt the "Build Once, Run Everywhere" strategy to a **monthly release cycle**, 
you would introduce a few modifications to the Git workflow, branching model, and CI/CD pipeline. The key changes would include:

- **Release Branch**: You’ll create a release branch that serves as a staging area for final testing before the monthly release.
- **Feature Freeze**: Towards the end of the month, a "feature freeze" will happen, where only critical fixes are allowed in the release branch.
- **Release Tagging**: The release will be tagged to mark the official version for deployment.

### 1. **Updated Git Branching Model**
- **Main Branch**: The main branch continues to represent the stable version of the application.
- **Feature Branches**: Developers work on feature branches from the main branch.
- **Release Branch**: At the end of the month (e.g., on the 28th), a release branch is created from the main branch, which will undergo final testing and bug fixes.
- **Tagging**: After the release branch is ready, a tag is created (e.g., `v1.0.0`) to mark the release version.
- **Production Deployment**: The tagged release will be deployed to production.

### 2. **Folder Structure Example**
```plaintext
my-go-project/
├── .github/
│   └── workflows/
│       └── ci-cd-pipeline.yml  (GitHub Actions file for CI/CD)
├── Dockerfile
├── k8s/
│   └── deployment.yaml       (Kubernetes deployment file)
├── main.go
└── README.md
```

### 3. **Git Workflow for Monthly Release Cycle**

#### 3.1. **Feature Branches** (Work in Progress)
- Developers continue working on features in separate branches.
```bash
git checkout main
git pull origin main
git checkout -b feature/add-login-functionality
# Develop your features...
git push origin feature/add-login-functionality
```

#### 3.2. **End of the Month - Create a Release Branch**
At the end of the month, create a **release branch**. This is done once the features for the month have been merged into the `main` branch, and no further features will be added until the next cycle.

```bash
git checkout main
git pull origin main
git checkout -b release/v1.0.0
# Make sure any feature branch is merged into the release branch
git merge feature/add-login-functionality
git push origin release/v1.0.0
```

#### 3.3. **Final Testing and Bug Fixes**
Once the release branch is created, focus on fixing any bugs or issues. No new features will be added to this branch.

```bash
# Make bug fixes and small improvements
git add .
git commit -m "Bug fix for login functionality"
git push origin release/v1.0.0
```

#### 3.4. **Tag the Release**
Once the release branch is stable, create a **release tag** to mark the release version (e.g., `v1.0.0`).

```bash
git checkout release/v1.0.0
git pull origin release/v1.0.0
git tag v1.0.0
git push origin v1.0.0
```

#### 3.5. **Merge the Release Branch Back into Main and Development**
After the release is tagged, merge the release branch back into both the `main` branch (for future development) and the `develop` branch (if you're following this approach).

```bash
# Merge release into main for the latest stable release
git checkout main
git merge release/v1.0.0
git push origin main

# Optionally, merge back into develop branch (if used)
git checkout develop
git merge release/v1.0.0
git push origin develop
```

#### 3.6. **CI/CD Pipeline**
The pipeline now has additional steps for the **release cycle**. Once the tag is pushed, it will trigger the build and deployment of the release.

### 4. **CI/CD Pipeline for Monthly Release Cycle**

In `.github/workflows/ci-cd-pipeline.yml`, update the GitHub Actions workflow for the monthly release cycle:

```yaml
name: CI/CD Pipeline for Monthly Release Cycle

on:
  push:
    branches:
      - main
      - release/*
    tags:
      - 'v*'

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout Code
      uses: actions/checkout@v2

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v1

    - name: Log in to Amazon ECR
      uses: aws-actions/amazon-ecr-login@v1

    - name: Build and Push Docker Image
      run: |
        docker build -t ${{ secrets.ECR_REPO_URI }}:$GITHUB_SHA .
        docker push ${{ secrets.ECR_REPO_URI }}:$GITHUB_SHA

    - name: Deploy to EKS (For Release Tag Only)
      if: startsWith(github.ref, 'refs/tags/v')
      run: |
        kubectl apply -f k8s/deployment.yaml
        kubectl rollout restart deployment/my-go-app-deployment
```

### 5. **Dockerfile Example (No Changes Required)**
The Dockerfile remains the same. It will be used to build the container image for all releases:

```Dockerfile
# Use the official Go image as the base image
FROM golang:1.20 as builder

# Set the current working directory inside the container
WORKDIR /app

# Copy the Go modules manifests
COPY go.mod go.sum ./

# Download Go dependencies
RUN go mod tidy

# Copy the source code into the container
COPY . .

# Build the Go binary
RUN go build -o my-go-app .

# Create a smaller image for runtime
FROM alpine:latest

WORKDIR /root/

# Copy the binary from the build image
COPY --from=builder /app/my-go-app .

# Expose the port the app will run on
EXPOSE 8080

# Run the Go binary
CMD ["./my-go-app"]
```

### 6. **Kubernetes Deployment (No Changes Required)**
The Kubernetes deployment file (`k8s/deployment.yaml`) remains unchanged. It is used to deploy the containerized Go application to AWS EKS:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-go-app-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-go-app
  template:
    metadata:
      labels:
        app: my-go-app
    spec:
      containers:
      - name: my-go-app
        image: <AWS_ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com/my-go-app:${GITHUB_SHA}
        ports:
        - containerPort: 8080
```

### 7. **EKS Deployment (No Changes Required)**

After the **CI/CD pipeline** builds the Docker image and pushes it to the container registry, the pipeline deploys the tagged release image to **AWS EKS** using Kubernetes. The Kubernetes deployment file will always pull the container image based on the release tag.

---

### Summary of Monthly Release Cycle Steps:

1. **Feature Development**: Features are developed in feature branches and merged into the `main` branch.
2. **Release Branch Creation**: At the end of each month, create a `release/vX.Y.Z` branch for final testing and bug fixing.
3. **Tag the Release**: Once stable, tag the release (e.g., `v1.0.0`).
4. **CI/CD Pipeline**:
    - Builds and pushes Docker images to ECR.
    - Deploys the release version to AWS EKS.
5. **Deploy to All Environments**: The release version is deployed to all environments (e.g., Alpha, Beta, and Production) in AWS EKS.

This workflow supports a **monthly release cycle** with minimal overhead while maintaining the "Build Once, Run Everywhere" principle.

## Build Once, Run Everywhere 3

To update the workflow for **multiple releases within a month**, you would adjust the Git branching model and CI/CD pipeline to accommodate several releases each month. This can be achieved by handling **feature, release, and hotfix branches** in parallel and allowing for versioning through tags for each release.

Here’s how to modify the process for **multiple releases per month**:

### Key Changes for Multiple Releases:
1. **Feature Branches**: Continue to work on feature branches, but ensure they are integrated frequently into the `main` branch.
2. **Release Branches**: Instead of one long-lived `release` branch, multiple `release/` branches are created as needed (e.g., `release/v1.0.0`, `release/v1.1.0`).
3. **Release Tags**: Each release (monthly) will get its own version number and tag (e.g., `v1.0.0`, `v1.1.0`).
4. **Hotfixes**: If a critical issue arises, create a **hotfix branch** to address the problem and release it quickly without waiting for the next feature cycle.

### 1. **Git Branching Model for Multiple Monthly Releases**
- **Main Branch**: The primary development branch where all changes are merged.
- **Feature Branches**: Work continues on new features, which are merged into `main`.
- **Release Branches**: Created periodically for releases (e.g., `release/v1.0.0`, `release/v1.1.0`).
- **Hotfix Branches**: Temporary branches for critical fixes to be released before the next full release.

### 2. **Folder Structure Example**
```plaintext
my-go-project/
├── .github/
│   └── workflows/
│       └── ci-cd-pipeline.yml  (GitHub Actions file for CI/CD)
├── Dockerfile
├── k8s/
│   └── deployment.yaml       (Kubernetes deployment file)
├── main.go
└── README.md
```

### 3. **Git Workflow for Multiple Releases Monthly**

#### 3.1. **Feature Development** (Ongoing)
- Developers create feature branches and work on features.
```bash
git checkout main
git pull origin main
git checkout -b feature/add-login-functionality
# Develop your feature...
git push origin feature/add-login-functionality
```

#### 3.2. **Create Release Branches** (Periodic Releases)
- When the application is ready for a release (multiple times a month), create a `release` branch from `main`. This release branch will be used for final testing, bug fixes, and preparation for deployment.
```bash
git checkout main
git pull origin main
git checkout -b release/v1.0.0
# Merge completed feature branches into this release branch
git merge feature/add-login-functionality
git push origin release/v1.0.0
```

#### 3.3. **Final Testing and Bug Fixes**
Once the release branch is created, make any final bug fixes or improvements.
```bash
# Make fixes and improvements
git add .
git commit -m "Fix issue with login"
git push origin release/v1.0.0
```

#### 3.4. **Tagging the Release**
Once the release branch is stable, tag the release with a version number (e.g., `v1.0.0`).
```bash
git checkout release/v1.0.0
git pull origin release/v1.0.0
git tag v1.0.0
git push origin v1.0.0
```

#### 3.5. **Merge Release Branch Back Into Main**
After the release tag is created, merge the `release` branch back into the `main` branch for the next feature cycle.
```bash
git checkout main
git merge release/v1.0.0
git push origin main
```

#### 3.6. **Hotfixes (If Needed)**
If a critical bug is found after a release, create a **hotfix branch** based on the release branch, fix the issue, and deploy it.
```bash
git checkout release/v1.0.0
git checkout -b hotfix/fix-login-issue
# Fix the issue...
git add .
git commit -m "Fix login issue"
git push origin hotfix/fix-login-issue

# Tag the hotfix release
git tag v1.0.1
git push origin v1.0.1

# Merge back into main and release branch
git checkout main
git merge hotfix/fix-login-issue
git push origin main

git checkout release/v1.0.0
git merge hotfix/fix-login-issue
git push origin release/v1.0.0
```

### 4. **CI/CD Pipeline for Multiple Releases**

In `.github/workflows/ci-cd-pipeline.yml`, update the workflow to handle **multiple releases** within the month. The pipeline will build and deploy whenever there is a push to the `main` or `release/*` branches, or a version tag (e.g., `v1.0.0`).

```yaml
name: CI/CD Pipeline for Multiple Releases

on:
  push:
    branches:
      - main
      - release/*
    tags:
      - 'v*'

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout Code
      uses: actions/checkout@v2

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v1

    - name: Log in to Amazon ECR
      uses: aws-actions/amazon-ecr-login@v1

    - name: Build and Push Docker Image
      run: |
        docker build -t ${{ secrets.ECR_REPO_URI }}:$GITHUB_SHA .
        docker push ${{ secrets.ECR_REPO_URI }}:$GITHUB_SHA

    - name: Deploy to EKS (For Release Tag Only)
      if: startsWith(github.ref, 'refs/tags/v')
      run: |
        kubectl apply -f k8s/deployment.yaml
        kubectl rollout restart deployment/my-go-app-deployment
```

### 5. **Dockerfile Example (No Changes Required)**
The `Dockerfile` remains the same. It will be used to build the container image for all releases.

```Dockerfile
# Use the official Go image as the base image
FROM golang:1.20 as builder

# Set the current working directory inside the container
WORKDIR /app

# Copy the Go modules manifests
COPY go.mod go.sum ./

# Download Go dependencies
RUN go mod tidy

# Copy the source code into the container
COPY . .

# Build the Go binary
RUN go build -o my-go-app .

# Create a smaller image for runtime
FROM alpine:latest

WORKDIR /root/

# Copy the binary from the build image
COPY --from=builder /app/my-go-app .

# Expose the port the app will run on
EXPOSE 8080

# Run the Go binary
CMD ["./my-go-app"]
```

### 6. **Kubernetes Deployment (No Changes Required)**
The Kubernetes deployment file (`k8s/deployment.yaml`) remains the same. It ensures that the latest version of the Docker container is deployed to **AWS EKS**.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-go-app-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-go-app
  template:
    metadata:
      labels:
        app: my-go-app
    spec:
      containers:
      - name: my-go-app
        image: <AWS_ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com/my-go-app:${GITHUB_SHA}
        ports:
        - containerPort: 8080
```

### 7. **EKS Deployment (No Changes Required)**

The **EKS deployment** process will remain as it was, where **AWS EKS** pulls the container image based on the tag version from **Amazon ECR** and deploys the updated container to the specified environments.

---

### Summary of Workflow for Multiple Monthly Releases:
1. **Feature Development**: Features are developed continuously and merged into `main`.
2. **Release Branch Creation**: For each release (e.g., `v1.0.0`, `v1.1.0`), a release branch is created from `main`.
3. **Tagging**: Each release (e.g., `v1.0.0`, `v1.1.0`) gets its own tag.
4. **Hotfixes**: Critical fixes are applied via a hotfix branch and tagged as necessary (e.g., `v1.0.1`).
5. **CI/CD Pipeline**: The pipeline builds and deploys the Docker container to EKS whenever a release tag or push to `release/*` happens.

By using this model, you can release multiple times each month and address both new features and critical issues (via hotfixes) with minimal disruption.


## Build Once, Run Everywhere 4

To update the Git workflow and CI/CD pipeline for **multiple releases** with **AWS CodePipeline** and integrate it with the **"Build Once, Run Everywhere"** strategy using **containerization** (Docker) and **AWS EKS**, here's how you would modify the process.

### Key Changes for AWS CodePipeline Integration:
1. **AWS CodeCommit (or GitHub integration)**: You can either use **AWS CodeCommit** for your Git repository or continue using **GitHub** and integrate it with **AWS CodePipeline**.
2. **Pipeline Stages**: AWS CodePipeline will have stages to:
    - **Build**: Use **AWS CodeBuild** to build the Docker container.
    - **Push**: Push the container image to **Amazon ECR**.
    - **Deploy**: Deploy the container image to **AWS EKS** using **kubectl**.

### 1. **AWS CodePipeline Setup for Multiple Releases**

The pipeline will be triggered when:
- There’s a push to the **`release/*`** branches or when a **tag** (like `v1.0.0`) is created for a release.

### 2. **Folder Structure Example**
```plaintext
my-go-project/
├── .github/
│   └── workflows/
│       └── ci-cd-pipeline.yml  (GitHub Actions file for CI/CD if you use GitHub)
├── Dockerfile
├── k8s/
│   └── deployment.yaml       (Kubernetes deployment file)
├── main.go
└── README.md
```

### 3. **Git Workflow (Unchanged)**

This workflow remains the same as before, where:
- Developers create and work on feature branches.
- Release branches are created at the end of the month or as required for multiple releases within the month.
- **Tags** (`v1.0.0`, `v1.1.0`, etc.) are created for each release.

### 4. **Set Up AWS CodePipeline**

To automate the **build** and **deployment** process in **AWS**, you can set up **AWS CodePipeline**.

#### 4.1. **Pipeline Structure**:
1. **Source Stage**:
    - **AWS CodeCommit** or **GitHub** will act as the source for the pipeline. When a new release branch is pushed or a tag is created, the pipeline will trigger.

2. **Build Stage (AWS CodeBuild)**:
    - **AWS CodeBuild** will build the **Docker image** and push it to **Amazon ECR**.

3. **Deploy Stage (AWS CodeDeploy / kubectl)**:
    - The image from Amazon ECR will be deployed to **AWS EKS** using **kubectl** commands.

#### 4.2. **Steps to Set Up AWS CodePipeline for Multiple Releases**

##### 4.2.1. **Source Stage: GitHub or CodeCommit Integration**
- In **AWS CodePipeline**, the **Source** stage listens for changes in your Git repository (GitHub or CodeCommit).
- If using **GitHub**, you would connect AWS CodePipeline to your GitHub repository and specify the branch or tag (e.g., `release/*` or `v*`).

```yaml
# Example for CodePipeline Source stage:
SourceAction:
  Name: GitHubSource
  ActionTypeId:
    Category: Source
    Owner: AWS
    Provider: GitHub
    Version: '1'
  Configuration:
    Owner: <GitHubUser>
    Repo: <RepositoryName>
    Branch: release/*
  OutputArtifacts:
    - Name: SourceOutput
```

##### 4.2.2. **Build Stage (AWS CodeBuild)**
- The **Build** stage uses **AWS CodeBuild** to build the Docker image and push it to Amazon ECR.

Create a **buildspec.yml** file in your project root to define the steps for **CodeBuild**:

```yaml
version: 0.2

phases:
  build:
    commands:
      # Login to ECR
      - $(aws ecr get-login --no-include-email --region us-west-2)
      
      # Build the Docker image
      - docker build -t my-go-app:$CODEBUILD_RESOLVED_SOURCE_VERSION .
      
      # Tag the Docker image with the commit hash or release version
      - docker tag my-go-app:$CODEBUILD_RESOLVED_SOURCE_VERSION <AWS_ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com/my-go-app:$CODEBUILD_RESOLVED_SOURCE_VERSION
      
      # Push the image to ECR
      - docker push <AWS_ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com/my-go-app:$CODEBUILD_RESOLVED_SOURCE_VERSION
artifacts:
  files:
    - '**/*'
```

##### 4.2.3. **Deploy Stage (AWS EKS)**
The **Deploy** stage uses **kubectl** to deploy the Docker image to **AWS EKS**. Here’s how you can do it:

- Use **AWS CodePipeline** to execute **AWS Lambda** or **AWS CodeDeploy** for the deployment.
- Alternatively, use a **manual action** to run **kubectl** commands directly to trigger the deployment in **AWS EKS**.

Create an **AWS Lambda function** or use an **AWS CodeDeploy** action to deploy the image to your EKS cluster:

```bash
# Example kubectl command to deploy to EKS:
kubectl set image deployment/my-go-app-deployment my-go-app=<AWS_ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com/my-go-app:$CODEBUILD_RESOLVED_SOURCE_VERSION --record
kubectl rollout restart deployment/my-go-app-deployment
```

Here’s an example of what the **Deploy Stage** might look like:

```yaml
DeployAction:
  Name: DeployToEKS
  ActionTypeId:
    Category: Deploy
    Owner: AWS
    Provider: EKS
    Version: '1'
  Configuration:
    ClusterName: <your-cluster-name>
    Kubeconfig: <path-to-kubeconfig> # If you're using a custom config file
  InputArtifacts:
    - Name: SourceOutput
  OutputArtifacts: []
  RunOrder: 1
```

Alternatively, you can use **AWS Lambda** to run the **kubectl** commands that interact with your **EKS** cluster.

### 5. **Dockerfile for ECR and EKS**

The **Dockerfile** remains unchanged. It will continue to build your Go application as a container.

```Dockerfile
# Use the official Go image as the base image
FROM golang:1.20 as builder

# Set the current working directory inside the container
WORKDIR /app

# Copy the Go modules manifests
COPY go.mod go.sum ./

# Download Go dependencies
RUN go mod tidy

# Copy the source code into the container
COPY . .

# Build the Go binary
RUN go build -o my-go-app .

# Create a smaller image for runtime
FROM alpine:latest

WORKDIR /root/

# Copy the binary from the build image
COPY --from=builder /app/my-go-app .

# Expose the port the app will run on
EXPOSE 8080

# Run the Go binary
CMD ["./my-go-app"]
```

### 6. **Kubernetes Deployment (EKS)**

The **Kubernetes deployment** file (`k8s/deployment.yaml`) remains the same. It ensures that the latest version of the Docker container is deployed to **AWS EKS**:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-go-app-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-go-app
  template:
    metadata:
      labels:
        app: my-go-app
    spec:
      containers:
      - name: my-go-app
        image: <AWS_ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com/my-go-app:${CODEBUILD_RESOLVED_SOURCE_VERSION}
        ports:
        - containerPort: 8080
```

### 7. **Pipeline for Multiple Releases**

**AWS CodePipeline** allows the deployment of multiple releases within the same month. For example, you could have a release on the 1st (`v1.0.0`), another on the 15th (`v1.1.0`), and a hotfix on the 30th (`v1.0.1`).

Each of these releases would trigger the pipeline, where the **build** and **deployment** stages are executed separately.

---

### Summary of Workflow for Multiple Monthly Releases in AWS CodePipeline:

1. **Feature Development**: Features are developed and pushed to the `main` branch.
2. **Release Branches**: When ready for release, a `release/vX.Y.Z` branch is created. Multiple releases can be created within a month (e.g., `v1.0.0`, `v1.1.0`).
3. **Tagging**: Each release is tagged (e.g., `v1.0.0`, `v1.1.0`).
4. **CI/CD Pipeline**: AWS CodePipeline automates:
    - **Source Stage**: Detects changes in the Git repository.
    - **Build Stage**: Uses **AWS CodeBuild** to build and push Docker images to **Amazon ECR**.
    - **Deploy Stage**: Deploys the image to **AWS EKS** using **kubectl** or AWS Lambda.
5. **Hotfixes**: Critical issues can be fixed via **hotfix branches** (e.g., `hotfix/fix-login-issue`), tagged, and deployed quickly.

This setup automates and streamlines the deployment process for **multiple releases** every month, minimizing overhead and ensuring consistency in all environments.

## Build Once, Run Everywhere 5

To adapt the **Build Once, Run Everywhere** strategy with **separate Kubernetes manifests** for each environment (like Alpha, Beta, and Production), we can structure the repository and pipeline such that the Kubernetes manifest files for different environments are stored in **separate repositories** or in **different directories** within the same repository.

This approach enables the same Docker image to be deployed across multiple environments with environment-specific configurations. The idea is to **build the Docker container once**, and then use Kubernetes manifests to define how it should run in different environments.

Here’s how to structure this workflow with separate repos for Kubernetes manifests per environment:

### Key Concepts:
- **Docker Image**: Built once for all environments.
- **Environment-Specific Kubernetes Manifests**: Different Kubernetes manifest files will be used for each environment (e.g., `alpha`, `beta`, `prod`).
- **Git Repositories**:
    - **Main Repo**: Contains the application code and Dockerfile.
    - **Alpha Repo, Beta Repo, Prod Repo (or subfolders)**: Contain Kubernetes manifests specific to each environment.

### 1. **Folder Structure Example (Multiple Repositories)**

- **Main Application Repo** (contains application code and Dockerfile):
```plaintext
my-go-project/
├── Dockerfile
├── main.go
├── go.mod
├── go.sum
└── README.md
```

- **Separate Environment Repos or Directories** (each environment repo or folder contains environment-specific Kubernetes manifests):
    - `alpha-repo/` (Kubernetes manifests for Alpha environment)
    - `beta-repo/` (Kubernetes manifests for Beta environment)
    - `prod-repo/` (Kubernetes manifests for Prod environment)

```plaintext
alpha-repo/
├── k8s/
│   ├── deployment.yaml  (Alpha environment-specific deployment)
│   ├── service.yaml     (Alpha service config)
└── README.md
```

```plaintext
beta-repo/
├── k8s/
│   ├── deployment.yaml  (Beta environment-specific deployment)
│   ├── service.yaml     (Beta service config)
└── README.md
```

```plaintext
prod-repo/
├── k8s/
│   ├── deployment.yaml  (Production environment-specific deployment)
│   ├── service.yaml     (Production service config)
└── README.md
```

### 2. **Git Workflow (Main Repo)**

- Developers will push code to the **main repository**, where all the features and application logic reside.
- Once the features are merged into the **main branch**, the **release branch** is created for the new release, tagged, and pushed.
- The **CI/CD pipeline** (using AWS CodePipeline) will build the Docker container and push it to **Amazon ECR**.

#### 2.1. **Git Workflow for Feature Development**:

```bash
# Create a feature branch
git checkout main
git pull origin main
git checkout -b feature/add-login-functionality
# Work on feature, commit, and push
git add .
git commit -m "Add login functionality"
git push origin feature/add-login-functionality
```

#### 2.2. **Create Release Branch** (Once the feature is complete):
```bash
git checkout main
git pull origin main
git checkout -b release/v1.0.0
# Merge features into release branch
git merge feature/add-login-functionality
git push origin release/v1.0.0
```

#### 2.3. **Tag the Release**:
```bash
git tag v1.0.0
git push origin v1.0.0
```

### 3. **CI/CD Pipeline for Multiple Environments**

The **CI/CD pipeline** should be set up to build the Docker image once and then deploy it to **multiple Kubernetes environments** (e.g., Alpha, Beta, Prod).

#### 3.1. **Build Docker Image (AWS CodeBuild)**:
In the **CodePipeline** build stage, **AWS CodeBuild** will build the Docker image and push it to **Amazon ECR**.

**buildspec.yml** (in the main repository):

```yaml
version: 0.2

phases:
  build:
    commands:
      # Login to ECR
      - $(aws ecr get-login --no-include-email --region us-west-2)
      
      # Build Docker image
      - docker build -t my-go-app:$CODEBUILD_RESOLVED_SOURCE_VERSION .
      
      # Tag Docker image with the version
      - docker tag my-go-app:$CODEBUILD_RESOLVED_SOURCE_VERSION <AWS_ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com/my-go-app:$CODEBUILD_RESOLVED_SOURCE_VERSION
      
      # Push Docker image to ECR
      - docker push <AWS_ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com/my-go-app:$CODEBUILD_RESOLVED_SOURCE_VERSION

artifacts:
  files:
    - '**/*'
```

#### 3.2. **Deployment to EKS with Kubernetes Manifests (per Environment)**:
The **AWS CodePipeline** will deploy the built Docker image to the appropriate **EKS cluster** using environment-specific Kubernetes manifests. The Kubernetes manifests are stored in separate repositories for each environment (Alpha, Beta, and Prod).

For example, the **Alpha environment** may have the following **deployment manifest**:

**alpha-repo/k8s/deployment.yaml**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-go-app-deployment-alpha
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-go-app
  template:
    metadata:
      labels:
        app: my-go-app
    spec:
      containers:
      - name: my-go-app
        image: <AWS_ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com/my-go-app:${CODEBUILD_RESOLVED_SOURCE_VERSION}
        ports:
        - containerPort: 8080
```

The **Beta** and **Prod** environments will have similar manifests but with different settings (e.g., scaling, resource limits, namespaces).

For **AWS CodePipeline** to deploy, you can use **AWS Lambda**, **AWS CodeDeploy**, or direct **kubectl** commands to interact with the Kubernetes environments.

#### 3.3. **AWS CodePipeline Stages**:
1. **Source Stage**: Triggers the pipeline when a new version (release tag or branch) is pushed to the repository.
    - This can be configured to watch specific branches or tags (e.g., `release/*` or `v*`).
2. **Build Stage (AWS CodeBuild)**: Builds the Docker container and pushes it to **Amazon ECR**.
3. **Deploy Stages** (AWS CodeDeploy or kubectl):
    - Separate **Deploy** stages for each environment (Alpha, Beta, Prod), where the correct Kubernetes manifests for each environment are applied using **kubectl** or **AWS CodeDeploy**.

Example **AWS CodePipeline YAML for deploy stages**:

```yaml
DeployToAlpha:
  Name: DeployToAlpha
  ActionTypeId:
    Category: Deploy
    Owner: AWS
    Provider: EKS
    Version: '1'
  Configuration:
    ClusterName: alpha-cluster
    Kubeconfig: <path-to-kubeconfig>
  InputArtifacts:
    - Name: SourceOutput
  OutputArtifacts: []
  RunOrder: 1

DeployToBeta:
  Name: DeployToBeta
  ActionTypeId:
    Category: Deploy
    Owner: AWS
    Provider: EKS
    Version: '1'
  Configuration:
    ClusterName: beta-cluster
    Kubeconfig: <path-to-kubeconfig>
  InputArtifacts:
    - Name: SourceOutput
  OutputArtifacts: []
  RunOrder: 2

DeployToProd:
  Name: DeployToProd
  ActionTypeId:
    Category: Deploy
    Owner: AWS
    Provider: EKS
    Version: '1'
  Configuration:
    ClusterName: prod-cluster
    Kubeconfig: <path-to-kubeconfig>
  InputArtifacts:
    - Name: SourceOutput
  OutputArtifacts: []
  RunOrder: 3
```

#### 3.4. **Manual Approval (Optional)**:
You can also include a **manual approval** stage before deploying to the **Prod** environment. This ensures that all releases are properly reviewed before being pushed to production.

### 4. **Kubernetes Deployment in Each Repo**

The **Kubernetes manifest files** in each environment repo (`alpha-repo`, `beta-repo`, `prod-repo`) should be adjusted according to each environment's specific requirements, like resource allocations, scaling, namespaces, etc.

For example:

- **Alpha** might have smaller resource allocations for testing.
- **Beta** might have more replicas for stress testing.
- **Prod** will have the final configurations for scaling and redundancy.

### 5. **EKS Deployment Process** (No Changes Required)

Once the image is pushed to **Amazon ECR** and the pipeline reaches the **deploy stage**, the Kubernetes manifest files specific to each environment will be applied to **EKS**.

You can automate the deployment with **kubectl** commands within the **AWS CodePipeline** deploy stages, or use **AWS CodeDeploy** for more sophisticated deployment strategies.

### Summary of Workflow for Multiple Environments:

1. **Main Repo** (Application Code): Contains all the source code, including the Dockerfile and Go application logic.
2. **Separate Repo/Directories for Kubernetes Manifests**: Each environment has its own repo or directory with environment-specific Kubernetes manifests.
3. **CI/CD Pipeline (AWS CodePipeline)**:
    - **Source Stage**: Watches changes in the Git repository (e.g., GitHub or CodeCommit).
    - **Build Stage (AWS CodeBuild)**: Builds the Docker image and pushes it to **Amazon ECR**.
    - **Deploy Stage**: Deploys the Docker image to **AWS EKS** using environment-specific Kubernetes manifests.
4. **Manual Approval (Optional)**: For Prod, an approval step can be added to ensure that the release is verified before deployment.

