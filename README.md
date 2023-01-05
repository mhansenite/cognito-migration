# migrate-cognito-user-pool-lambda

## Usage

Follow these steps to use the migration Lambda function:

1. Create a new user pool client in the old user pool
   This client must have the OAuth flow `ALLOW_ADMIN_USER_PASSWORD_AUTH` enabled.
2. Configure all clients in the new user pool that are allowed to trigger user migration
   These clients must use the OAuth flow `USER_PASSWORD_AUTH`.
3. Build the lambda source code

   ```bash
   npm install && npm run build
   ```
4. Create in Lambda function in the AWS console in the same account as the new user pool

   * Configure the `OLD_USER_POOL_REGION`, `OLD_USER_POOL_ID`, and `OLD_CLIENT_ID` environment variables
   * Grant the required permissions for accessing the user pool

     If the old user pool is in the same AWS account: `Allow` the actions `cognito-idp:AdminGetUser` and `cognito-idp:AdminInitiateAuth` in the execution role of the lambda function

     If the old user pool is in a different AWS account:

     1. Create a role in the account that owns the user pool that `Allow`s the `cognito-idp:AdminGetUser` and `cognito-idp:AdminInitiateAuth` actions and that trusts the execution role of the lambda function
     2. `Allow` the action `sts:AssumeRole` for the ARN of the created role in the execution role of the lambda function
     3. Configure the `OLD_ROLE_ARN` and `OLD_EXTERNAL_ID` environment variables for the lambda function
        * To make this more clear you need to have a policy and a role for both new and old accounts.  the old needs to trust the role of the new and the new still needs access to update its own pools (See the old-trust-policy.json and the new-trust-policy.json)
        * Also look at the new-role-policy.json and the old-role-policy.json to see what permissions are needed for both accounts
5. Configure the trigger _User Migration_ for the new User Pool to call the migration lambda function

## Using AWS CLI

If you wish to use [AWS CLI](https://docs.aws.amazon.com/cli/latest/)
This reduces the need to navigate around AWS Console which is always in flux and not the easiest to figure out.

Maintain a txt list of the following variables as you work your way through this

* `NEW_ACCOUNT_ID` - account ID of your destination aws account usually a 12 digits (222222222222)
* `OLD_ACCOUNT_ID` - account ID of your source aws account usually a 12 digits (111111111111)
* `OLD_USER_POOL_ID` - the pool id you are migrating *from* (us-east-2_xyzABC)
* `OLD_USER_POOL_ARN` - the pool Arn you are migrating *from* (arn:aws:cognito-idp:us-east-2:111111111111:userpool/us-east-2_xyzABC)
* `NEW_USER_POOL_ARN` - the pool Arn you are migrating *from* (arn:aws:cognito-idp:us-east-2:222222222222:userpool/us-east-2_xyzABC)
* `OLD_USER_POOL_REGION` - the region that pool is located in (us-east-1 or us-east-2 etc...)
* `NEW_USER_POOL_ID` - the pool you are migrating *to* (us-east-2_xyzDEF)
* `ROLE_ARN` (created in step 1)
* `NEW_ROLE_ARN` (created in step 1)
* `OLD_ROLE_ARN` (created in step 1)
* `POLICY_ARN` (created in step 2)
* `NEW_POLICY_ARN` (created in step 2)
* `OLD_POLICY_ARN` (created in step 2)
* `OLD_CLIENT_ID` (created in step 4)
* `LAMBDA_ARN` (created in step 5)

---



1. Create Role

   * Update the role name to match your DevOps procedures
   * Note the Arn returned from this as it will be your `ROLE_ARN` if used `NEW_ROLE_ARN`, `OLD_ROLE_ARN`
   * For different accounts the OLD account needs a different trust policy update the old-trust-policy and use those respectively

SameAccount

```bash
   aws iam create-role --role-name cognito-migration-lambda-xxxx \
                       --assume-role-policy-document file://trust-policy.json
```

DifferentAccounts

NEW Account

```bash
   aws --profile new iam create-role --role-name cognito-migration-lambda-xxxx \
                       --assume-role-policy-document file://trust-policy.json
```

OLD Account

```bash
   aws --profile old iam create-role --role-name cognito-migration-lambda-xxxx \
                       --assume-role-policy-document file://old-trust-policy.json
```

---



2. Create Permissions for your lambda function to run
   * Update lambda-role-policy.json to the ARN of the *OLD* cognito user-pool (the one your migrating from)
   * "Resource": "arn:aws:cognito-idp:XXXXXXXXXXX" -> `OLD_USER_POOL_ARN`
   * Name your policy to match your DevOps procedures "cognito-migration-lambda-policy-xxxx"
   * For different accounts update new-role-policy.json and old-role-policy.json and use those respectively

SameAccount

```bash
   aws iam create-policy --policy-name cognito-migration-lambda-policy-xxxx \
                         --policy-document file://lambda-role-policy.json
```

DifferentAccounts

NEW Account

```bash
   aws --profile new iam create-policy --policy-name cognito-migration-lambda-policy-xxxx \
                         --policy-document file://new-role-policy.json
```

OLD Account

```bash
   aws --profile old iam create-policy --policy-name cognito-migration-lambda-policy-xxxx \
                         --policy-document file://old-role-policy.json
```

This allows your lambda function to authenticate and look up users against the old cognito instance
Note the Arn returned from the command `POLICY_ARN` `NEW_POLICY_ARN` `OLD_POLICY_ARN`

---


3. Attach Cloudwatch Permissions and the newly Created Policies to roles
   * Update role names to match your DevOps procedures
   * Different accounts use the respective commands don't for get to update the variables

SameAccount
```bash
   # Standard lambda execution policy, including cloud logging
   aws iam attach-role-policy --role-name cognito-migration-lambda-xxxxx \
                              --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

   # Attach the policy you just created in step 2
   aws iam attach-role-policy --role-name cognito-migration-lambda-xxxxx \
                              --policy-arn POLICY_ARN
```
DifferentAccounts

NEW Account
```bash
   # Standard lambda execution policy, including cloud logging
   aws --profile new iam attach-role-policy --role-name cognito-migration-lambda-xxxxx \
                              --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

   # Attach the policy you just created in step 2
   aws --profile new iam attach-role-policy --role-name cognito-migration-lambda-xxxxx \
                              --policy-arn NEW_POLICY_ARN
```
OLD Account

```bash
   # Standard lambda execution policy, including cloud logging
   aws --profile old iam attach-role-policy --role-name cognito-migration-lambda-xxxxx \
                              --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

   # Attach the policy you just created in step 2
   aws --profile old iam attach-role-policy --role-name cognito-migration-lambda-xxxxx \
                              --policy-arn OLD_POLICY_ARN
```

---
4. Create user pool client in old user pool
   * Update user-pool-id with the ID of the *OLD* user pool
   * This is the client that the lambda function will connect to validate user / passwords with
   * Note the ClientId returned from this as it will be your `OLD_CLIENT_ID`
   * Different accounts will need to run this but only on the old account. 

SameAccount
```bash
   aws cognito-idp create-user-pool-client \
      --user-pool-id [OLD_USER_POOL_ID] \
      --client-name lambda-migration-client \
      --no-generate-secret \
      --explicit-auth-flows "ALLOW_USER_PASSWORD_AUTH" "ALLOW_ADMIN_USER_PASSWORD_AUTH"
```

DifferentAccounts

NEW Account
```bash
NA
```
OLD Account
```bash
   aws --profile old cognito-idp create-user-pool-client \
      --user-pool-id [OLD_USER_POOL_ID] \
      --client-name lambda-migration-client \
      --no-generate-secret \
      --explicit-auth-flows "ALLOW_USER_PASSWORD_AUTH" "ALLOW_ADMIN_USER_PASSWORD_AUTH"
```


---
5. Create lambda function
   * Edit lambda-skeleton.json
   * Build the function code
   * Different accounts update the new-skeleton.json file 
   * Note the Lambda arn `LAMNDA_ARN`

```bash
   npm install && npm run build
```

* Deploy it
  * Note the Arn returned from this, this is your `LAMBDA_ARN`

SameAccount
```bash
   aws lambda create-function --cli-input-json file://lambda-skeleton.json
```

DifferentAccounts

NEW Account

```bash
   aws --profile new lambda create-function --cli-input-json file://new-skeleton.json
```
OLD Account
```bash
   NA
```
---
6. Attach lambda to new user pool
   * This is where you hook up your lambda function to your new cognito instance
   * Update the `NEW_USER_POOL_ID` and `LAMBDA_ARN`
   * Different Accounts you only need to add this to the new Account so that it will call lambda when the try to login. 

SameAccount
```bash
   aws cognito-idp update-user-pool \
  --user-pool-id NEW_USER_POOL_ID \
  --lambda-config  UserMigration=LAMBDA_ARN
```
DiffrentAccounts

NEW Account
```bash
   aws --profile new cognito-idp update-user-pool \
  --user-pool-id NEW_USER_POOL_ID \
  --lambda-config  UserMigration=LAMBDA_ARN
```

OLD Account
```bash
NA
```
# This is an adaptation from the https://github.com/Collaborne/migrate-cognito-user-pool-lambda.git

See this [blog post](https://medium.com/collaborne-engineering/migrate-aws-cognito-user-pools-ff2a91a745a2?sk=f91b73b84396db6f41a294f54bfeb2db) for a description
