### 4. Configuring GitLab CI/CD

This section outlines the steps to configure GitLab CI/CD for deploying a Nuxt 2 application to staging and production
environments. The `.gitlab-ci.yml` file and `ecosystem.config.js` facilitate automated deployment using Node.js and PM2.

#### 4.1 Define CI/CD Variables

Navigate to **Settings > CI/CD > Variables** in GitLab. Add these variables to securely store sensitive data:

- `STAGING_SSH_PRIVATE_KEY`: Private SSH key for the staging server.
- `PRODUCTION_SSH_PRIVATE_KEY`: Private SSH key for the production server.

Ensure these keys match those configured for your `DEPLOY_USER` on the respective servers.

#### 4.2 Overview of `.gitlab-ci.yml`

This file defines the pipeline with the following stages:

- **Test**: Runs tests to validate the application.
- **Build**: Compiles the application for deployment.
- **Deploy to Staging**: Deploys the build to staging.
- **Deploy to Production**: Deploys the build to production.

```yaml
image: node:18

stages:
  - test
  - build
  - deploy-production

variables:
  PRODUCTION_SERVER: staging-server-ip
  DEPLOY_USER: deployer
  PRODUCTION_APP_NAME: demo-deploy-nuxt2-app-production
  PRODUCTION_PORT: 3200
  PRODUCTION_APP_PATH: /var/www/production
  DEPLOY_FILES: ".nuxt static nuxt.config.js package.json yarn.lock ecosystem.config.js"

cache:
  paths:
    - node_modules/
    - .yarn-cache/

before_script:
  - yarn config set cache-folder .yarn-cache
  - yarn install --frozen-lockfile

test:
  stage: test
  script:
    - yarn test

build:
  stage: build
  script:
    - yarn build
  artifacts:
    paths:
      - .nuxt/
      - static/
      - nuxt.config.js
      - package.json
      - yarn.lock
      - ecosystem.config.js
    expire_in: 1 hour

deploy-production:
  stage: deploy-production
  environment:
    name: production
    url: https://yourdomain.com
  script:
    - apt-get update -qy
    - apt-get install -y openssh-client
    - mkdir -p ~/.ssh
    - echo "$PRODUCTION_SSH_PRIVATE_KEY" | tr -d '\r' > ~/.ssh/id_rsa
    - chmod 600 ~/.ssh/id_rsa
    - ssh-keyscan -H $PRODUCTION_SERVER >> ~/.ssh/known_hosts

    # Create new release directory
    - export RELEASE_DIR=$(date +%Y%m%d%H%M%S)
    - |
      ssh $DEPLOY_USER@$PRODUCTION_SERVER "
      mkdir -p $PRODUCTION_APP_PATH/releases/$RELEASE_DIR;
      "

    # Copy built artifacts
    - scp -r $DEPLOY_FILES $DEPLOY_USER@$PRODUCTION_SERVER:$PRODUCTION_APP_PATH/releases/$RELEASE_DIR/

    # Update symlinks and restart application
    - |
      ssh $DEPLOY_USER@$PRODUCTION_SERVER "
      cd $PRODUCTION_APP_PATH;
      if [ -L current ]; then mv current previous; fi;
      ln -s releases/$RELEASE_DIR current;
      cd current;
      yarn install --frozen-lockfile --production;
      export APP_NAME=$PRODUCTION_APP_NAME;
      export NODE_ENV=production;
      export PORT=$PRODUCTION_PORT;
      export APP_PATH=$PRODUCTION_APP_PATH/current;
      pm2 startOrGracefulReload ecosystem.config.js --update-env;
      pm2 save;

      # Cleanup old releases (keep last 5)
      cd releases;
      ls -1dt */ | tail -n +6 | xargs rm -rf;
      "
  only:
    - main
```

##### Key Highlights

1. **Caching**:
   ```yaml
   cache:
     key: ${CI_COMMIT_REF_SLUG}
     paths:
       - node_modules/
       - .yarn-cache/
   ```
   Speeds up pipelines by caching dependencies.

2. **Artifacts**:
   Artifacts from the build stage are stored temporarily for deployment.
   ```yaml
   artifacts:
     paths:
       - .nuxt/
       - static/
       - nuxt.config.js
       - package.json
       - yarn.lock
       - ecosystem.config.js
     expire_in: 1 hour
   ```

3. **Deployment Scripts**:
   Deployment involves creating a release directory, transferring files, updating symlinks, and restarting the app using
   PM2.

#### 4.3 Server Configurations

Ensure these server configurations:

- `DEPLOY_USER` has access to `/var/www/staging` and `/var/www/production`.
- PM2 is installed and properly set up.

#### 4.4 `ecosystem.config.js` Customization

The `ecosystem.config.js` file specifies PM2 settings:

```javascript
module.exports = {
    apps: [
        {
            name: `demo-deploy-nuxt2-app-${process.env.NODE_ENV}`,
            script: './node_modules/nuxt/bin/nuxt.js',
            args: 'start',
            cwd: process.env.APP_PATH || '/var/www/staging/current',
            instances: 'max',
            exec_mode: 'cluster',
            env: {
                NODE_ENV: process.env.NODE_ENV,
                PORT: process.env.PORT || 3100,
            },
            max_memory_restart: '1G',
            error_file: `/var/log/pm2/${process.env.APP_NAME || 'staging'}-error.log`,
            out_file: `/var/log/pm2/${process.env.APP_NAME || 'staging'}-out.log`
        }
    ]
}
```

Adjust paths and variables as necessary.

#### 4.5 Set Up the Pipeline

Commit the `.gitlab-ci.yml` and `ecosystem.config.js` files to your repository and push changes to trigger the pipeline.

##### Workflow

- **Staging Deployment**: Automatically deploys from the `staging` branch.
- **Production Deployment**: Automatically deploys from the `main` branch.

#### 4.6 Testing and Troubleshooting

1. Push code changes to `staging` and verify deployment at `https://staging.yourdomain.com`.
2. Merge changes to `main` to test production deployment at `https://yourdomain.com`.
3. For issues:
    - Check GitLab pipeline logs for errors.
    - Inspect PM2 logs (`/var/log/pm2`) on the server.
    - Verify SSH keys and configurations.

This configuration ensures a streamlined deployment process for your Nuxt 2 application.

