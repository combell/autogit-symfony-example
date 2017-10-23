# Autogit: Symfony Example

This repository show you step by step how you can deploy a Symfony app to Combell
hosting using the Autogit feature.

* [Introduction](#introduction)
* [Create project](#create-project)
* [Git setup](#git-setup)
* [.autogit.yml](#autogityml)
* [Document root](#document-root)
* [Deployment hooks](#deployment-hooks)
  * [Composer](#composer)
  * [Shared files and folders](#shared-files-and-folders)
* [Deployment](#deployment)

## Introduction

All software projects benefit from a system like Git. Git is used to
 [track changes of code over time](https://git-scm.com/book/en/v2/Getting-Started-About-Version-Control).
You keep track of changes by [performing commits](https://git-scm.com/book/en/v2/Git-Basics-Recording-Changes-to-the-Repository)
 and some commits contain new features or bugfixes.

Since your Git repository now contains new versions of your project after commits, you can use it to
 [create releases](https://git-scm.com/book/en/v2/Git-Basics-Tagging).

With the Autogit feature of Combell, deploying is as simple as [pushing your commits](https://git-scm.com/book/en/v2/Git-Basics-Working-with-Remotes)
 to a repository hosted on the [Combell servers](https://www.combell.com/en/hosting/web-hosting).

## Create project

Create a new Symfony project using composer.

    composer create-project symfony/framework-standard-edition myproject
    cd myproject

## Git setup

Initialize a Git repository in the new folder and register the Combell Git remote.

The exact location of the repository is shown in the **control panel**, in this case it is
"domainbe@ssh.domain.be:auto.git".

    git init .
    git add .
    git commit -m "composer create-project symfony/framework-standard-edition"
    git remote add combell domainbe@ssh.domain.be:auto.git

## .autogit.yml

Copy over the .autogit.yml file and add it to the repository.

    scp domainbe@ssh.domain.be:autogit.yml.example .autogit.yml
    git add .autogit.yml
    git commit -m "Add the default .autogit.yml template"

## Document root

Symfony uses `web` as the document root, but Combell looks for the `www` folder.
Luckily for us, Git supports symlinks in the repository.
    
    ln -s web www
    git add www
    git commit -m "Symlink the www folder to the web folder (combell www -> symfony web)"

## Deployment hooks

When we push to the remote, our `.gitignore` file excludes the `vendor/` directory we need to run the application
in production. We would also like the log and session files to survive new releases.

### Composer

After the shared symlinks are created in the new release folder, we want to run `composer install`

    diff --git a/.autogit.yml b/.autogit.yml
    index ad503f9..1ddac5b 100644
    --- a/.autogit.yml
    +++ b/.autogit.yml
    @@ -34,7 +34,8 @@ hooks:
     #                present at every release (config, logs, ...)
       sharedsymlink_before: |
         exit 0
    -  sharedsymlink_after: | 
    +  sharedsymlink_after: |
    +    SYMFONY_ENV=prod composer install --no-dev --classmap-authoritative
         exit 0
     
     # SYMLINK: Set current symlink to newest release

And add it to the repository

    git commit -am "Run 'composer install' after the shared symlinks are created"

### Shared files and folders

The database (and other) settings in Symfony are stored in `app/config/parameters.yml`. The `var/` directory contains
log files, session files and caches.

We can register those in `.autogit.yml`:

    diff --git a/.autogit.yml b/.autogit.yml
    index 4ab5c46..722b24b 100644
    --- a/.autogit.yml
    +++ b/.autogit.yml
    @@ -13,8 +13,9 @@
     #######################################################################################
     
     # Example shared files and folders:
    -# shared_files: [ etc/config.yml ]
    -# shared_folders: [ var/log ]
    +shared_files:
    +  - app/config/parameters.yml
    +shared_folders: [ var ]

And add it to the repository

    git commit -am "Add the parameters file and var directory to shared objects"

When we symlink the `var/` directory, we need to delete it from the git repository (or the symlink will error out).

We could remove it from the repository, but we can also remove it from the release folder during deployment:

    diff --git a/.autogit.yml b/.autogit.yml
    index 37fdc61..eca3762 100644
    --- a/.autogit.yml
    +++ b/.autogit.yml
    @@ -29,6 +29,7 @@ hooks:
       install_before: |
         exit 0
       install_after: |
    +    rm -Rf var/ # Get rid of the git var folder
         exit 0
     
     # SHAREDSYMLINK: Create symlink to shared files and folders
    @@ -36,6 +37,9 @@ hooks:
       sharedsymlink_before: |
         exit 0
       sharedsymlink_after: |
    +    mkdir -p var/cache
    +    mkdir -p var/logs
    +    mkdir -p var/sessions
         SYMFONY_ENV=prod composer install --no-dev --classmap-authoritative --prefer-dist
         exit 0

And add it to the repository

    git commit -am "Remove the git var/ folder and recreate the subdirs when the shared symlink has been created"

## Deployment

    git push combell master
