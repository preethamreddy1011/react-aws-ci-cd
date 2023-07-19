# Getting Started with Create React App

This project was bootstrapped with [Create React App](https://github.com/facebook/create-react-app).

## Available Scripts

In the project directory, you can run:

### `npm start`

Runs the app in the development mode.\
Open [http://localhost:3000](http://localhost:3000) to view it in your browser.

The page will reload when you make changes.\
You may also see any lint errors in the console.

### `npm test`

Launches the test runner in the interactive watch mode.\
See the section about [running tests](https://facebook.github.io/create-react-app/docs/running-tests) for more information.

### `npm run build`

Builds the app for production to the `build` folder.\
It correctly bundles React in production mode and optimizes the build for the best performance.

The build is minified and the filenames include the hashes.\
Your app is ready to be deployed!

See the section about [deployment](https://facebook.github.io/create-react-app/docs/deployment) for more information.

### `npm run eject`

**Note: this is a one-way operation. Once you `eject`, you can't go back!**

If you aren't satisfied with the build tool and configuration choices, you can `eject` at any time. This command will remove the single build dependency from your project.

Instead, it will copy all the configuration files and the transitive dependencies (webpack, Babel, ESLint, etc) right into your project so you have full control over them. All of the commands except `eject` will still work, but they will point to the copied scripts so you can tweak them. At this point you're on your own.

You don't have to ever use `eject`. The curated feature set is suitable for small and middle deployments, and you shouldn't feel obligated to use this feature. However we understand that this tool wouldn't be useful if you couldn't customize it when you are ready for it.

## CI/CD steps

### Create the S3 Bucket
Login to the S3 console, click ‘Create bucket’ and give the bucket a name like “applicatio-name-hosting”. Use the following settings when setting it up:

- ACLs enabled: true
- Turn off “Block all public access” and accept the disclaimer below it
- Accept or modify any of the other defaults as needed
- Open up the newly created bucket, click on ‘Permissions’ and then edit the bucket policy so that is looks like this:

    ``` json
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Sid": "PublicReadGetObject",
                "Effect": "Allow",
                "Principal": "",
                "Action": [
                    "s3:GetObject",
                    "s3:GetObjectVersion"
                ],
                "Resource": "arn:aws:s3:::<bucket-name>/"
            }
        ]
    }
    ```
### Create the CloudFront Distribution
Open up the CloudFront dashboard, click “Create Distribution”. In the resulting create screen, specify the following:

- Origin Domain: in the drop-down select the ARN of the S3 bucket from the previous step.
- View protocol policy: choose “Redirect HTTP to HTTPs”

Once the distribution is created, you will need to make one modification to the Error pages behavior so that react-router plays well with the site being hosted on CloudFront and S3.
- Open up the newly created Distribution, then open up “Error Pages” tab, and then click “Create custom response”. Set the following properties:
    - HTTP error code: 403
    - Customize error message: yes
    - HTTP Response page path: index.html
    - HTTP Response code: 200

### Setup the CodePipeline and CodeBuild Projects
Create a new CodePipeline
Open the AWS CodePipeline console, click on “Create pipeline”

- Service role: select “New service role”
- Accept the default Role name populated by CodePipeline and check the “Allow AWS CodePipeline to create a service role” option.

Clicking “Next”, you will then be prompted to choose a “Source provider” for the pipeline. In the dropdown box, choose the source code service where the source code for the project is hosted. Once selected, then you then need to configure the connection properties to that service. Ultimately after the connection is configured, you will need to provide the following pieces of information:

- Repository name
- Branch name

Clicking “Next”, you will then be prompted to choose a “Build provider”. In the dropdown box, select “AWS CodeBuild”, then click on “Create project”. This will trigger a new pop-up window which will walk you through the process of creating the CodeBuild project before returning you back to the CodePipeline creation wizard.

In the CodeBuild setup wizard, make the following choices:
- Environment image: Managed Image
- Operating system: Amazon Linux 2
- Runtime(s): Standard
- Image: aws/codebuild/amazonlinux2-x86_64-standard:4.0
- Accept all other defaults for now and press “Continue to CodePipeline”. Take note of the ARN of the CodeBuild service role created as part of the CodeBuild project as you will need it later.

At this point, you should be back in the CodePipeline wizard, click “Next” and then choose “Skip deploy stage” and then “Finish” to start the creation of the CodePipeline.

### Grant the CodeBuild IAM Role Permission to Access the S3 Bucket
Open the S3 Console and then open the S3 bucket we created previously. Click on Permissions and then in Bucket Policy edit the JSON policy so that it looks like this:
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": [
                "s3:GetObject",
                "s3:GetObjectVersion"
            ],
            "Resource": "arn:aws:s3:::<bucket-name>/*"
        },
        {
            "Sid": "Stmt1597860486271",
            "Effect": "Allow",
            "Principal": {
                "AWS": "<arn of CodeBuild IAM role>"
            },
            "Action": "*",
            "Resource": "arn:aws:s3:::<bucket-name>/*"
        }
    ]
}
```

### Grant the CodeBuild Service Role Permissions to the CloudFront CDN
Open the IAM console and create a new Policy with the following JSON:
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": "cloudfront:CreateInvalidation",
            "Resource": [
                "<arn of CloudFront Distribution>"
            ]
        }
    ]
}
```
After creating the above Policy, then open up the IAM Roles section, locate the CodeBuild service role and click it open. Under the ‘Permissions’ tab, click ‘Add Permissions’ and attach the Policy you just created in the previous step.

### Configure the CodeBuild project “buildspec”
In the CodeBuild console, open up the CodeBuild project created when we created the CodePipeline pipeline. Under ‘Edit’, choose ‘Buildspec’ and then change the ‘Build specification’ option to be ‘Insert build commands’ and then choose ‘Open Build Editor’.

In the resulting window, you will configure the build commands run by CodeBuild through a standardized YAML file. (Take note of the <> brackets in the YAML, you will need to replace these with the actual values specific to your deployment)
```yaml
version: 0.2

phases:
  install:
    #If you use the Ubuntu standard image 2.0 or later, you must specify runtime-versions.
    #If you specify runtime-versions and use an image other than Ubuntu standard image 2.0, the build fails.
    runtime-versions:
      nodejs: 16
  pre_build:
    commands:
      - echo Installing source NPM dependencies...
      - npm install
  build:
    commands:
      - echo Build started on 'date'
      - npm run build
  post_build:
    commands:
      - aws s3 cp --recursive --acl public-read ./build s3://<bucket-name>/
      - aws s3 cp --acl public-read --cache-control="max-age=0, no-cache, no-store, must-revalidate" ./build/index.html s3://<bucket-name>/
      - aws cloudfront create-invalidation --distribution-id <cloudfront-distribution-id> --paths /index.html
artifacts:
  files:
    - "build/*"
    - "build/**/*"
```

### Run the CodePipeline and Verify it Works
Head back over to the CodePipeline console, locate the CodePipeline you created previously. When you open it you will see that the Source step succeeded, while the Build step failed. 
Now that we have finished the configuration, click on the X on the right-hand side of the Build phase. This will trigger CodePipeline to retry the Build phase again. This time you should see the Build phase succeed!