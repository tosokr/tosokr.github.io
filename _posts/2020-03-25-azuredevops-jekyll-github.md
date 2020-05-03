---
title: Using Azure DevOps to build and release a Jekyll site
date: 2020-03-25 11:00:00 +0000
description: How to access Key Vault reference stored in App Configuration from .NET Framework console application
categories: [Azure DevOps]
tags: [Azure DevOps,Jekyll]
header:
 teaser: "/assets/img/posts/teasers/azureDevOps.png"
permalink: /posts/azuredevops-jekyll-github/
---
>"Jekyll is a static site generator. You give it text written in your favorite markup language and it uses layouts to create a static website. You can tweak how you want the site URLs to look, what data gets displayed on the site, and more."

When I decided to create a blog where I can get the technical stuff out of my head, I set up a goal – it needs to be FREE. After researching, I come out with the solution: static website produced by Jekyll hosted on GitHub Pages. For the building and releasing, I'm using a public project in Azure DevOps. For the public projects, Microsoft is providing for free 10 Microsoft-hosted parallel jobs that can run for up to 360 minutes (6 hours) each time, with no overall time limit per month.

<h3 data-toc-skip>Source control</h3>
Because my repository is a fork from [cotes2020/jekyll-theme-chirpy](https://github.com/cotes2020/jekyll-theme-chirpy) repository, it is much simpler to me to keep the source in [GitHub](https://github.com/tosokr/mysite).

<h3 data-toc-skip>Build</h3>
For building the website, I use the prebuilt docker image: jekyll/builder. The build pipeline definition looks like this:
```yaml
# Trigger the pipeline execution for every commit on the master branch
trigger: 
- master

# Use a Microsoft hosted agent based on the latest version of ubuntu
pool:
  vmImage: 'ubuntu-latest'
steps:
# Run a docker image
- task: Docker@0
  displayName: 'Run Jekyll'
  inputs:
    containerRegistryType: 'Container Registry'
    action: 'Run an image'
    # use the latest version of jekyll/builder docker image from Docker Hub
    imageName: 'jekyll/builder:latest'
    # mount the source code directory and binaries directory in the container
    volumes: |
      $(build.sourcesDirectory):/srv/jekyll
      $(build.binariesDirectory):/srv/jekyll/_site
    # set an environment variable inside the container
    envVars: 'JEKYLL_ENV=production'
    # execute a custom script when the container starts
    containerCommand: '/srv/jekyll/tools/azuredevopsbuild.sh -d /srv/jekyll/_site'
    # do not run the container in background
    detached: false
# publish the artifact to Azure DevOps
- task: PublishBuildArtifacts@1
  inputs:
    pathtoPublish: '$(build.binariesDirectory)'
    artifactName: site
```
You can find the custom run script in my GitHub source repository [here]( https://raw.githubusercontent.com/tosokr/mysite/master/tools/azuredevopsbuild.sh)

<h3 data-toc-skip>Release</h3>
To be able to push the static website to GitHub, you need a way to authenticate. Generate your Personal Access Token on this [page](https://github.com/settings/tokens). After copying the token in the release pipeline, create a secret variable with the name PAT and paste your token. Azure DevOps never expose secret variables value in the logs and to any user. 

Install the Azure DevOps extension named [GitHub Pages Publish](https://marketplace.visualstudio.com/items?itemName=AccidentalFish.githubpages-publish). Add the GitHub Pages Publish task under the agent job:
![Desktop View]({{ "/assets/img/posts/devops/jekyllRelease.png" | relative_url }})

That is. Happy Jekylling :)