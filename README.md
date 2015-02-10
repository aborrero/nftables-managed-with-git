# nftables ruleset managed with git

In this article I will be mixing some ideas and concepts both of my own and I've seen in the web:
 * git hooks to check content of new commit
 * linux net namespaces to check if ruleset could load
 * atomically replace a nftables ruleset

Mainly, getting nftables hooked in git is matter of combing these elements:
 * a sudo security policy
 * a git config and workflow to handle the checks and deployments
 * a nftables workflow and ruleset architecture

## What we want to achieve

We have a team of sysadmins who are working together to maintain a big ruleset (thousand of rules). Keeping track of changes is almost mandatory, and git does this thing very well.

Also, we want to prevent some human mistakes in terms of bad nftables syntax. In the devel world, this is equivalent to prevent developers to push code which fails to build from source. A basic QA.

If all is OK, replace the ruleset in an atomic way.

## Files layout

I assume this dirs scheme:
 * the nftables ruleset is at `/etc/nftables.d` (don't touch this dir by hand!)
 * git repo in bare mode is at `/srv/git/nft-firewall.git`
 * sudoers policy is at `/etc/sudoers.d/nft-git`

## The git part

Create a git repo in bare mode somewhere, for example `/srv/git/nft-firewall.git` (with `git init --bare`).

Put update and post-receive hook scripts under `/srv/git/nft-firewall.git/hooks/` with proper execs permissions.

The update hook will do the "QA" part when the user push changes from their local repos:

 1. Create a temp dir under /tmp
 2. Export the ruleset there (the new ruleset isn't in the filesystem yet, that's why git archive is used)
 3. Create a linux netnamespace
 4. Load the ruleset in the new network ns
 5. If the ruleset fails to load, reject the commit.
 6. If the ruleset loads, all is OK, do some cleanups and exit.
 7. The post-receive hook will run after the update hook, and takes care of the effective deployment of the just-changed ruleset:
 8. Checkout the git repo to /etc/nftables.d (this is, actually deploy the nftables config to the filesystem).
 9. Clean untracked files and dirs at /etc/nftables.d (don't touch this dir by hand!)
 10. Load the new ruleset.

## The nftables layout

nftables comes with a set of handy options which will permit you to organize the ruleset (thus, the firewall) in very flexible ways.

In this example, I use this layout:
* `/etc/nftables.d/ruleset.nft` (the main file)
* `/etc/nftables.d/inet-filter-chain-input.nft` (filter rules in the input chain for the inet family)
* `/etc/nftables.d/inet-filter-chain-forward.nft` (filter rules in the forward chain for the inet family)
* `/etc/nftables.d/inet-filter-chain-output.nft` (filter rules in the output chain for the inet family)
* `/etc/nftables.d/inet-filter-sets.nft` (data sets for the inet filter table)
 
So, we will be 'including' all the files from `ruleset.nft`, plus flushing the previous ruleset.  
In the other files, we only define each chain and rules. In the set file we define sets which are globals to be used by all rules in the inet filter table.  
The nftables ruleset is meant to be loaded with 'nft -f ruleset.nft' which will perform an atomic replacement of the ruleset.

## The sudo policy

Before any other step, be sure you are following the security policy of your organization regarding this.  
In order to allow all these operations with a unprivileged users (which BTW is modifying the ruleset via a remote git push), we need a concrete sudo policy.

Creating netnamespaces and modifying the nftables ruleset is a privileged operation.  
My recomendation at this point is a sudoers file like the one you can find at the github repo.  
The `NOPASSWD` option is required for sudo to don't ask user password while in the git hooks.

You should take care also of standard unix users/groups layout. Give appropriate permissions to working dirs/scripts. For example, put your operators in a group `git`, give the git bare repo group `git` and give g+w.

## The workflow

A sys-admin team member could end in a workflow like this:
1. Clone the nft-firewall.git repo
2. Make changes to the ruleset (add a rule, replace other... whatever)
3. Run some local tests (sure)
4. Commit on your local repo
5. Push to the git server
6. The git server will run the hook scripts, testing and deploying the new ruleset
7. Other sysadmin clones or pull the repo and go to step 2.

## More info

This was inspired by this blogpost by [Vincent Bernat]([http://vincent.bernat.im/en/blog/2014-netfilter-firewall-script.html).  
Check the [nftables wiki](http://wiki.nftables.org).  
Also, the official docs about [git hooks](http://git-scm.com/book/en/v2/Customizing-Git-Git-Hooks).
