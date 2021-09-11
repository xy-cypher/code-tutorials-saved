Source: https://sourceforge.net/p/omegat/wiki/Team%20projects%20with%20Github/
# Automated team projects management with Github

## History
|Date  	| Person	| Comment |
|---	|---	|---	|
| 2021.09.02  | Manuel Souto 	| Creation 	|
| 2021.09.08  | Manuel Souto 	| Update: repository mappings and section 2.6	|


## Acknowledgements
Warm thanks to Kos Ivantsov for his great advice.

## Introduction

This tutorial describes the necessary steps to create a Github repository non-interactively. 

The use case is for an organization that must fully automate the whole process and needs to create team projects for one or more languages. 

The intended audience is localization engineers and technically oriented PMs. You have one of those two roles. 

In this tutorial, "project" refers to an OmegaT project, which is a folder containing at least the `omegat.project` file. 

## 1 Preconditions

A number of preconditions are necessary to carry out the steps below.

### 1.1 For translators

The members of your translation team must [have a Github account](https://docs.github.com/en/get-started/signing-up-for-github/signing-up-for-a-new-github-account) and [a personal access token](https://docs.github.com/en/github/authenticating-to-github/keeping-your-account-and-data-secure/creating-a-personal-access-token#creating-a-token). 

### 1.2 For the project manager

Before creating the team projects, you must have:

* [installed gh on the command line](https://cli.github.com/manual/installation)
* [created an organization in Github](https://docs.github.com/en/organizations/collaborating-with-groups-in-organizations/creating-a-new-organization-from-scratch)
* [renamed the repository default branch from `main` to `master`](https://docs.github.com/en/organizations/managing-organization-settings/managing-the-default-branch-name-for-repositories-in-your-organization#setting-the-default-branch-name)
* [added your translators as members of the organization](https://docs.github.com/en/enterprise-server@3.0/organizations/managing-membership-in-your-organization/adding-people-to-your-organization)
* [set the organization's base permissions to give members writing privileges](https://docs.github.com/en/organizations/managing-access-to-your-organizations-repositories/setting-base-permissions-for-an-organization)

## 2 Managing the team project

The following commands can be run in either git bash, Windows WLS or bash on real GNU/Linux. Upper cased names are used as placeholders for the actual repository, organization, team, etc.

### 2.1 Authentication

To execute the steps below through SSH, the project manager must be authenticated. The following commands (followed by the your passphrase) can be run once to avoid having to type the passphrase every time a repo is created:

```bash
$ eval `ssh-agent -s`
$ ssh-add ~/.ssh/*_rsa
Enter passphrase for /home/USER/.ssh/id_rsa:
Identity added: /home/USER/.ssh/id_rsa (/home/USER/.ssh/id_rsa)
```

It will ask for the passphrase in the second command, and that's it. You only need to do this once per session.

### 2.2 Creating the repository

Navigate to the parent folder where you would like to create the OmegaT folder that you want to push to the repository. There, run the following command: 

```bash
$ yes no | gh repo create ORG/REPO --confirm --private --team TEAM
```

The basic command is `gh repo create REPO`. The rest of arguments are optional:

* `--confirm` (or `-y` for short) skips the confirmation prompt
* `--private` makes the new repository private (alternatively, you can use `--public` if privacy is not required)
* `--team TEAM` points to the organization team that will be granted access to the repo
* `yes no |` answers "No" to any further prompts, such as whether you'd like to add a .gitignore or a license

The expected output of that command is:

```bash
✓ Created repository ORG/REPO on GitHub
Initialized empty Git repository in /home/USER/path/to/parentdir/REPO/.git/
✓ Initialized repository in "REPO"
```

You can see these and other options with [`gh repo create --help`](https://cli.github.com/manual/gh_repo_create).

### 2.3 Initializing the OmegaT project

After creating the repository, the next step is to initialize the OmegaT project. You may do so from inside the project folder with the following command (to create a `en-gl` project):

```
cd REPO
java -jar OmegaT.jar team init en gl
```

That creates  a barebones project folder structure:

```
souto@ameijoa:REPO$ tree
.
├── dictionary
├── glossary
│   └── glossary.txt
├── omegat
│   └── project_save.tmx
├── omegat.project
├── source
├── target
└── tm
```

Finally, add source files, config files, language assets (TMs, glossaries, etc.) as needed.

### 2.4 Pushing the OmegaT project to the repository

The project is ready to be pushed to the remote repository. From inside the project folder, do:

```bash
$ git add .
$ git commit -m "Initial commit -- uploading project files"
$ git push --set-upstream origin master
```

Example output:

```
  Enumerating objects: 82, done.
  Counting objects: 100% (82/82), done.
  Delta compression using up to 8 threads
  Compressing objects: 100% (79/79), done.
  Writing objects: 100% (82/82), 308.64 KiB | 750.00 KiB/s, done.
  Total 82 (delta 28), reused 0 (delta 0)
  remote: Resolving deltas: 100% (28/28), done.
  To github.com:ORG/REPO.git
  
   * [new branch]      master -> master
   Branch 'master' set up to track remote branch 'master' from 'origin'.
```

### 2.5 Sharing the team project with your translator(s)

So as to download the team project on their side, your translator(s) need to know the URL of the team project. The URL is predictable: `https://github.com/ORG/REPO.git`. 

If you would like to confirm it and you use the HTTPS protocol, you can get the URL with the command:

```bash
git config --get remote.origin.url
```

If you use the SSH protocol, you can get the URL like so:

```bash
git config --get remote.origin.url | sed -e 's/:/\//g'| sed -e 's/ssh\/\/\///g'| sed -e 's/git@/https:\/\//g'
```

### 2.6 Harvesting the translations

To obtain the translated documents in the target language, you have two options: you may ask your translator to generate the target files (e.g. `Ctrl+D`) and commit them to the remote repository (i.e. **Project > Commit Target Files**). 

Alternatively, you may simply pull all translations from the repository to a local copy of the project in your machine (outside the git checkout folder) and generate the target files there. Running OmegaT on the project in console mode (`java -jar /path/to/OmegaT.jar /path/to/proj --mode=console-translate`) should do the trick.

### 2.7 Multilingual projects 

If you are managing a multilingual project, where translations need to be produced in several target languages, you may do the steps above for each of them. It's easy to automate that with a script, here's a suggestion in pseudo-code, for which you need a list of all the target language tags (and assuming the source language is constant and already defined in the template).

- For each tag in the list of language tags:
  - create repo with language tag in its name, e.g. `REPO_gl`
  - initialize the project
  - add source files, language assets, config files, etc.
  - add, commit and push files to the remote repository
  - add URL to a list that you can then publish or share with your translators

Your translators can then start working.

When the time to harvest the translations comes, you can do a similar iteration to compile and collect the target documents.

* For each project/repo folder `$proj`:
  * change directory to that folder
  * pull contents from the repo (check `git pull --help` for details)
  * change directory to the parent folder
  * generate the target files (e.g. `java -jar /path/to/OmegaT.jar $proj --mode=console-translate`)
  * put the target files in the deliverable bundle or folder you will submit to the client

### 2.8 Different language tasks

Your project might have different levels of intervention (translation, revision, review, etc.), so that different linguists or users must work on it. You must add all those users as team members to your organization `ORG`. 

If you want a reviser not to be able to start until the translator is done, in that case you must make sure that your reviser does not have access to the repository until the translator has finalize the translation task. Likewise, you might want to revoke the translator's access to the project once the reviser has started revising. 

> @PENDING: How to grand and revoke access to the REPO or to the ORG using `gh` in the command line?

## 3 Repository mappings

Instead of including all files in the OmegaT project that you commit to the repo, you can have a distributed organization of your project(s) with files stored in different locations. That has the advantage of making your commit for every language in a multilingual project much lighter.

You must commit at least the project settings (i.e. `omegat.project`) file for each project, but OmegaT can fetch everything else from elsewhere, through links included in the repository mapping section of the project settings file. For example, OmegaT could fetch the source files from a common repository, language assets from another repo, etc. 

If the files are hosted in a versioning repository (e.g. github, gitlab, etc.), you can either map files or folders. If your files are hosted in a normal server, only files can be mapped.

For example, given a remote repository that contains the following files:

```sed
.
├── proj_conf
│   └── segmentation.conf
└── source
    └── final
        ├── file1.docx
        └── file2.docx
```

the following mapping will fetch all files from the remote folder `source/final` and put them in the `source` folder of the local project. Also, it will fetch remote file `proj_conf/segmentation.conf` and put it in the `omegat` folder of the local project.

```xml
<repository type="git" url="https://github.com/ORG/REPO_source.git">
  <mapping local="source/" repository="source/final"/>
  <mapping local="omegat/segmentation.conf" repository="proj_conf/segmentation.conf"/>
</repository>
```

You may also fetch a subset of the files in a remote folder by explicitly including only the ones that a certain pattern matches, e.g. matching the target language of the project. The following example fetches any existing TMX files for a project with `gl` (Galician) as target language from a remote location that contains TMs for all languages. Notice that all files need to be excluded first.

```xml
<repository type="git" url="https://github.com/ORG/REPO_assets.git">
  <mapping local="/tm" repository="/"/>
    <excludes>**/*</excludes>
    <includes>*gl*.tmx</includes>
</repository>
```

You might map several folders from the same repository, which in the case of TMs allows to use different levels of validation (e.g. enforced, auto-populated, penalties, etc.). The following examples map TMs named as per year 2021 from the remote root folder to the `/tm/auto` folder of the project and TMs named as per year 2018 from the `penalty-02` remote folder to the `/tm/penalty-02` folder of the project.

```xml
<repository type="git" url="https://github.com/ORG/REPO_assets.git">
  <mapping local="/tm/auto" repository="/"/>
    <excludes>**/*</excludes>
    <includes>*2021*gl*.tmx</includes>
  <mapping local="/tm/penalty-02" repository="/penalty-02"/>
    <excludes>**/*</excludes>
    <includes>*2019*gl*.tmx</includes>
</repository>
```

It is also recommended to add the map the main repository itself (where the `omegat.project` file is hosted).

```xml
<repository type="git" url="https://github.com/ORG/REPO_gl.git">
  <mapping local="/" repository="/"/>
</repository>
```

All the above examples would be children of the  `<repositories>` node in the project settings. The same principle would apply to other files to be added to the project (e.g. glossaries). 

> @PENDING: How to create the entire project folder structure on the command line (as the GUI does when downloading the project) from a repo that only contains the `omegat.project`?
