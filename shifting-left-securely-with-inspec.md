# Shifting Left Securely With inSpec

**sta·bil·i·ty**<br />
*stuh-bil-i-tee*


1. the state or quality of being stable.
1. firmness in position.
1. continuance without change; permanence.
1. resistance to change, especially sudden change or deterioration.
1. steadfastness; constancy, as of character or purpose.

Stability is something that we attempt to ensure by reducing the vectors that can make changes to a system. In practice, that means "deployments are executed by trusted individuals with admin access". 

The problem with the "separation of duties" approach to solve all your stablity issues is that everyone can make mistakes. A job title doesn't imply infallibilty. Every sysadmin who thinks that their title means they won't make a mistake will tell me a story over a couple of pints about when they totally messed up production.

It's also a huge burden on one group to be able to understand all the options and variance that steps in a "run book" will result in.

How do we usually do this? We might have a project that is 8 sprints long. And we don't think about security until the 8th sprint which is the "hardening" sprint.

Naturally, we fail all of this hardening, because we haven't been thinking about security all this time. Security and compliance need to be a consideration from the very beginning. We do this with tooling AND with culture. In this post, I'll be mostly focusing on how to leverage tooling to assist in supporting the cultural changes required. Specifically, I will be focusing on using inSpec to verify your infracode (in this case I'll be using Chef, but this will work just as well with other infracode tools).

## Shift left

What is "shifting left" anyway? The idea is that we move our testing further "to the left" in our pipeline. The closer to the introduction of the defect that it is discovered, the cheaper and easier it is to fix.

Of course, this introduces some challenges, right? If my tests don't pass, I can change the tests. We need to structure our systems to prevent this.

## How does this help with security?

Compliance and security are just another aspect of quality. It would sound ludicrous to only QA test our applications right before we deploy to production. It should sound just as ridiculous to wait until a "hardening sprint" to start thinking about security and compliance.

Another problem is that often times, these tests we do at the end are heavy. That is to say, they are figuratively or literally expensive; they require large resources, or expensive per-seat licenses, so we don't use them as often as we should. We need something better.

## Trying to prevent specific behaviors is a losing battle

* If you spend time keeping people from doing x, y, or z
* They will instead do a, b, or c to get the outcome they want
* Even worse, no communication happens. Insights are lost.

No matter how much you try to block the "how", you should focus on the "what". Consider the outcome, not the mechanism that lead to that outcome. 

## Perceived problem with distributed configuration management

When I suggest to sysadmins that developers should write Chef cookbooks, this is generally what they think is going to happen:

* Developer reads on Stack Overflow that disabling `selinux` will make his Node app work better
* Developer updates his cookbook to disable `selinux`
* Sysadmins get fired because of 3viL haxx0rz

## The better way

* Developer reads on Stack Overflow that disabling `selinux` will make his Node app work better
* Developer updates his cookbook to disable `selinux`
* Developer runs local tests which include compliance checks
* Compliance checks test for state of `selinux`
* Tests fail. Developer says "Welp, I guess I can't do that."

## What if the developers don't run those local tests?

The pipeline catches them.

They'll do better next time.

Even organizations that have a high-trust cultures, but still test everything. Take a page from Ronald Reagan - "Trust, but verify". I don't even trust *myself* to remember to test all the time. Remember, we enable local testing, but we *don't count on it*.

##  If you truly care about a thing, you care enough to write a test

Often times, the excuse given is "I don't have time to write a test for this thing". That's a cop-out, and I'm here to tell you that it won't fly. This goes back to the point of lightweight tools for compliance and testing - if it's too onerous to write these tests, nobody will do them. The good news is, it's not that hard. 

When you are writing these tests, think about this: we are testing for outcomes. Outcomes are what matter. Our pipeline is creating an artifact (in the case of infracode, this artifact is a converged node). What matters is the state of that outcome. *How* you got it there is not the question. We should be testing compliance and security against artifacts, and the outcome we are testing is "is this thing the way it should be, or is it a scary nightmare that should never see production"?

## Democrotize your testing

Remember when I talked about the hubris of sysadmins? Infosec folks do it too. Tools are kind of the least important thing to think about, but make sure it's not a tool that can only run tests from the security folks. If you care about compliance, move that stuff to the left. If your tool can't do that, it's time to find another tool.

That doesn't mean you need to throw out what you have, but consider adding something. The more that you can emulate whatever you care about testing in production into your pipeline, the happier you will be. Monitoring is just testing with a time dimension; this applies to compliance as well.

What you do *not* want to do is have a test in the pipeline that cannot be run by the local developer. Basically, if you do this, you're a a big meanie[1].

## Enter inSpec

InSpec is compliance as code. We write controls that test for the state of the system, and report back on this state. It's also pretty easy to read, as it's written in a spec-style langauge. Consider the difference between these two ways of checking the SSH protocol version on a system:

### Shell

``` sh
> grep "^Protocol" /etc/ssh/sshd_config | sed 's/Protocol //'
 2
 ```
 ### inSpec
```ruby
control 'ssh-1234' do
  impact 1.0
  title 'Server: Set protocol version to SSHv2'
  desc "
    Set the SSH protocol version to 2. Don't use legacy
    insecure SSHv1 connections anymore...
  "

  describe sshd_config do
   its('Protocol') { should eq 2 }
  end
end
```

As you can see, with inSpec, not only is if fairly human-readable, it provides *context*. We understand the impact of the control, and we can even add a nice description to explain what this is all about. The user experience[2] is better.

## Examples

(All code for this post can be found at [github.com/mattstratton/sysadvent-2017](https://www.github.com/mattstratton/sysadvent))

### Cookbook

We start by taking a look at a basic cookbook that creates a user and makes it own the `/var/log` directory (which we probably don't want to happen):

``` ruby
user 'sa2017-app' do
  comment 'SysAdvent User'
  system true
  shell '/bin/false'
end

directory '/var/log' do
  owner 'sa2017-app'
end
```

### Compliance profile

Our compliance profile for confirming that `/var/log` is owned by root looks like this:

```ruby
title 'Directory Tests'

control 'log-1.0' do                        
  impact 0.7                                
  title 'Ownership of /var/log'             
  desc 'The /var/log directory should be owned by root'
  describe file('/var/log') do
    it { should be_directory }
    it { should be_owned_by 'root' }
  end
end
```

### Tying it together

In this case, I've created a Jenkins pipeline that responds to changes on the cookbook's GitHub repo. The [Jenkinsfile](https://github.com/mattstratton/sa2017-app/blob/master/Jenkinsfile) defines the following stages:

1. Lint - Runs `foodcritic` and `cookstyle` against the cookbook
1. Smoke - Runs `kitchen test` using the tests in the cookbook itself
1. Compliance - Runs `kitchen test`, but includes our compliance profile, loaded for a `git` URL.

In order to accomplish the `Compliance` stage, we generate a new `kitchen.yml` file, which includes the path to the test we want to include. There are other ways to accomplish this, and it will depend upon how you build your pipelines. But hopefully this illustrates one way it can be done.

To prove the point, here's the output in Jenkins when the Compliance stage is run:

```
Profile: My Compliance (sa2017-compliance)
Version: 0.1.0
Target:  docker://4967508ea0b8f431b161edf561164ab8a49eba780b58ed85673f64c60b3bb8bd

[38;5;9m  ×  log-1.0: Ownership of /var/log (1 failed)[0m
[38;5;41m     ✔  File /var/log should be directory[0m
[38;5;9m     ×  File /var/log should be owned by "root"
     expected `File /var/log.owned_by?("root")` to return true, got false[0m

Profile Summary: 0 successful controls, [38;5;9m1 control failure[0m, 0 controls skipped
```




[1] - Sorry for the strong language.<br/>
[2] - And I bet you thought "user experience" was just for front-end developers.
