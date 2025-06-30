# sync_azure_aws

#Gitlab CI

aws_cd:sync_images:
  stage: test
  image:
    name: gcr.io/go-containerregistry/crane:debug
    entrypoint: [""]
  extends: [.aws_variables]
  before_script:
    - crane auth login ${AZURE} -u $AZURE_USER -p $AZURE_PASSWORD
    - wget http://amazon-ecr-credential-helper-releases.s3.us-east-2.amazonaws.com/0.9.0/linux-amd64/docker-credential-ecr-login
    - chmod +x docker-credential-ecr-login && mv -v docker-credential-ecr-login /usr/bin/
    - sed -i 's/^{/{\"credHelpers\":{\"362485432214.dkr.ecr.eu-central-1.amazonaws.com\":\"ecr-login\"},/' ~/.docker/config.json
    - env
  script:
    - >
      chmod +x sync.sh
      ./sync.sh

####################################

#!/bin/bash
set -euo pipefail

# Configurable variables
JSON_FILE="list_images.json"
AZURE="aoscloud.azurecr.io"
AWS_ECR="111111111.dkr.ecr.eu-central-2.amazonaws.com"

# Loop over each image in JSON
jq -c '.[]' "$JSON_FILE" | while read -r item; do
  # Extract raw fields and do substitution
  NAME=$(echo "$item" | jq -r '.name')
  SRC=$(echo "$item" | jq -r '.source' | sed "s|\${AOSCLOUD_ACR}|$AOSCLOUD_ACR|g" | sed "s|<TAG>|latest|g")
  DST=$(echo "$item" | jq -r '.destination' | sed "s|\${AWS_ECR}|$AWS_ECR|g" | sed "s|<TAG>|latest|g")

  echo "Syncing $NAME..."
  echo "  Source:      $SRC"
  echo "  Destination: $DST"

  crane copy "$SRC" "$DST" -a

done


