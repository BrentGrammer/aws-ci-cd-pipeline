# AWS CI/CD Pipeline

- An exercise in creating an automated CI/CD pipeline for a Containerized (Docker) application using AWS Tooling

## Architecture of Pipeline

- GitHub Repo where the app is stored
- CodeBuild to produce deploy artifacts (container) and push to ECR
-

## Pipeline Setup

### Create an ECR Repository to Store Container Images

- AWS Console > ECR > Private > Repositories > Create Repository
- Enter a name (must be unique within the repository), i.e. `appnamepipeline` and click Create

### Create CodeBuild Project to Build a Docker Image from files in your App Repo

- [Video Demo](https://learn.cantrill.io/courses/1101194/lectures/40856334)
  Build configuration is in a file `buildspec.yml`
- AWS Console > CodeBuild > Build projects > Create Project
- Name the project: `myappname-build`
- Manage Credentials if there are no creds for the source
  - For GitHub, Installing the GitHub App (AWS Connector for GitHub) is generally the recommended option
- Enter a branch name i.e. `main` for the Source version
  - **NOTE:** You can leave this blank if you don't want the build to start automatically on push (instead you would go into the project and click "Start Build")
- Environment section: Select an image to use (can use a custom image from your ECR)
- Specify a service role that the code build project will use to interact with Amazon Services (can optionally have it create one for you)
  - Environment > Additional Configuration > environment variables
  - These variables can be utilized in the `buildspec.yml` configuration file
    ```
    AWS_DEFAULT_REGION = your region (i.e. us-east-1)
    AWS_ACCOUNT_ID = copy and paste an account id that has admin priviledges
    IMAGE_TAG = latest
    IMAGE_REPO_NAME = name of the repo where built images go (use the ECR repo you created, i.e. `catpipeline` shown here:)
    ```
    <img src="img/reponame.png" height="25%" width="25%" />
    <br>
    <br>
- Select buildspec file and point to your `buildspec.yml` file (CodeBuild by default will look for a file by this name in the root directory of your project repository)
  <br>
  <img src="img/buildspec.png" height="50%" width="50%" />
  <br>
  <br>
- **Artifacts** are output from CodeBuild which you can use as input to CodeDeploy to deploy your code to the infrastructure
- **Logs** can be specified and optionally use S3. Enter a group name and make the stream name the name of your pipeline (codebuild project name)
  <br>
  <img src="img/logs.png" height="50%" width="50%" />
  <br>
  <br>

### Update Permissions for CodeBuild Role

- Give the role used by CodeBuild to access services needed
- AWS Console > IAM > Roles > look for the role created in the build codebuild project process
- Click on the role > Add permissions > create inline policy
  - Add permissions to interact with ECR:
    ```json
    {
      "Statement": [
        {
          "Action": [
            "ecr:BatchCheckLayerAvailability",
            "ecr:CompleteLayerUpload",
            "ecr:GetAuthorizationToken",
            "ecr:InitiateLayerUpload",
            "ecr:PutImage",
            "ecr:UploadLayerPart"
          ],
          "Resource": "*",
          "Effect": "Allow"
        }
      ],
      "Version": "2012-10-17"
    }
    ```
  - Give the policy a name like `Codebuild-ECR` and create the policy

### Buildspec.YML - What CodeBuild does to files in your project repo

- Create a buildspec.yml file and add it to the root folder of your project
- After pushing to your repo main branch that should kick off a build (or you can go into the project and click "Start Build")
- This will build and create the image and store it in the ECR registry repo:
  <br>
  <img src="img/imagecreated.png" height="50%" width="50%" />
  <br>
  <br>

### Testing out an image from your ECR

- Start an EC2 instance with Docker installed and ssh into it
- login to ECR: `aws ecr get-login-password --region <AWS_REGION> | docker login --username AWS --password-stdin ACCOUNT_ID_REPLACEME.dkr.ecr.<AWS_REGION>.amazonaws.com`
- get the URI of the image from your ECR repo:
  <br>
  <img src="img/imageuri.png" height="50%" width="50%" />
  <br>
  <br>
- on the Instance run `docker pull <imageuri_from_your_ecr>`
- `docker images` to get the id of the image you downloaded
- Then run it with `docker run -p 80:80 <image_id>` (adjust ports exposed if needed)
- Go to public IP to see it served

### CodePipeline
- AWS Console > CodePipeline > Create Pipeline
- Select "Custom Pipeline" then "Next"
- Name your pipeline, i.e. `appnamepipeline`
- Allow AWS to create a new service role for the pipeline to interact with AWS resources
- **Advanced Settings** - This is where you can specify where Artifacts are output to.
  - These can be used as input to any stage in a code pipeline and how you tie together stages
  - By default, a S3 bucket is created in the account to store artifacts, but you could also choose a custom location
- Click next and select a Source (i.e. GitHub via GitHub app or whatever source you set up)
- Add a Build Provider (i.e. CodeBuild if you're using that)
  <br>
  <img src="img/provider.png" height="50%" width="50%" />
  <br>
  <br>
- Creating the pipeline will run an initial run
- You can look for the artifact bucket (default) with a name like "codepipeline-region...":
  <br>
  <img src="img/artifactbucket.png" height="50%" width="50%" />
  <br>
  <br>

