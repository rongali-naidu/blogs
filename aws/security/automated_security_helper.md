## What is Amazon Security Helper tool?

https://github.com/awslabs/automated-security-helper

## How to set up in Linux?

```
# Set up some variables
REPO_DIR="${HOME}"/Documents/repos/reference
REPO_NAME=automated-security-helper

# Create a folder to hold reference git repositories
mkdir -p ${REPO_DIR}

# Clone the repository into the reference area
git clone https://github.com/awslabs/automated-security-helper "${REPO_DIR}/${REPO_NAME}"

# Set the repo path in your shell for easier access
#
# Add this (and the variable settings above) to
# your ~/.bashrc, ~/.bash_profile, ~/.zshrc, or similar
# start-up scripts so that the ash tool is in your PATH
# after re-starting or starting a new shell.
#
export PATH="${PATH}:${REPO_DIR}/${REPO_NAME}"
```
## Run the ash tool

```
ash --version
ash --source-dir ~/final_code 
```

## How to add suppression rules in cloudformation?

```
 TestBucket:
    Type: 'AWS::S3::Bucket'
    Properties:
      BucketName: !Sub "test-${AWS::Region}-${AWS::AccountId}"
      VersioningConfiguration:
        Status: Enabled     
    Metadata:
      cdk_nag:
        rules_to_suppress:
          - id: "AwsSolutions-S1"
            reason: "Since the S3 Bucket is used for an example demonstration, disabled the access logging"
      checkov:
        skip:
          - id: "CKV_AWS_18"
            comment: "Since the S3 Bucket is used for an example demonstration, disabled the access logging"
```
