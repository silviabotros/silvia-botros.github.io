---
layout: post
title: Chef audit mode
date: '2017-05-07 20:49:11'
tags:
- infra_as_code
---

I have spoken before about how important it is for me and my team to make as many parts of the database stack match our larger infrastructure. One of the most crucial ways to do this is to make sure that not only are we deploying and managing databases using configuraton management, but that the cookbooks remain in lock step with changes our cookbooks at large move towards.

When i first learned chef, the way to test your chef cookbooks was [chef minitest](https://github.com/chef/minitest-chef-handler). But that is [now deprecated](https://github.com/chef/minitest-chef-handler/blob/master/README.md#deprecation-notice) and is no longer the recommended test method of chef cookbooks. So what is a DBA who is not always in tune with chef land to do? learn from her teammates of course! 😀

With the help of our ops engineering team members, I was directed at some examples in newer cookbooks we have and to the resources supported by InSpec which is what chef-audit is based on.

_So how does chef audit mode work?_

Chef audit works exactly like recipe code. In fact, you can write the audit code right inside the recipe it is auditing. It is very natural in its language which makes it very easy to write *before* the actual chef code (hint hint: TDD FTW!) and it has lots of [resource types](http://serverspec.org/resource_types.html) which make for simple, easy to read and maintain, tests.

_So if I have a cookbook that was using minitest, how can I make it move to chef audit?_

You need to decide where your audit code will live. Your options are:

* One gigantic audit recipe that is included in your runlist however you normally include recipes in the cookbook. This option will put all the code in one file but can get unwieldy and large as a cookbook gets more complex

* An `_audit_foo.rb` recipe for every `foo.rb` recipe you already have. The audit ones have to be separately included as well. This is arguably the most organized manner to do this and can work very well for books with lots of internal recipes where one large audit file can get very large. But it can also feel like recipe sprawl.

* Add the audit code directly inside each recipe. This is nice because you can then see in the same file both the code that makes the changes and the audits that run to validate the policies around these changes. But again, this can get harder to use if the individual recipe is long or complex

Each of these, as you can see, have pros and cons. The good thing about this flexibility is that you can pick what works best for your cookbook or organization :D

Now onto an example...or two...

Say you have this code block in a recipe to drop a script file in a specific spot and make it executable.
```ruby
cookbook_file '/usr/local/bin/test_backup.sh' do
  source 'test_backup.sh'
  owner 'mysql'
  group 'mysql'
  mode 0o755
  action :create
end
```

The code to audit this block would look like this
```ruby
control_group 'MySQL BackupTests : Archive access' do
  control 'test_backup.sh' do
      describe file('/usr/local/bin/test_backup.sh') do
        it { should exist }
        it { should be_file }
        it { should be_executable }
        it { should be_owned_by 'mysql' }
      end
    end
end
```

As you can see, the audit part is very natural in its language, and the resources are quite simple to use. So how do we tell chef to run this audit code in our test environment?

Presuming you use test kitchen, you need to edit that config to enable audit mode.
Under the provisioner section of your `.kitchen.yml` config file, add these 2 lines:

```
client_rb:
  audit_mode: :enabled
```

So what are the advantages of audit_mode vs the old minitests? well besides the fact that minitest is deprecated, I find that having the test code as part of the recipe tree (and better yet, can even be *in* the recipe directly) gives a very nice single view of what every recipe should look like and what my view of the state of the host should be based on that. That should come even handier for anyone trying to look at a cookbook I wrote and understand what the cookbook is supposed to do.

Aa I started a journey with chef [a few years back](https://dbsmasher.com/2015/02/12/-learning-configuration-management-as-a-dba/), converting my knowledge on how to build our databases into repeatable cookbooks, I will be spending the next months with the rest of our data ops team converting our resources into chef audit code.
