
# Chemster Workflow Interactive Program
Add this script to your terminal to use the custom actions to create a project, create a new branch from an existing project, commit code and generate a merge request.  This program does all the heavy lifting to properly set up, maintain and keep projects, CI & stories in sync.

## Dependencies
Please make sure the common dependencies are installed:
1. Git
2. Curl
3. AWS Cli
4. Docker
5. Homebrew

You can install aws-cli by following this guide:
[https://docs.aws.amazon.com/cli/latest/userguide/install-macos.html](https://docs.aws.amazon.com/cli/latest/userguide/install-macos.html)


## Setup

Set up your ssh key on Gitlab:
Please make sure, you have added a ssh key in the gitlab profile so that you can clone, commit and push projects.

```sh
ssh-keygen -t rsa -b 4096
pbcopy < ~/.ssh/id_rsa.pub
```
Then go to [https://gitlab.com/profile/keys] (https://gitlab.com/profile/keys)
and add in your ssh key.

Next, clone this repo to your machine:
```sh
git clone git@gitlab.com:chemster/chemster-engineering/eng-toolbox/workflow-automation.git
```

Lastly, add this script in your terminal by:
```sh
source <complete-path>/chemster-workflow.sh
```
You will need to repeat the above in every terminal tab or new tab.

If you do not want to source the code every time, there is a small wrapper in `chemster.sh`, that you can use instead of the `chemster` function.
So if you include that in your `PATH`, it will be available in your terminal.

## Commands
After adding the script and exporting the Clubhouse api token, you can use our commands which are:
### Install

```sh
chemster install
```
1. Interactively sets up homebrew
2. Installs brew jq rename node truncate
3. NPM install serverless and serverless-offline
4. Makes the chemster projects directory
5. Makes the templates directory (where we put all of our starters)
6. Clones all the starters, pipeline templates and language specific shared utils
7. Interactively sets up your clubhouse token
8. Interactively sets up your gitlab token
9. Interactively sets up the AWS CLI


### New project
You can run this command as:
```sh
chemster new
```
This command does the most of any of the commands in this program.  The following command line args can be passed to speed up project creation:
```
-d Deployment
-t type (language)
-n name
```
```sh
chemster new -t node -d sls -n edit-brief
```

Below is a list of all the actions handled by this flag:
Based on the language and deployment type:
1. Create the project directory in the chemster-projects directory.
2. Pulls the correct starter from our Eng toolbox
3. Copies the correct files to the project root
4. Removes any unnecessary files from the starter
5. Copies the correct gitlab-ci file from the pipeline templates
6. Creates a initDetail.yml that contains all the project details for reuse
7. Creates a local-env file and adds it to the .gitignore list (this is used in commits and reviews later)
8. Makes a new project on gitlab with several custom settings
9. Adds the clubhouse api webhook to the project to keep the two linked
10. Adds the slack notifications for notifications sent to the eng-workflow channel (merge requests and pipeline failures)
11. Moves the starter remote from origin to base-template
12. Set the new gitlab project URL to be origin
13. Copies and stores a git--project-config file to be loaded during new branch creation to ensure all submodules and base-templates references are set up correctly
14. Makes the first commit to push master to origin.
15. For ECS, it will build a docker image and then pushes it to the AWS ECR
16. For Lambda, SERVERLESS_XXXX ENVs are added to the gitlab project as well as to your terminal.  They are dynamic variables used in the serverless.yml file (see sls-init below)

Optional:
1. If you have a story to work on, it goes out and gets the name and then uses it to create a branch following the naming convention to keep gitlab and clubhouse in sync
2. Then during the first commit on the branch it initiates a webhook on clubhouse to move the story to In Development ` [chXXX] [start]`
3. For ECS deployments, a clubhouse infrastructure chore will be created for supporting resources needed for deployment.  This functionality may be used for other type of clubhouse requests in the future.  


### New branch
```sh
chemster branch
```

This command creates a branch on local that will follow the naming conventions to keep clubhouse and gitlab in sync, as well as keeping our pipeline templates and notifying us if a change occurred in a starter repo.

The branch flag does the following:
If you are in a directory with a .git folder it will default to this directory, otherwise you can put in any directory name that lives within CHEMSTER_PROJECTS

Goes to the project directory and...
1. Checks out master (or whatever branch specified)
2. Copies the git-project-config file to .git/config
3. Reads in the initDetail.yml file for the project configurations
4. With the story ID it goes to clubhouse to get the story name.
5. Creates a new branch following the naming convention to keep gitlab and clubhouse in sync
6. Checks if the pipeline-template matching this deployment has changed then overwrites the .gitlab-ci.yml file
7. Checks if a default starter file was changed and notifies to review the change and implement if needed
8. Then it does a commit on the branch to initiate the clubhouse webhook which moves the story to In Development ` [chXXX] [start]`


### Commit
```sh
chemster commit
```
We have wrapped up git commit / git push command and tied in a few other things to make our life easier.  

1. Checks if the pipeline-template matching this deployment has changed then overwrites the .gitlab-ci.yml file
2. Checks if a default starter file was changed and notifies to review the change and implement if needed
3. Checks for untracked files and asks to add them to the commit
4. Reads in the local-env file and then pushes anything new or updated to gitlab
5. Runs a git commit
6. Runs a git push
7. Read in the local-env file and add/update the dotenv variables to Gitlab
8. For ECS, builds and pushes an updated docker image to ECR


### Merge Request
```sh
chemster review
```

This flag opens a merge request and moves your story to ready for review.  Under the hood this is what is happening:

1. The 7 steps above in commit with one adjustment, on commit the [pr] tag is added to move the story to ready for review
2. Goes to the clubhouse to grab the story description and inserts it into the merge request description
3. Creates the merge request on gitlab
4. This triggers a notification to the eng-workflow channel


## Quick Commands
The following commands are some light weight utilities that only verify a gitlab token is set up.  

### Initialize Existing Project
Run this command when you are going to be working on an existing project and you want to load up the ENVs on Gitlab for a .env file.

1. Goes out and gets all the ENVs stored on a project
2. Truncates both the local-env file and .env file if they exist
3. Removes the DOTENV_ prefix from gitlab and adds the entries to a local-env file
4. Removes the DEV_ prefix and adds the entries to a .env file for your project.

```sh
  chemster init
```

### Build a .env file
This command can be run anytime a local-env file exists and there are entries with the DEV_ prefix.
```sh
  chemster senddotenv
```

### Send ENV changes to Gitlab
This command can be run anytime a local-env file exists.  The objective was to provide a simple and easy way to push local-env changes up to gitlab outside or a chemster commit or chemster review.

```sh
  chemster senddotenv
```


### Serverless Initialize
This is needed if you want to use serverless-offline.  It is not currently added to the plugins of the serverless.yml or as a package on the project.  

Serverless offline needs some of our ci env variable to run so sls-init was created to export those variables so you can run sls commands locally. You must be in your serveress project folder to run this command.

  ```sh
  chemster sls-init
```


### Updating Starter
```sh
chemster update-starters
```

This will update all starter repositories. Note that this happens automaticaly once a day when running any chemster command.
