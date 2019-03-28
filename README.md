# AppSync Real-Time Reference Architecture

![Overview](/media/realtime-refarch.png)

## Getting Started

### Prerequisites

- [AWS Account](https://aws.amazon.com/mobile/details) with appropriate permissions to create the related resources
- [NodeJS](https://nodejs.org/en/download/) with [NPM](https://docs.npmjs.com/getting-started/installing-node)
- [AWS CLI](http://docs.aws.amazon.com/cli/latest/userguide/installing.html) with output configured as JSON `(pip install awscli --upgrade --user)`
- [AWS Amplify CLI](https://github.com/aws-amplify/amplify-cli) configured for a region where [AWS AppSync](https://docs.aws.amazon.com/general/latest/gr/rande.html) and all other services in use are available `(npm install -g @aws-amplify/cli)`
- [AWS SAM CLI](https://github.com/awslabs/aws-sam-cli) `(pip install --user aws-sam-cli)`
- [Create React App](https://github.com/facebook/create-react-app) `(npm install -g create-react-app)`
- [Install JQ](https://stedolan.github.io/jq/)
- If using Windows, you'll need the [Windows Subsystem for Linux (WSL)](https://docs.microsoft.com/en-us/windows/wsl/install-win10)
- Create an account on TMDb and generate an [API Key](https://developers.themoviedb.org/3/getting-started/introduction)

### Back End Setup

1. Clone this repository 

    ```bash
    git clone https://github.com/aws-samples/appsync-refarch-realtime.git
    cd aws-appsync-chat-starter-react
    ```

2. Install the required modules:

    ```bash
    npm install
    ```

3. Init the directory as an amplify Javascript project using the React framework:
   
    ```bash
    amplify init
    ```

4. Now it's time to provision your cloud resources based on the local setup and configured features, execute the following command accepting all default options:

   ```bash
   amplify push
   ```

   Wait for the provisioning to complete. Once done, a `src/aws-exports.js` file with the resources information is created.

5. Execute the following commands in a shell terminal to set up some environment variables as well as configure IAM authentication:

    ```bash
    export AWS_REGION=$(jq -r '.providers.awscloudformation.Region' amplify/#current-cloud-backend/amplify-meta.json)
    export GRAPHQL_API_ID=$(jq -r '.api[(.api | keys)[0]].output.GraphQLAPIIdOutput' ./amplify/#current-cloud-backend/amplify-meta.json)
    export GRAPHQL_API_NAME=$(aws appsync get-graphql-api --api-id $GRAPHQL_API_ID --region $AWS_REGION | jq -r '.graphqlApi.name')
    export GRAPHQL_ENDPOINT=$(sed -n 's/.*"aws_appsync_graphqlEndpoint": "\(.*\)".*/\1/p' src/aws-exports.js)
    export UNAUTH_ROLE=$(jq -r '.providers.awscloudformation.UnauthRoleName' amplify/#current-cloud-backend/amplify-meta.json)
    export ID_POOL_ID=$(sed -n 's/.*"aws_cognito_identity_pool_id": "\(.*\)".*/\1/p' src/aws-exports.js)
    export ID_POOL_NAME=$(jq -r '.auth[(.auth | keys)[0]].output.IdentityPoolName' ./amplify/#current-cloud-backend/amplify-meta.json)
    export DEPLOYMENT_BUCKET_NAME=$(jq -r '.providers.awscloudformation.DeploymentBucketName' ./amplify/#current-cloud-backend/amplify-meta.json)
    export STACK_NAME=$(jq -r '.providers.awscloudformation.StackName' ./amplify/#current-cloud-backend/amplify-meta.json)
    aws cognito-identity update-identity-pool --identity-pool-id $ID_POOL_ID --identity-pool-name $ID_POOL_NAME --allow-unauthenticated-identities  --region $AWS_REGION
    aws appsync update-graphql-api --api-id $GRAPHQL_API_ID --name $GRAPHQL_API_NAME --authentication-type AWS_IAM --region $AWS_REGION
    ```

6. Edit the file  `sam-app/get-movie/app.js` and add your TMDb API Key (refer to Prerequisites) on line 5:

    ```javascript
    let url = 'https://api.themoviedb.org/3/movie/popular?api_key=<YOUR API KEY HERE>&language=en-US&page=';
    ```

7. Install the Lambda dependencies and deploy the backend with SAM:
    ```bash
   cd ./sam-app/get-movie;npm install; cd ../..
   sam package --template-file ./sam-app/template.yaml --s3-bucket $DEPLOYMENT_BUCKET_NAME --output-template-file packaged.yaml --region $AWS_REGION
   export STACK_NAME_SAM="$STACK_NAME-lambda"
   sam deploy --template-file ./packaged.yaml --stack-name $STACK_NAME_SAM --capabilities CAPABILITY_IAM --parameter-overrides unauthRole=$UNAUTH_ROLE graphqlApi=$GRAPHQL_API_ID graphqlEndpoint=$GRAPHQL_ENDPOINT --region $AWS_REGION
   ```

8. Execute the following command to access and query your API directly from the AWS Console (or manually go to the [AWS AppSync Console](https://console.aws.amazon.com/appsync/home), access your API and open the `Queries` section)

    ```bash
    amplify console
    ? Please select the category or provider: api
    ? Please select from one of the below mentioned services: GraphQL
    ```

8.  Execute each one of these 5 mutations to create the data sctructure that will collect and store votes in the Reviews table:

    ```graphql
    mutation createVotes1 {
        createReviews(input:{
            id:1, type:"love", votes:0, topMovie:"N/A", topVotes:0
        }){
            id
            type
            votes
            topMovie
            topVotes
        }
    }

    mutation createVotes2 {
        createReviews(input:{
            id:2, type:"like", votes:0, topMovie:"N/A", topVotes:0
        }){
            id
            type
            votes
            topMovie
            topVotes
        }
    }

    mutation createVotes3 {
        createReviews(input:{
            id:3, type:"meh", votes:0, topMovie:"N/A", topVotes:0
        }){
            id
            type
            votes
            topMovie
            topVotes
        }
    }

    mutation createVotes4 {
        createReviews(input:{
            id:4, type:"unknown", votes:0, topMovie:"N/A", topVotes:0
        }){
            id
            type
            votes
            topMovie
            topVotes
        }
    }

    mutation createVotes5 {
        createReviews(input:{
            id:5, type:"hate", votes:0, topMovie:"N/A", topVotes:0
        }){
            id
            type
            votes
            topMovie
            topVotes
        }
    }
    ```

9.  Finally, execute the following command to install your project package dependencies and run the application locally:

        
        amplify serve
        
10.  Open different browsers and test realtime subscriptions. Alternativelly publish your application and use the public link:

        ```bash
        amplify add hosting
        amplify publish
        ```

### Clean Up


To clean up the project, you can simply delete the stack created by the SAM CLI:

```bash
aws cloudformation delete-stack --stack-name $STACK_NAME_SAM --region $AWS_REGION
```

and use:

```bash
amplify delete
```

to delete the resources created by the Amplify CLI.

## License Summary
This sample code is made available under a modified MIT license. See the LICENSE file.