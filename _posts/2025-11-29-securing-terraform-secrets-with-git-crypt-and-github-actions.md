---
layout: post
title: Securing Terraform secrets with git-crypt and GitHub Actions
date: 2025-11-29 00:00:00 +0000
summary: "How to bypass GPG agent errors in GitHub Actions by using git-crypt symmetric keys."
minutes: 5
---

If you are managing Infrastructure as Code (IaC) with Terraform, you eventually hit the wall: Secrets.

You need to commit .tfvars files containing database passwords and API keys, but you can't put them in plaintext Git. Tools like git-crypt are perfect for this—they transparently encrypt secrets on commit and decrypt them on checkout.

But if you've ever tried to run git-crypt unlock inside a GitHub Action, you've likely seen this nightmare:

```bash
gpg: cannot open '/dev/tty': No such device or address
error: inappropriate ioctl for device
```

I spent hours fighting GPG agents, loopback pinentries, and allow-preset-passphrase configurations only to realize I was doing it wrong.

Here is the robust, crash-proof way to combine git-crypt, Terraform, and GitHub Actions without fighting GPG agents.

The Strategy: Symmetric Keys
The common mistake is trying to import your private GPG key into the CI runner. GPG is designed for humans to type passwords interactively. CI runners are robots.

Instead of GPG keys, we will use git-crypt's Symmetric Key feature. This allows us to export a binary "master key" file that works without a passphrase, GPG agent, or TTY.

## Step 1: Local Setup

First, ensure you have git-crypt installed locally (e.g., brew install git-crypt). There's a good guide here: [https://buddy.works/guides/git-crypt](https://buddy.works/guides/git-crypt) but the short version is:

- Windows

1. install gpg4win https://gpg4win.org/download.html
2. install git-crypt and add to your PATH - https://github.com/AGWA/git-crypt/releases

- MacOS

    ```bash
    brew install git-crypt
    brew install gpg
    ```

- Linux

    ```bash
    apt-get install -y git-crypt
    sudo apt-get install gnupg
    ```

Then initalize your repo with git-crypt:

```bash
# Initialize git-crypt
git-crypt init

# Configure .gitattributes to encrypt sensitive files
echo "secret.tfvars filter=git-crypt diff=git-crypt" >> .gitattributes
git add .gitattributes
git commit -m "Configure git-crypt"
```

## Step 2: Export the Symmetric Key

This is the magic step. We need to export the repository's unlock key and encode it so we can store it as a GitHub Secret.

On your local machine (where the repo is already unlocked):

1. Export the key to a binary file.

```bash
git-crypt export-key ./git-crypt-key
```

2. Crucial Check: Ensure the file is exactly 64 bytes. If it's 37 bytes or empty, something went wrong.

```bash
ls -l ./git-crypt-key
```

3. Convert it to a Base64 string (single line, no newlines).

- macOS:

    ```bash
    cat ./git-crypt-key | base64 | tr -d '\n' | pbcopy
    ```

- Linux:

    ```bash
    base64 -w 0 ./git-crypt-key
    ```

- Windows:

    ```ps1
    [Convert]::ToBase64String([IO.File]::ReadAllBytes("git-crypt-key")) | Set-Clipboard
    ```

## Step 3: Configure GitHub Secrets

Go to your Repository Settings > Secrets and variables > Actions.

Create a new Repository Secret.

Name: GIT_CRYPT_KEY.

Value: Paste the Base64 string you generated in Step 2.

## Step 4: The Wrapper Script (tf.sh)

Running Terraform in CI produces a lot of ANSI color codes `([32m, [1m)` which look great in logs but break Markdown summaries.

Create a wrapper script tf.sh to handle the commands and strip colors when necessary.

```bash
#!/bin/bash
set -euo pipefail

component_path=${1?Please pass component path}
action=${2?Please pass action}
env=${3?Please pass environment}

# Common flags to ensure clean output
common_args="-no-color -lock=false"

# Initialize
terraform init -reconfigure -no-color

case "$action" in
  plan)
    # Silence the binary generation, show the text output
    terraform plan $common_args -out=tfplan >/dev/null
    terraform show -no-color tfplan
    rm tfplan
    ;;
  apply)
    terraform apply $common_args -auto-approve
    ;;
esac
```

__TIP__: Don't forget to make the script executable: `chmod +x tf.sh`

## Step 5: The GitHub Workflow

We will implement a "Plan/Apply" workflow using GitHub Environments.

- Job 1 (Plan): Runs on every trigger. Shows you what will happen.

- Job 2 (Deploy): Only runs if you select "Apply".

This uses git-crypt to unlock the repo using the Base64 key we stored.

{% raw %}

```yaml
name: Infrastructure Deploy
run-name: "${{ inputs.component_path }}: ${{ inputs.action }} on ${{ inputs.environment }}"

on:
  workflow_dispatch:
    inputs:
      component_path:
        description: 'Path to TF component'
        required: true
        type: string
      action:
        description: 'Action'
        required: true
        type: choice
        options: 
          - plan
          - apply
      environment:
        description: 'Target Environment'
        required: true
        type: choice
        options: [dev, prod]

jobs:
  plan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: sudo apt-get install -y git-crypt

      # THE FIX: Unlock using the symmetric key
      - name: Unlock Secrets
        env:
          GIT_CRYPT_KEY_BASE64: ${{ secrets.GIT_CRYPT_KEY }}
        run: |
          echo "$GIT_CRYPT_KEY_BASE64" | base64 -d > ./git-crypt-key
          git-crypt unlock ./git-crypt-key
          rm ./git-crypt-key

      - name: Run Plan
        run: |
          ./tf.sh ${{ inputs.component_path }} plan ${{ inputs.environment }} > plan_output.txt
          
          # Write to GitHub Step Summary
          echo "## Terraform Plan" >> $GITHUB_STEP_SUMMARY
          echo '```terraform' >> $GITHUB_STEP_SUMMARY
          cat plan_output.txt >> $GITHUB_STEP_SUMMARY
          echo '```' >> $GITHUB_STEP_SUMMARY

  deploy:
    needs: plan
    if: ${{ inputs.action == 'apply' }}
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }} # Triggers manual approval if configured
    steps:
      - uses: actions/checkout@v4
      - run: sudo apt-get install -y git-crypt
      
      # Unlock again (Fresh VM)
      - name: Unlock Secrets
        env:
          GIT_CRYPT_KEY_BASE64: ${{ secrets.GIT_CRYPT_KEY }}
        run: |
          echo "$GIT_CRYPT_KEY_BASE64" | base64 -d > ./git-crypt-key
          git-crypt unlock ./git-crypt-key
          rm ./git-crypt-key

      - name: Run Apply
        run: ./tf.sh ${{ inputs.component_path }} apply ${{ inputs.environment }}
```
{% endraw %}

## Bonus: Manual Approvals without GitHub Enterprise

If you are on GitHub Free (Private Repo), you don't get the "Review Deployment" button.

However, the workflow above includes a logical gate.

1. Run the workflow with Action: Plan. Check the output summary.

2. If it looks good, run it again with Action: Apply.

This effectively creates a manual approval step without paying for GitHub Enterprise.

## Conclusion

By switching from GPG keys to Symmetric Keys, we bypassed the fragile agent/TTY layer entirely. git-crypt behaves predictably, Terraform runs securely, and you get a clean, automated infrastructure pipeline.

You could of course improve the `tf.sh` script for import, destroy and other terraform actions, but I'm not going to give away all my secrets! 😊
