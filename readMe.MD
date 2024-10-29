# Deploy to Amazon LightSail with GitHub Actions

This workflow will automate building, pushing a new container image to Amazon Elastic Container Registry (ECR), and deploying the new image to an Amazon Lightsail instance. It also handles the cleanup of previous images from both Lightsail and ECR.

## Prerequisites

-	**AWS Account**: Ensure you have access to AWS services like Lightsail and ECR.
-	**Amazon ECR Repository**: Create an ECR repository to store your Docker images.
-	**Lightsail Instance**: Have a running Lightsail instance with Docker installed.
-	**AWS CLI Installed**: Make sure the AWS CLI is installed and configured in your environment.
-	**GitHub Secrets**: Set the following secrets in your GitHub repository:
    -	**AWS_ACC_ID**: Your AWS Account ID.
    -	**AWS_REGION**: The AWS region (e.g., us-west-1).
    -	**ECR_REPOSITORY**: Your ECR repository name.
    -	**CONTAINER_NAME**: Name of the container running on your Lightsail instance.
    -	**AWS_ACCESS_KEY_ID**: AWS access key.
    -	**AWS_SECRET_ACCESS_KEY**: AWS secret key.
    -	**LIGHTSAIL_SSH_KEY**: Base64 encoded SSH private key for your Lightsail instance.
    -	**LIGHTSAIL_INSTANCE_IP**: IP address of your Lightsail instance.
-	**Dockerfile**: Ensure that your project has a Dockerfile to build the Docker image.

## Workflow Explanation

### 1. Checkout Code

The workflow starts by fetching the latest version of your code from the repository.

```yaml
- name: Checkout
  uses: actions/checkout@v4
```

### 2. Install AWS CLI & Lightsailctl

**AWS CLI** is updated and `lightsailctl` is installed to allow communication with AWS Lightsail.

```yaml
- name: Upgrade AWS CLI version and setup lightsailctl
  run: |
    aws --version
    curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
    unzip awscliv2.zip
    sudo ./aws/install --bin-dir /usr/local/bin --install-dir /usr/local/aws-cli --update
    sudo curl "https://s3.us-west-2.amazonaws.com/lightsailctl/latest/linux-amd64/lightsailctl" -o "/usr/local/bin/lightsailctl"
    sudo chmod +x /usr/local/bin/lightsailctl
```

### 3. Configure AWS Credentials

The workflow configures AWS credentials using GitHub Secrets to authenticate with AWS services.

```yaml
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v1
  with:
    aws-access-key-id: ${{ env.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ env.AWS_SECRET_ACCESS_KEY }}
    aws-region: ${{ env.AWS_REGION }}
```

### 4. Login to ECR

Authenticates Docker to push and pull images from your Amazon ECR repository.

```yaml
- name: Login to Amazon ECR
  uses: aws-actions/amazon-ecr-login@v1
```

### 5. Build and Push Docker Image

This step builds a Docker image from the checked-out code and pushes it to ECR.

```yaml
- name: Build, tag, and push image to Amazon ECR
  run: |
    `docker build` -t $ECR_REGISTRY/$ECR_REPOSITORY:latest .
    docker push $ECR_REGISTRY/$ECR_REPOSITORY:latest
```

### 6. Delete Previous ECR Image

To avoid ECR image build-up, the workflow deletes the oldest image in your ECR repository.

```yaml
- name: Delete Previous Image from ECR
  run: |
    IMAGE_DIGEST=$(`aws ecr describe-images` \
      --repository-name $ECR_REPOSITORY \
      --registry-id $AWS_ACCOUNT_ID \
      --query 'sort_by(imageDetails,&imagePushedAt)[0].imageDigest' \
      --output text)
    if [ "$IMAGE_DIGEST" != "None" ]; then
      aws ecr batch-delete-image \
        --repository-name $ECR_REPOSITORY \
        --registry-id $AWS_ACCOUNT_ID \
        --image-ids imageDigest=$IMAGE_DIGEST
    fi
```

### 7. Setup SSH Key

Sets up an SSH key to allow remote access to the Lightsail instance.

```yaml
- name: Set up SSH key
  run: |
    mkdir -p ~/.ssh
    echo "${{ env.LIGHTSAIL_SSH_KEY }}" | base64 -d > ~/.ssh/id_rsa.pem
    chmod 600 ~/.ssh/id_rsa.pem
    ssh-keyscan -t rsa "${{ env.LIGHTSAIL_INSTANCE_IP }}" >> ~/.ssh/known_hosts
```

### 8. Stop Old Container and Run New One

Connects to the Lightsail instance via SSH, stops and removes the existing Docker container and image, then pulls and runs the new image.

```yaml
- name: Connect to Lightsail instance, stop and remove container/images, pull latest image, and run new container
  run: |
    ssh -o StrictHostKeyChecking=no -i ~/.ssh/id_rsa.pem ubuntu@"${{ env.LIGHTSAIL_INSTANCE_IP }}" << 'EOF'
    CONTAINER_NAME=${{env.CONTAINER_NAME}}
    if [ "$(docker ps -q -f name=$CONTAINER_NAME)" ]; then
      docker stop $CONTAINER_NAME
      docker rm $CONTAINER_NAME
    fi
    existing_image=$(docker images -q ${{ env.AWS_ACC_ID }}.dkr.ecr.${{ env.AWS_REGION }}.amazonaws.com/${{env.ECR_REPOSITORY}})
    if [ -n "$existing_image" ]; then
      docker rmi -f $existing_image
    fi
    docker pull ${{ env.AWS_ACC_ID }}.dkr.ecr.${{ env.AWS_REGION }}.amazonaws.com/${{env.ECR_REPOSITORY}}:latest
    docker run -d -p 5000:5000 --name $CONTAINER_NAME ${{ env.AWS_ACC_ID }}.dkr.ecr.${{ env.AWS_REGION }}.amazonaws.com/${{env.ECR_REPOSITORY}}:latest
    aws s3 cp s3://cartle-backend/.env /home/ubuntu/.env
    docker cp /home/ubuntu/.env "$CONTAINER_NAME:`/app/.env`"
    docker restart $CONTAINER_NAME
    EOF
```

## Conclusion

This GitHub Actions workflow allows for an automated CI/CD pipeline, pushing Docker images to ECR and deploying them to an Amazon Lightsail instance. It ensures that both Lightsail and ECR stay clean by removing old images.