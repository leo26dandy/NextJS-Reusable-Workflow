# Automated Deployment of Next.js App to Hostinger via GitHub Actions

This project uses GitHub Actions to automatically deploy the Next.js app to Hostinger when changes are pushed to the `main` (production) or `dev-stable` (staging) branches.

## Branches & Environments

- **main**: Deploys to production directory, uses the production PM2 process.
- **dev-stable**: Deploys to staging directory, uses the staging PM2 process.

## Setup Steps

1. **Store Secrets in GitHub Repo**
   - `HOSTINGER_HOST`: Your server’s IP or domain
   - `HOSTINGER_USER`: SSH username
   - `HOSTINGER_SSH_KEY`: Private SSH key (no passphrase)
   - `HOSTINGER_PORT`: SSH port (default 22)

2. **Workflow Overview**
   - On push to `main` or `dev-stable`, the workflow:
     1. Sets environment variables based on the branch.
     2. Connects to Hostinger via SSH.
     3. Navigates to the correct project directory.
     4. Cleans untracked and changed files.
     5. Pulls the latest code.
     6. Installs dependencies and builds the project.
     7. Restarts or starts the app with PM2.

3. **Example Workflow YAML**

```yaml
name: Deploy Next.js to Hostinger with PM2

on:
  push:
    branches:
      - main
      - dev-stable

env:
  PROD_DIR: /home/youruser/htdocs/yourproductionpath/
  STAGING_DIR: /home/youruser/htdocs/yourstagingpath/
  PROD_PM2: yourapp_prod
  STAGING_PM2: yourapp_staging

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Set deployment variables
        run: |
          if [[ "${GITHUB_REF##*/}" == "main" ]]; then
            echo "DIR=$PROD_DIR" >> $GITHUB_ENV
            echo "PM2_NAME=$PROD_PM2" >> $GITHUB_ENV
          else
            echo "DIR=$STAGING_DIR" >> $GITHUB_ENV
            echo "PM2_NAME=$STAGING_PM2" >> $GITHUB_ENV
          fi

      - name: Deploy to Hostinger
        uses: appleboy/ssh-action@v1.0.0
        with:
          host: ${{ secrets.HOSTINGER_HOST }}
          username: ${{ secrets.HOSTINGER_USER }}
          key: ${{ secrets.HOSTINGER_SSH_KEY }}
          port: ${{ secrets.HOSTINGER_PORT || 22 }}
          script: |
            cd ${{ env.DIR }}
            git config --global --add safe.directory ${{ env.DIR }}
            git reset --hard
            git clean -fd
            git fetch origin
            git checkout ${GITHUB_REF##*/}
            git pull origin ${GITHUB_REF##*/}
            export NVM_DIR="$HOME/.nvm"
            [ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh"
            nvm use 22
            npm install --force
            npm run build
            pm2 restart ${{ env.PM2_NAME }} || pm2 start npm --name ${{ env.PM2_NAME }} -- run start -- -p 3001
```

## Tips

- Make sure your deployment directories (`PROD_DIR`, `STAGING_DIR`) and PM2 names match your server setup.
- The SSH key stored in secrets should be added to your server’s `authorized_keys`.
- Adjust the Node.js version (`nvm use 22`) as needed.

---

_This setup ensures seamless, environment-specific deployment for both production and staging branches with a single workflow._
