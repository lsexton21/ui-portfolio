## Creating a Bitbucket Pipeline and Serverless file for deploying a PHP REST API as a Lambda function

The following two files is a perfect example of how I was able to deploy our signup PHP REST API repo as an AWS lambda function.


---
####  This is the bitbucket pipeline itself which uses a php image just to install composer to create the vendors directory for the repo, then uses a node.js image to run the serverless file to deploy the repo as an AWS lambda function

```
pipelines:
  custom:
    "Deploy to Development":
      - step:
          name: Composer install
          image: php:8.3-alpine
          caches:
            - composer
          script:
            - apt-get update && apt-get install -y unzip git
            - curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer
            - composer install --prefer-dist --optimize-autoloader --no-dev
          artifacts:
            - vendor/**
            - bootstrap/cache/*.php
      - step:
          name: Deploy Lambda
          deployment: dev
          image: node:16-alpine
          caches:
            - node
          script:
            - npm install -g serverless && npm install
            - serverless config credentials --provider aws --key $AWS_ACCESS_KEY_ID --secret $AWS_SECRET_ACCESS_KEY
            - serverless deploy --stage dev
    "Deploy to Production":
      - step:
          name: Composer install
          image: php:8.3-alpine
          caches:
            - composer
          script:
            - apt-get update && apt-get install -y unzip git
            - curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer
            - composer install --prefer-dist --optimize-autoloader --no-dev
          artifacts:
            - vendor/**
            - bootstrap/cache/*.php
      - step:
          name: Deploy Lambda
          deployment: production
          image: node:21-alpine
          caches:
            - node
          script:
            - apk add python3
            - npm install -g serverless && npm install
            - serverless config credentials --provider aws --key $AWS_ACCESS_KEY_ID --secret $AWS_SECRET_ACCESS_KEY
            - serverless deploy --stage prod
```


---


#### This is most of the serverless file which configures the lambda.  There were some added complexities in this one that were fun to sort out, such as cooperating with our AWS vpc, and adding an AWS gateway proxy so the lambda function could double as a REST API.
```
service: signup-php-lambda

provider:
  name: aws
  region: ${param:awsRegionName}
  runtime: provided.al2
  httpApi:
    cors: true
  stage: ${opt:stage,'dev'}
  deploymentMethod: direct
  vpc:
    securityGroupIds:
      - ${param:securityGroup}
    subnetIds:
      - ${param:subnet1}
      - ${param:subnet2}
      - ${param:subnet3}

plugins:
  - ./vendor/bref/bref

functions:
  index:
    handler: public/index.php
    timeout: 898
    memorySize: 512
    reservedConcurrency: 5
    runtime: php-81-fpm
    url: true
    events: 
      - httpApi:
            path: /{proxy+}
            method: '*'
    environment:
    # the rest of this file just handles the environment variables
```
