---
title: "Share your first open-source project on GitHub"
author: ""
type: ""
date: 2020-08-04T12:00:00+03:00
subtitle: ""
image: ""
tags: []
private: false
---
Do you want to share your code to other developers but you don’t know how? In the open-source community there are some approaches that you have to be familiar with. You cannot just push your code to a repository and wait for somebody to use it. You have to go through several steps before your project will be noticed by other people.

<!--more-->

Why on GitHub? GitHub is a de facto standard for open-source projects. You share your code with other users, you can find like-minded people that will contribute to your project. GitHub has some tools for helping developers like Pull Request, Actions, Packages.

---

**Disclaimer:** I cannot write about everything in detail because there is too much information. I will write only about basic concepts. I will give you links to documentation by the way for further learning.

---

## Step 0: Find an Idea
If you already have an idea for your project then you're in luck but if you don’t you have to start from looking for the idea.

The best open-source project is a project that you make for yourself then share to other people. Analyze your daily habits and investigate what kind of tool you aren’t enough in your day to day life.

## Step 1: Choose a Technology Stack
**For a long-term project**, it’s better to apply the technology stack that you are already familiar with.

Why is the familiar technology stack? Because when you use unknown technology you can spend too much time learning this new technology that may cause you to lose motivation or to meet a problem that you won’t be able to solve. But if you are confident about your abilities, you can apply unfamiliar technology for a long-term project.

**For a short-term project**, choose any technology that you want. You can even change the stack at any time and completely rewrite the project if the technology doesn’t suit you.

## Step 2: Write Readme and License Files
Okay, you have found an idea and have chosen your favorite technology stack. Now, you can set up a new repository for your project on GitHub.

It is time to talk about two main files inside your repository — readme, and license.

Your open-source project has to have a readme file that contains information about the app. The readme file should consist of three sections at least:
1. A detailed description of your app and what the app can do, as well as its goals. This allows you and other people to know what the app is for.
2. An installation guide. This allows you to think about the installation process and make it more convenient for the user in advance.
3. A user guide on how to use the app. Here you can also picture which user interface you will use in the future and describe the user guide in advance.

What about the license? I always choose the MIT license for all my open-source projects. [Read this article](https://choosealicense.com/licenses/) and choose the most suitable license for your app. The license is necessary because other users have to know how they can use your source code.

Also, it will be convenient to have `.gitignore` file for the app’s sources because you won’t commit unnecessary files accidentally.

## Step 3: Create a Simple Working Version of the App
Unless you have any code for your app you can create a minimal working version. The code has to compile successfully but it can do nothing useful. It could be a blank page with your name, or simple CLI command, or bot that greets you. Working code is required for configuration of a development environment on GitHub.

## Step 4: Automate Building of the App
GitHub Actions is a CI tool that allows you to execute a build script for your app automatically after occurance of some events in your repository. 

The build script has to have two checks at least :
* Building — the most important check that verifies the source code on syntax errors
* Running unit tests — also an important check that verifies the app’s features, doing what they must do without bugs.

If you want more reliability, you can set up yet another two checks:
* Static analysis — this verifies source code on code style, security vulnerabilities, and will fail the build if they have issues.
* Code coverage —this verifies counts of code lines that run by the unit tests and fail the build if the coverage is less than a certain percent. The best percent of coverage is at least 80%. For example, for 100 code lines, this means unit tests must run 80 lines of the source code.

If one of the checks fails then GitHub Actions will inform you about that. In case of fail ...

At the moment we interested in three types of repository events:
1. pushing a new commit to main branch
2. creating a pull request
3. creating a new release

GitHub already has GitHub Actions templates for different languages. Your task is to follow the wizard’s suggestion for your language and do small changes if you need it.

## Step 5: Automate Release of the App
Publish a new release on GitHub after completing each significant feature. Set up yet another GitHub Action that will trigger on release tag and will build app packages.

Type of package depends on type of your project:
* For service, it will be a docker image that you can push to GitHub Packages. 
* For CLI and other tools, it will be a runnable binary file that you can attach to GitHub Release assets. 
* For a library, it will be a package that a user can add to his project as a depensendy. The library package has to be pushed to GitHub Packages or another package store like npmjs.com, maven.org, etc.

## Step 6: Manage Tasks in Your App's Project
Usually, you develop open-source projects in your free time. Of course, you want to develop it more productively, because your free time is limited. Here project management will help you. Before starting planning tasks, you need to design the architecture of the app.

How do I do this? Just draw an architecture scheme on paper. This may not be easy, initially draw a draft version of the scheme and improve it as you develop the app.

Now you have gotten the architecture’s scheme and you know what kind of tasks you will do. Find the most important features of the app and hence the necessary parts of the scheme for the features. Split this part of the scheme into small tasks where each task takes less than an hour.

Write down each task somewhere, then evaluate priority for each task by their importance and sort tasks by priority. Put these tasks to the readme file or your favorite to-do list, or create for each task the issue on GitHub.

Read my article about tasks estimation.

Periodically repeat the process of task planning and evaluating priority, because as you develop the app the requirements will change, as will the architecture of the app. Don’t be afraid to drop tasks that lose their value.

## The Time Has Come!
Finally, you can begin developing your app. I recommend you take the following approach:
1. Take a task with high priority.
1. Create a branch in git for the task.
1. Implement the task.
1. Write several unit tests (or you can use TDD).
1. Push the branch to GitHub.
1. Open Pull Request on GitHub.
1. Wait until GitHub Action has built your commit. If the build failed you must fix it and then rebuild.
1. When the build is passed and becomes green you can merge this Pull Request.
1. Publish the release, wait until GitHub Action has built the app packages.
1. Use the latest version of the app by yourself.

After every implemented task you must have the correct working app which deployed somewhere.

## Promote the App
to let users know about the project

write an articles
film the video.

## Conclusion
Let’s recap the most useful tips:
* Find an idea that interests you
* Write a detailed readme file which includes a description of the project, installation guide, and user * guide
* Choose a license for the source code and create the license file
* Write the minimum working app and set up a development environment for it
* Use GitHub Actions for automated build
* Develop an app’s architecture and draw an architecture scheme
* Plan and prioritize tasks
* Use your app yourself
* Promote your app to let users know about the project

That’s all, don’t stop and continuously improve your app! 

Thanks for reading, I hope you have found this article helpful.
