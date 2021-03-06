Building and working with the debian packages
=============================================

[TOC]

Debian is one of the main supported operating systems in Aegir. See also the following instructions:

* [Install using Debian packages](/install/#debianubuntu)
* [Upgrade using Debian packages](/install/upgrade/#upgrades-with-debian-packages)

The following is aimed at developers wishing to maintain their own Debian
packages or work within the packaging framework.

Basic requirements
------------------

You need the following packages to build the Aegir Debian packages:

    apt-get install devscripts git-buildpackage

See also the section below on [Adding a new uploader](#adding-a-new-uploader).

Creating a Debian mini release
------------------------------

- Create a feature branch
 `git checkout -b 3.11.x 3.11.0`
- Commit or cherry-pick the desired fix
- Update the version number in provision.info
- Update to debian/changes ( specify 'testing' instead of unstable)
- Commit
- Set a tag
- Push to gitlab
- Wait for the [pipelines on GitLab.com](https://gitlab.com/aegir/provision/pipelines) to complete (especially the publish job)
- Test the packages using the [testing repository](http://debian.aegirproject.org/dists/testing/)
- Promote the packages to the stable repository.
On the repo server as the reprepro user do:
```
reprepro copy stable testing aegir3
reprepro copy stable testing aegir3-hostmaster
reprepro copy stable testing aegir3-provision
reprepro copy stable testing aegir3-cluster-slave
```

- Merge the feature branch into the main branch
- Broadcast? Mention in the irc/matrix room. Maybe Twitter, email.

Building a package for a new release
------------------------------------

Assuming we have just released 3.3, the following instructions will merge that
code into the `upstream` branch (which is used to create the Debian diff) and
then merged again in the debian branch (where the Debian code lives). We then
use `git-buildpackage` to build the package and tag it, then push those changes
back in the repository.

    cd provision
    git pull
    # if you previously ran release.sh, run:
    git reset --hard 7.x-3.4
    # otherwise run this next line:
    dch -v 3.3 -D unstable new upstream release
    git-buildpackage -kanarcat@koumbit.org
    dput aegir ../build-area/aegir3-provision_3.3_i386.changes

*Note*: Version numbers are slightly different in Debian - we use the "magic"
`~` separator to indicate that 3.0~alpha2 is actually lower than 3.0...

Packages are initially uploaded to the `unstable` repository for initial
test builds. The idea is that this final package can be moved to `testing` for
broader testing, using the command:

    sudo -u reprepro reprepro -b /srv/reprepro/ copy testing unstable aegir2 aegir2-provision aegir2-hostmaster aegir2-cluster-slave

When confirmed as ready, it is migrated to the `stable` repository, using the command:

    sudo -u reprepro reprepro -b /srv/reprepro/ copy stable testing aegir2 aegir2-provision aegir2-hostmaster aegir2-cluster-slave

Building a branch package
-------------------------

Sometimes you want to have a test package for a given branch without going
through a full release. This can be done by pushing the feature branch to GitLab.
You can then download the packages as build artefacts.


Installing packages manually
----------------------------

    dpkg -i aegir3-provision_3.4~rc3+g6632e6e-1_all.deb

We also make sure our custom makefile fetches the right one from provision:

    -includes[aegir] = "http://drupalcode.org/project/provision.git/blob_plain/7.x-3.4-rc3:/aegir.make"
    +includes[aegir] = "http://drupalcode.org/project/provision.git/blob_plain/7.x-3.x:/aegir.make"

Developing on Debian
--------------------

To develop third party extensions to Aegir on Debian, it is recommended to
install the Debian packages. If you are working on Aegir core, this could be a
bit trickier since the files are not where you expect them to be and are not
deployed as git repositories however.

You can, however, copy in place a .git directory using the following:

    git clone --branch=7.x-3.4-rc3 http://git.drupal.org/project/provision
    cp -Rp provision/.git /usr/share/drush/commands/provision/.git
    cd /usr/share/drush/commands/provision
    git stash

This will bring back a bunch of files that are removed from the Debian package,
so it will yield warnings on uninstall of the Debian package but it should
otherwise work.

You can do something similar with the frontend.

Package versioning
------------------

The stable repository should contain the latest release. The testing repository
will also contain the latest release (unless we're in the process of building a
release) but could have fixes to the Debian package that are being tested. The
unstable repository is automatically built from the stable branch and may be
broken.

To see what changes are done to the Debian package, see the
[debian/changelog](http://drupalcode.org/project/provision.git/blob/refs/heads/debian:/debian/changelog)
which is maintained on the [debian
branch](http://drupalcode.org/project/provision.git/shortlog/refs/heads/debian).
To see which version of the package is currently available in the repository,
you will unfortunately need to parse
the Packages file for
[unstable](http://debian.aegirproject.org/dists/unstable/main/binary-amd64/Packages),
[testing](http://debian.aegirproject.org/dists/testing/main/binary-amd64/Packages)
or
[stable](http://debian.aegirproject.org/dists/stable/main/binary-amd64/Packages).


Replacing an expired key
------------------------

    gpg --gen-key
    gpg --list-keys
    gpg --keyserver pgp.mit.edu --send-keys <key id>
    sudo -u reprepro -i
    gpg --search-keys <key id>
    gpg --fingerprint foo@bar.com ; gpg --check-sigs foo@bar.com # check if this is the real key
    echo allow * by key <key id> >> /srv/reprepro/conf/uploaders

This new key.asc should be placed on http://debian.aegirproject.org/key.asc and http://cgit.drupalcode.org/provision/tree/debian/key.asc

Extending the lifetime of a key
-------------------------------


    reprepro@zeus:~$ gpg --edit-key 3376CCF9

    gpg> expire
    Changing expiration time for the primary key.
    Please specify how long the key should be valid.
             0 = key does not expire
          <n>  = key expires in n days
          <n>w = key expires in n weeks
          <n>m = key expires in n months
          <n>y = key expires in n years
    Key is valid for? (0) 3y
    Key expires at Sat 12 Oct 2019 09:12:54 AM EDT
    Is this correct? (y/N) y
    gpg> save

    gpg --armor --export 3376CCF9 > key.asc

This new key.asc should be placed on http://debian.aegirproject.org/key.asc and http://cgit.drupalcode.org/provision/tree/debian/key.asc


How the archive was built
-------------------------

The following documentation was used: <https://wiki.koumbit.net/RepreproConfiguration>
