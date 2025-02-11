# Course Onboarding

## TL;DR

Onboarding steps:

- [Git](#git) and [GitHub account](#GitHub)
- [GCP account](#aws-account)
- [PyCharm (or any other alternative)](#pycharm)
- [Clone the course repo](#clone-the-course-repository-into-pycharm)
- Install `gcloud`


## Git and GitHub

During the course we will explore and practice continues deployment techniques with ArgoCD. 
You will be required to work against a dedicated GitHub repo with your YAML manifests.  

If you haven't already, please create a [GitHub account](https://github.com/) and install Git on your local machine.


## GCP Account

We'll be working in a shared GCP account. 
Credentials for a shared GCP account will be provided to you in class. 

## PyCharm

We need some **Integrated Development Environment (IDE)** for convenient editing YAML files and applying them using an available terminal.

You can also use any IDE to your choice. Below is installing information for Pycharm. 

PyCharm is an IDE software for code development, with Python as the primary programming language. 
Install **PyCharm Community** from: https://www.jetbrains.com/pycharm/download/ (scroll down for community version).

### Clone the course repository into PyCharm

Cloning a GitHub project creates a local copy of the repository on your local computer.

Here is how to clone the repository using PyCharm UI:

1. Open PyCharm. 
    - If no project is currently open, click **Get from VCS** on the Welcome screen.
    - If your PyCharm is opened of some existing project, go to **Git | Clone** (or **VCS | Get from Version Control**).

2. In the **Get from Version Control** dialog, specify `https://github.com/alonitac/KubernetesForDevelopers25.git`, the URL of our shared repository. 
2. If you are not yet authenticated to GitHub, PyCharm will offer different types of authentication methods. 
   We suggest to choose the **Use Token** option, and click the **Generate** button in order to generate an authentication token in GitHub. 
   After the token was generated in GitHub website, copy the token to the designated place in PyCharm. 
3. In the **Trust and Open Project** security dialog, select **Trust Project**. 

At the end, you should have an opened PyCharm project with all the files and folders from the cloned GitHub repository, ready for you to work with.
