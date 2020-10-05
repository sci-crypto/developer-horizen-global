![horizen_logo](https://www.horizen.io/assets/img/icons/page_media/logo_with_tagline_2.png)

# Build Status

[![Documentation Status](https://readthedocs.org/projects/developerhorizenglobal/badge/?version=latest)](https://docs.horizen.global/en/latest/?badge=latest)

# Horizen Sidechains SDK documentation.

## This repo contains the restructured text source files for our [Sidechain SDK wiki](https://docs.horizen.io) which are hosted via [readthedocs.org](https://readthedocs.org)

### We encourage community input to this repo. You will find the requirements to do so below

#### Ways to Contribute

- **Raise an issue [here](https://github.com/HorizenOfficial/developer-horizen-global/issues)** 
Ensure you issue includes a full and concise description of the changes you feel are needed. Please ensure your issue is specific to the page and section of the documentation you are referring to. If your contribution is complex, please see the next bullet point.

- **[Fork](https://docs.github.com/en/free-pro-team@latest/github/getting-started-with-github/fork-a-repo) our [repository](https://github.com/HorizenOfficial/developer-horizen-global) and clone locally**.
You can then make your proposed changes, and push to your remote repository. At this point create your pull request against the **[official repo](https://github.com/HorizenOfficial/developer-horizen-global)**.
**When creating your pull request against our repo please raise it against our [development](https://github.com/HorizenOfficial/developer-horizen-global/blob/development) branch. If you issue a pull request against the master branch, you will be asked to change it.**
  
We will review pull requests in the order that they arrive. In some cases where content is based on features that are still in development or we have large high priority tasks in progress, we may take some time to either comment on, merge, or request changes to your contribution.

#### Local development tools

- We include a local docker build tool to allow you to see changes as you are making them on your machine. This allows you to see what your contribution looks like in you local browser of choice before you actually submit your pull request.
To use this functionality please ensure you have **[Docker](https://docs.docker.com/engine/install/)** correctly installed for your platform along with **[docker-compose](https://docs.docker.com/compose/install/)**.
From the terminal change directory to the root of the project and issue the following command.

    ```bash
    docker-compose up --build
    ```

    After the build finishes you can open your browser and head to **[http://127.0.0.1:8100](http://127.0.0.1:8100)**.

    The browser window will automatically refresh on every code edit to the local source files to allow you to see how the documentation would look like as if it were live on our system.