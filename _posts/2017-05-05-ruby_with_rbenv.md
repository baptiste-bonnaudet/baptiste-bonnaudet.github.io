---
layout: post
title:  "Some help with rbenv for clean Ruby environments"
date:   2017-07-05 11:57:05 -0400
categories: ruby rbenv workstation
---

Ruby environments... What a joke. I find ruby to be an elegant language, full of really cool idioms, awesome meta-programming features and I even think it's the most DevOpsy language with Golang. But I can't like it, I tried, seriously. All that because of the environment. I don't count the times I bricked my environment somehow. I tried RVM, rbenv, at work we even thought about having chef managing our workstation for rubies or embedding ruby environments in containers. Until I realized rbenv was the best solution when configured and used correctly.

* if you are working with chefdk, never use its embedded ruby for something else that chef kitchens and chef cookbooks!
* one `rbenv init` per repository and commit the damn `.ruby-version` so everyone use the same version of ruby
* use the gemset plugin extensively and do one `rbenv gemset init` per repository and commit the damn `.rbenv-gemsets` so if someone forgets to init its gemset it still exists.

## Here is an example of how to set a ruby environment

If not already done, install homebrew
```
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

### Install rbenv
```
brew install rbenv
rbenv init
```

### Load rbenv automatically
append the following to ~/.zshrc:
```
eval "$(rbenv init -)"
export PATH=~/.rbenv/bin:${PATH}
```

### Add plugin capabilities to rbenv
```
mkdir -p ~/.rbenv/plugins
git clone https://github.com/znz/rbenv-plug.git ~/.rbenv/plugins/rbenv-plug
```

### Install additional rbenv plugins
```
# rbenv-binstubs plugin makes rbenv transparently aware of project-specific binstubs created by bundler.
rbenv plug https://github.com/ianheggie/rbenv-binstubs.git

# rbenv-chefdk plugin lets you treat ChefDK's embedded ruby as another version in rbenv.
rbenv plug https://github.com/docwhat/rbenv-chefdk.git

# rbenv-gemset plugin allows you to use "gemsets", sandboxed collections of gems. This lets you have multiple collections of gems installed in different sandboxes, and specify (on a per-application basis) which sets of gems should be used.
rbenv plug https://github.com/jf/rbenv-gemset.git

# rbenv-vars plugin lets you set global and project-specific environment variables before spawning Ruby processes.
rbenv plug https://github.com/rbenv/rbenv-vars.git

# ruby-build plugin provides an rbenv install command to compile and install different versions of Ruby
rbenv plug https://github.com/rbenv/ruby-build.git
```

### Install ruby, make it the default
```
rbenv install 2.4.0
rbenv global 2.4.0
rbenv shell 2.4.0
```
### Install desired gems into 2.1.6
```
gem install bundler
gem install pry
```
