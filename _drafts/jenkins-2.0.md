---
layout: post
title:  "Starting fresh with Jenkins 2.0"
date: 2016-04-27 18:00:00 +0100
categories: 
 - Docker
 - Jenkins
 - Jenkinsfile
tags:
 - Docker
 - Jenkins
 - Jenkinsfile
commentIssueId: 3
---
**Now Jenkins 2 recently got released it was time for me to start using it. Just changing the tag number of the Docker image was not enough. 
In this post I will describe what you need to do to get started.**

{::options parse_block_html="true" /}
<div class="toc">
<h4>Contents</h4>

* TOC
{:toc}

</div>


## Tooling

- Docker
- Text editor

## Jenkins 2
In Jenkins 2 one of the main new features is "Pipeline as code". Your pipeline/build steps are now bundled with your code in your git/svn repo. 
To have the buildsteps defined as code has some advantages, but for me the biggest one is as your project evolves your build steps will probably evolve as well. So when for some reason you need to go back in time (perhaps to fix a production issue) you will build with the steps that are required for that moment in time.
The can configure the buildsteps in a file called "Jenkinsfile". 

In the example below I have added some of the items I use a lot myself. 
The language in the script is Jenkins DSL, which is almost identical to groovy.

## Dockerfile
As most Docker projects you need a Dockerfile. We are using the official [Jenkins](https://hub.docker.com/_/jenkins/) Dockerimage as a base (line 1).
Jenkins relies heavily on plugins. To add plugins we use a [file](/images/plugins.txt) with all the required plugins. With the plugins file I use, Jenkins still wants to get some more... So don't be alarmed when you see a message of this starting it the first time. To install the plugins during the build of the Docker image we copy the plugins.txt into the image (line 3) and use it as a source when executing the plugin installer (line 4).
{% highlight ruby linenos %}
FROM jenkinsci/jenkins:2.7
#plugins
COPY plugins.txt /usr/share/jenkins/plugins.txt
RUN /usr/local/bin/plugins.sh /usr/share/jenkins/plugins.txt

#config
COPY jobs/docker-webapp.xml /usr/share/jenkins/ref/jobs/docker-webapp/config.xml
COPY config.xml /usr/share/jenkins/ref/config.xml

USER root
#Adding Jenkins user to the correct groups
RUN usermod -a -G users jenkins
RUN usermod -a -G 100 jenkins

USER jenkins
{% endhighlight %}

## Jenkinsfile
The Jenkinsfile in here is just an example of what you can do. It is not really a working script as I would never use scripts called somescript.sh ;-) .
{% highlight groovy linenos %}
node('master') {
  
  stage "prepare"
  checkout scm

  stage "build"
  node('docker-buildslave-centos7') {
    withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'foo-credenitals', passwordVariable: 'FOO_PASS', usernameVariable: 'FOO_USER']]) {
      sh "sed -i 's/CHANGEUSER/${env.FOO_USER}/g' somescript.sh"
      sh "sed -i 's/CHANGEPASS/${env.FOO_PASS}/g' somescript.sh"
    }
    sh "chmod +x somescript.sh"
    sh "./somescript.sh"
    stash includes: '*.bin', name: 'bins', useDefaultExcludes: false
  }

  stage "test"
  node('docker-test-centos7') {
    checkout scm
    unstash bins
    sh "./testscript.sh"
  }

  stage "store"
  unstash bins
  # Upload somewhere

}

{% endhighlight %}
