# learn-codebuild-aws-cli-update-env-var
how to update environment variable in codebuild

List projects in codebuild
```bash
aws codebuild list-projects | grep $project_keyword
```
List all in a project in codebuild
```bash
aws codebuild batch-get-projects --names $your-project-name
```
List specifics in a project in codebuild
```bash
aws codebuild batch-get-projects --names deploy-project-c8av9 --query 'projects[0].environment.environmentVariables'
or
aws codebuild batch-get-projects --names deploy-project-c8av9 --query 'projects[0].environment.environmentVariables[?name==`client_secret` || name==`project_name`]'
```
Here is a complete script that lists the environment variables client_secret and project_name for each project matching deploy-project:
```bash
#!/bin/bash

## List all projects and filter for projects starting with 'deploy-project'
project_names=$(aws codebuild list-projects --query 'projects' --output text | tr '\t' '\n' | grep 'deploy-project')

# Iterate over each project name and query the environment variables
for project_name in $project_names; do
    echo "Project: $project_name"
    aws codebuild batch-get-projects --names "$project_name" --query 'projects[0].environment.environmentVariables[?name==`client_secret` || name==`project_name`]' --output json
    echo
done
```
## Script to Update client_secret Environment Variable
```bash
#!/bin/bash

# List all projects and filter for projects starting with 'deploy-project'
project_names=$(aws codebuild list-projects --query 'projects' --output text | tr '\t' '\n' | grep 'deploy-project')

# New value for the client secret key
new_client_secret_key="new-client-secret-key"

# Iterate over each project name
for project_name in $project_names; do
    echo "Updating project: $project_name"
    
    # Get the current project configuration
    project=$(aws codebuild batch-get-projects --names "$project_name" --query 'projects[0]')

    # Extract current environment variables
    env_variables=$(echo "$project" | jq -c '.environment.environmentVariables')

    # Update or add the client_secret environment variable
    updated_env_variables=$(echo "$env_variables" | jq --arg name "client_secret" --arg value "$new_client_secret_key" 'map(if .name == $name then .value = $value else . end)')

    # If client_secret does not exist, add it
    if ! echo "$updated_env_variables" | grep -q '"name": "client_secret"'; then
        updated_env_variables=$(echo "$updated_env_variables" | jq --arg name "client_secret" --arg value "$new_client_secret_key" '. + [{"name": $name, "value": $value, "type": "PLAINTEXT"}]')
    fi

    # Update the project with the new environment variable
    aws codebuild update-project \
        --name "$project_name" \
        --environment "type=$(echo "$project" | jq -r '.environment.type'),image=$(echo "$project" | jq -r '.environment.image'),computeType=$(echo "$project" | jq -r '.environment.computeType'),environmentVariables=$updated_env_variables"
    
    echo "Updated client_secret for project $project_name"
    echo
done
```
## Insert new env variable to codebuild
```bash
#!/bin/bash

# Define the input file and a temporary output file
input_file="env.json"
temp_file="temp_file.json"

# Use jq to insert the new variable before s3_bucket
jq '(.environmentVariables | map(if .name == "cluster_name" then [{name: "cluster_version", value: "1.24", type: "PLAINTEXT"}, .] else [.] end) | flatten) as $newVars | .environmentVariables = $newVars' "$input_file"  > "$temp_file"

# Replace the original file with the updated one
mv "$temp_file" "$input_file"

echo "Inserted cluster_version before cluster_name in $input_file"
```
