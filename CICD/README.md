# CodeBuildDocker.json
- Use this template if you are simply building a Docker image (x86 or ARM64) from a public repository (e.g. GitHub) and publishing it to a Elastic Container Registry repository.
- Ensure that you have a valid Dockerfile and buildspec.yml in the root directory of your repository.
# MultiArchCodePipeline.json
- Use this template if you want to build two types of Docker images - x86 and ARM64 - from one source code and publishing them to the same Elastic Container Registry repository.
- Before creating a stack with this template, you need to create a connection to a code repository at https://console.aws.amazon.com/codesuite/settings/connections as the connection ARN is required as a parameter in the template.
- Ensure that you have a valid Dockerfile, buildspec.yml and buildspec-manifest.yml in the root directory of your repository.

For the examples of Dockerfile, buildspec.yml and buildspec-manifest.yml, please refer to [this repository](https://github.com/tchangkiat/sample-express-api). The Dockerfile example is suitable only for NodeJS applications.
