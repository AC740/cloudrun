# Deploy a Python App to Google Cloud Run via GitHub Actions

This repository contains a simple Python Flask application and a GitHub Actions workflow to automatically build a container and deploy it to Google Cloud Run on every push to the `main` branch.

The setup uses a secure, keyless authentication method called **Workload Identity Federation** to grant GitHub Actions access to Google Cloud.

## File Structure

Your repository should contain the following files at the root:

```
.
├── .github/workflows/
│   └── google-cloudrun-source.yml
├── Dockerfile
├── main.py
└── requirements.txt
```

-----

## One-Time Setup on Google Cloud

The following `gcloud` commands only need to be run once to set up the necessary infrastructure and permissions in your Google Cloud project.

### 1\. Set Up Environment Variables

Run these in your Cloud Shell or local terminal to make the next steps easier.

```bash
# --- UPDATE THESE VALUES ---
export PROJECT_ID="[your project id]"
export REGION="us-central1"
export GITHUB_REPO="AC740/quest-gcloud" # Your GitHub Org/User and Repo name
export SERVICE_ACCOUNT_NAME="github-deploy-sa" # A name for your new service account
export ARTIFACT_REPO="cloudrun-repo" # A name for your new Artifact Registry repo

# --- NO CHANGES NEEDED BELOW ---
export SERVICE_ACCOUNT_EMAIL="${SERVICE_ACCOUNT_NAME}@${PROJECT_ID}.iam.gserviceaccount.com"
export PROJECT_NUMBER=$(gcloud projects describe "${PROJECT_ID}" --format="value(projectNumber)")
```

### 2\. Enable Required APIs

This command enables all the necessary services for the deployment pipeline.

```bash
gcloud services enable \
  run.googleapis.com \
  cloudbuild.googleapis.com \
  artifactregistry.googleapis.com \
  iam.googleapis.com \
  iamcredentials.googleapis.com \
  --project="${PROJECT_ID}"
```

### 3\. Create the Artifact Registry Repository

This is where your container images will be stored.

```bash
gcloud artifacts repositories create "${ARTIFACT_REPO}" \
  --project="${PROJECT_ID}" \
  --repository-format="docker" \
  --location="${REGION}" \
  --description="Repository for Cloud Run images"
```

### 4\. Create the Service Account

This is the identity that GitHub Actions will use inside your GCP project.

```bash
gcloud iam service-accounts create "${SERVICE_ACCOUNT_NAME}" \
  --project="${PROJECT_ID}" \
  --display-name="GitHub Actions Deploy SA"
```

### 5\. Configure Workload Identity Federation

This securely links your GitHub repository to the service account without needing to store secret keys.

**a. Create the Workload Identity Pool:**

```bash
gcloud iam workload-identity-pools create "github-pool" \
  --project="${PROJECT_ID}" \
  --location="global" \
  --display-name="GitHub Actions Pool"
```

**b. Create the Provider within the Pool:**

```bash
gcloud iam workload-identity-pools providers create-oidc "github-provider" \
  --project="${PROJECT_ID}" \
  --location="global" \
  --workload-identity-pool="github-pool" \
  --display-name="GitHub Actions Provider" \
  --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository" \
  --issuer-uri="https://token.actions.githubusercontent.com"
```

**c. Link Your GitHub Repo to the Service Account:**
This is the final step that allows actions from your specific repository to impersonate the service account.

```bash
WORKLOAD_IDENTITY_POOL_ID=$(gcloud iam workload-identity-pools describe "github-pool" --project="${PROJECT_ID}" --location="global" --format="value(name)")

gcloud iam service-accounts add-iam-policy-binding "${SERVICE_ACCOUNT_EMAIL}" \
  --project="${PROJECT_ID}" \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/${WORKLOAD_IDENTITY_POOL_ID}/attribute.repository/${GITHUB_REPO}"
```

### 6\. Grant IAM Permissions

The final step is to grant the necessary roles to the two service accounts involved in the process.

**a. Permissions for the GitHub Service Account (`github-deploy-sa`):**
This account needs permission to build, store, and deploy.

```bash
# To submit builds and store source code
gcloud projects add-iam-policy-binding "${PROJECT_ID}" --member="serviceAccount:${SERVICE_ACCOUNT_EMAIL}" --role="roles/cloudbuild.builds.editor"
gcloud projects add-iam-policy-binding "${PROJECT_ID}" --member="serviceAccount:${SERVICE_ACCOUNT_EMAIL}" --role="roles/storage.objectAdmin"

# To push the final container image to Artifact Registry
gcloud projects add-iam-policy-binding "${PROJECT_ID}" --member="serviceAccount:${SERVICE_ACCOUNT_EMAIL}" --role="roles/artifactregistry.writer"

# To deploy to Cloud Run and act as the Cloud Run service identity
gcloud projects add-iam-policy-binding "${PROJECT_ID}" --member="serviceAccount:${SERVICE_ACCOUNT_EMAIL}" --role="roles/run.admin"
gcloud projects add-iam-policy-binding "${PROJECT_ID}" --member="serviceAccount:${SERVICE_ACCOUNT_EMAIL}" --role="roles/iam.serviceAccountUser"

# To view logs in the GitHub Actions runner (optional but recommended)
gcloud projects add-iam-policy-binding "${PROJECT_ID}" --member="serviceAccount:${SERVICE_ACCOUNT_EMAIL}" --role="roles/logging.viewer"
```

**b. Permissions for the Cloud Build Service Account:**
The Cloud Build "worker" service also needs permission to use your new service account during the deployment.

```bash
# Allow the Cloud Build worker to act as your GitHub SA during deployment
gcloud iam service-accounts add-iam-policy-binding "${SERVICE_ACCOUNT_EMAIL}" \
  --project="${PROJECT_ID}" \
  --member="serviceAccount:${PROJECT_NUMBER}@cloudbuild.gserviceaccount.com" \
  --role="roles/iam.serviceAccountUser"
```

-----

## GitHub Actions Workflow File

Create the file `.github/workflows/cloud-run-deploy.yml` with the following contents.

```yaml
name: 'Build and Deploy to Cloud Run'

on:
  push:
    branches:
      - 'main'

permissions:
  id-token: write
  contents: read

env:
  PROJECT_ID: "[your project id]"
  # IMPORTANT: Find your Project Number in the GCP Console or with 'gcloud projects describe'
  PROJECT_NUMBER: "196200699792"
  SERVICE_ACCOUNT: "github-deploy-sa@[your project id].iam.gserviceaccount.com"
  REGION: "us-central1"
  SERVICE: "my-service"
  ARTIFACT_REPO: "cloudrun-repo"

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: 'Checkout'
        uses: 'actions/checkout@v4'

      - name: 'Authenticate to Google Cloud'
        uses: 'google-github-actions/auth@v2'
        with:
          # You may need to update the provider ID if you named it differently
          workload_identity_provider: 'projects/${{ env.PROJECT_NUMBER }}/locations/global/workloadIdentityPools/github-pool/providers/github-provider'
          service_account: '${{ env.SERVICE_ACCOUNT }}'

      - name: 'Set up Cloud SDK'
        uses: 'google-github-actions/setup-gcloud@v2'

      - name: 'Build and Push Docker Image'
        run: |-
          # This builds your Dockerfile and pushes the container to Artifact Registry
          gcloud builds submit --tag ${{ env.REGION }}-docker.pkg.dev/${{ env.PROJECT_ID }}/${{ env.ARTIFACT_REPO }}/${{ env.SERVICE }}

      - name: 'Deploy to Cloud Run'
        run: |-
          # This deploys the specific image you just built
          gcloud run deploy ${{ env.SERVICE }} \
            --image ${{ env.REGION }}-docker.pkg.dev/${{ env.PROJECT_ID }}/${{ env.ARTIFACT_REPO }}/${{ env.SERVICE }} \
            --region ${{ env.REGION }} \
            --platform managed \
            --allow-unauthenticated
```

Once these steps are complete, any push to the `main` branch will automatically build and deploy your application to Cloud Run.
