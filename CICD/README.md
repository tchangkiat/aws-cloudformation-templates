# CodeBuildDocker.json
- Use this template if you are simply building a Docker image (x86 or ARM64) from a public repository (e.g. GitHub) and publishing it to a Elastic Container Registry repository.
# MultiArchCodePipeline.json
- Use this template if you want to build two types of Docker images - x86 and ARM64 - from one source code and publishing them to the same Elastic Container Registry repository.
- Before creating a stack with this template, you need to create a connection to a code repository at https://console.aws.amazon.com/codesuite/settings/connections as the connection ARN is required as a parameter in the template.
