README.md
Trivy Overview
Trivy is an open-source vulnerability scanner developed by Aqua Security. It is used to detect vulnerabilities and misconfigurations in container images and other artifacts.
This setup integrates Trivy into an Azure DevOps CI/CD pipeline to automatically scan container images for security vulnerabilities during the build and deployment process. �
Trivy-Doc.docx
Pipeline Execution
The Trivy scanning process is implemented using a reusable Azure DevOps pipeline template.
File used:
YAML
trivy.steps.yml
The process includes the following stages:
Install Trivy
Run Trivy Scan
Publish Trivy Report Artifact
Publish Test Results
1. Install Trivy
This step installs Trivy on the Ubuntu build agent.
Actions Performed
Updates Ubuntu package lists
Installs wget
Downloads the specified Trivy Debian package
Installs Trivy
Verifies installation using trivy -v
Azure DevOps YAML
YAML
- bash: |
    sudo apt-get update
    sudo apt-get install -y wget

    wget https://github.com/aquasecurity/trivy/releases/download/v${{ parameters.trivy_Version }}/trivy_${{ parameters.trivy_Version }}_Linux-64bit.deb

    sudo dpkg -i trivy_${{ parameters.trivy_Version }}_Linux-64bit.deb

    trivy -v
  enabled: ${{ parameters.enabled }}
  displayName: Install Trivy
2. Run Trivy Scan
This step performs vulnerability scanning on the container image and generates XML reports.
Scan Details
Two separate scans are executed:
Severity
Purpose
CRITICAL, HIGH
High priority vulnerabilities
MEDIUM, LOW
Lower priority vulnerabilities
Features
Uses custom template trivy.tpl
Generates XML reports
Scans only vulnerabilities using --scanners vuln
Saves reports in artifact staging directory
Azure DevOps YAML
YAML
- task: CmdLine@2
  enabled: ${{ parameters.enabled }}
  displayName: Run Trivy Scan
  inputs:
    script: |

      mkdir -p $(Build.ArtifactStagingDirectory)/${{ parameters.artifactPath }}

      trivy image --severity CRITICAL,HIGH \
        --format template \
        --template "@Pipelines/build/trivy.tpl" \
        -o $(Build.ArtifactStagingDirectory)/${{ parameters.artifactPath }}/trivy-report-critical-high.xml \
        "${{ parameters.imageRepository }}:${{ parameters.container_tag }}" \
        --scanners vuln

      trivy image --severity MEDIUM,LOW \
        --format template \
        --template "@Pipelines/build/trivy.tpl" \
        -o $(Build.ArtifactStagingDirectory)/${{ parameters.artifactPath }}/trivy-report-medium-low.xml \
        "${{ parameters.imageRepository }}:${{ parameters.container_tag }}" \
        --scanners vuln
3. Publish Trivy Report Artifact
This stage publishes generated Trivy reports as Azure DevOps pipeline artifacts.
Actions Performed
Publishes all generated XML reports
Makes reports downloadable from pipeline execution results
Azure DevOps YAML
YAML
- task: PublishPipelineArtifact@1
  enabled: ${{ parameters.enabled }}
  inputs:
    targetPath: '$(Build.ArtifactStagingDirectory)/${{ parameters.artifactPath }}'
    artifact: ${{ parameters.artifactPath }}
    publishLocation: 'pipeline'
  displayName: Publish Trivy Report Artifact
Accessing Reports
Open Azure DevOps Pipeline Run
Open the pipeline job execution
Navigate to the published artifacts section
4. Publish Test Results
This step publishes Trivy XML reports as JUnit test results.
Features
Processes XML files as JUnit test results
Merges all test results into a single report
Displays vulnerabilities inside the Azure DevOps Tests tab
Each vulnerability appears as a failed test case
Azure DevOps YAML
YAML
- task: PublishTestResults@2
  enabled: ${{ parameters.enabled }}
  inputs:
    testResultsFormat: 'JUnit'
    testResultsFiles: '$(Build.ArtifactStagingDirectory)/${{ parameters.artifactPath }}/*.xml'
    mergeTestResults: true
    failTaskOnFailedTests: false
    testRunTitle: 'Trivy - Image Vulnerabilities'
  displayName: Publish Trivy Test Results
Pipeline Output
After pipeline execution, the following outputs are available:
1. Pipeline Artifacts
Generated XML reports containing:
Vulnerability details
Severity information
Separate reports for:
Critical/High
Medium/Low
2. Test Results
Azure DevOps test visualization showing:
Vulnerability statistics
Failed test entries for each vulnerability
Severity breakdown
Example Pipeline Flow
Plain text
Install Trivy
      ↓
Run Vulnerability Scan
      ↓
Generate XML Reports
      ↓
Publish Artifacts
      ↓
Publish Test Results
Reference Documentation
trivy.dev⁠�
cve.org⁠�
Summary
This Trivy integration enables automated vulnerability scanning in Azure DevOps pipelines by:
Installing Trivy dynamically
Scanning container images
Generating XML vulnerability reports
Publishing reports as pipeline artifacts
Displaying vulnerabilities in Azure DevOps test results
This helps teams identify and track container security issues during CI/CD execution.
