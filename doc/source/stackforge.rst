:title: StackForge

.. _stackforge:

StackForge
##########

StackForge is the way that OpenStack related projects can consume and
make use of the OpenStack project infrastructure. This includes Gerrit
code review, Jenkins continuous integration, GitHub repository
mirroring, and various small things like IRC bots, pypi uploads, RTFD
updates. Projects should make use of StackForge if they want to run
their project with Gerrit code review and have a trunk gated by Jenkins.

StackForge projects are expected to be self sufficient when it comes to
configuring Gerrit/Jenkins/Zuul etc. The openstack-infra team can
provide assistance as resources allow, but should not be relied on.

What StackForge is not:

* Official endorsement of a project by OpenStack.
* Access to a GitHub organization (StackForge projects are mirrored to
  GitHub, this is all the GitHub org is used for).
* A guarantee of eventual OpenStack incubation (Though it is a good
  first step in that process as it exposes the project to the OpenStack
  way of doing things).

Audience
********

The focus of StackForge is to provide a place for OpenStack contributors
to maintain related unofficial projects using the same tools and
procedures as they employ when working on official OpenStack projects,
to make it easier for other OpenStack developers to contribute effort to
those projects and in some cases to ease a project's path to incubation
and official integration. As such, the target audience for this document
is current OpenStack developers who are assumed to already be familiar
with how changes are uploaded and reviewed within OpenStack projects. As
an introduction to OpenStack contribution, it is recommend to first read
https://wiki.openstack.org/wiki/How_To_Contribute and in particular the
https://wiki.openstack.org/wiki/Gerrit_Workflow article linked from it.

Add a Project to StackForge
***************************

Create a new StackForge Project with Puppet
===========================================

OpenStack uses Puppet and a management script to create Gerrit projects
with simple changes to the openstack-infra/config repository. To start make
sure you have cloned the openstack-infra/config repository
``git clone https://git.openstack.org/openstack-infra/config``.

First you need to add your StackForge project to the master project list.
Edit ``modules/openstack_project/files/review.projects.yaml`` and add a
new section for your project in alphabetical order within the file.
It should look something like::

  - project: stackforge/project-name
    description: Latest and greatest cloud stuff.
    upstream: git://github.com/awesumsauce/project-name.git

The description will set the project description on the GitHub
StackForge mirror, and the upstream should point at an existing
repository which can be used to preseed Gerrit with an initial commit
history. Both of these are optional. Note that the current tools
assume that the upstream repo will have a master branch.

Note: Ensure the source repo has been evaluated and only required branches
and tags remain when it seeds the stackforge repo. Cleaning up a repo of
unnecessary branches and tags after the merge requires an openstack-infra
core member to do so.

The next step is to add a Gerrit ACL config file. Edit
``modules/openstack_project/files/gerrit/acls/stackforge/project-name.config``
and make it look like::

  [access "refs/heads/*"]
  abandon = group project-name-core
  label-Code-Review = -2..+2 group project-name-core
  label-Workflow = -1..+1 group project-name-core

  [access "refs/tags/*"]
  pushSignedTag = group project-name-release

  [receive]
  requireChangeId = true
  requireContributorAgreement = true

  [submit]
  mergeContent = true

The access sections in the example ACL grant the project's core group
approval privileges and the ability so set/un-set Workflow status on
changes, as well as the ability to push tags. The other sections set
some required options for Gerrit to function normally (enforcing
presence of a Change-Id in commits and allowing changes to be merged).
This example also expects contributors to agree to a standard
OpenStack CLA, join the OpenStack Foundation and submit contact
information (this feature can be disabled by setting
requireContributorAgreement to false).

That is all that is necessary to add a StackForge project to Gerrit;
however, this project isn't very useful until we setup Jenkins jobs for
it and configure Zuul to run those jobs. Continue reading to configure
these additional tools.

Add Jenkins Jobs to StackForge Projects
=======================================

In the same openstack-infra/config repository (and in the same change
if you like) we need to edit additional files to setup Jenkins jobs
and Zuul for the new StackForge project.

If you are interested in using the standard python Jenkins jobs (docs,
pep8, python 2.6 and 2.7 unittests, and coverage), edit
``modules/openstack_project/files/jenkins_job_builder/config/projects.yaml``
and add a new section for your project in alphabetical order in the file. It
should look something like::

  - project:
      name: project-name
      node: bare-trusty
      tarball-site: tarballs.openstack.org

      jobs:
        - python-jobs

List of jobs included to the ``python-jobs`` jobs group is located in
``modules/openstack_project/files/jenkins_job_builder/config/python-jobs.yaml``.
For document publication there's also a publisher job template for the
popular `Read the Docs`_ documentation hosting service, which can be
used by adding the ``hook-{name}-rtfd`` template to the jobs list::

  - project:
      name: project-name
      node: bare-trusty
      tarball-site: tarballs.openstack.org

      jobs:
        - python-jobs
        - hook-{name}-rtfd

.. _Read the Docs: https://readthedocs.org/

If you aren't ready to run any gate tests or other project-specific
jobs yet, you don't need to edit ``projects.yaml``.

Now that we have Jenkins jobs we need to tell Zuul to run them when
appropriate. Edit
``modules/openstack_project/files/zuul/layout.yaml``
and add a new section for your project in alphabetical order within the file.
It should look something like::

  - name: stackforge/project-name
    template:
      - name: merge-check
    check:
      - gate-project-name-docs
      - gate-project-name-pep8
      - gate-project-name-python26
      - gate-project-name-python27
      - gate-project-name-python33
    gate:
      - gate-project-name-docs
      - gate-project-name-pep8
      - gate-project-name-python26
      - gate-project-name-python27
      - gate-project-name-python33
    post:
      - project-name-coverage

If you aren't ready to run any gate tests yet and did not configure
python-jobs in projects.yaml, it should look like this instead::

  - name: stackforge/project-name
    template:
      - name: merge-check
      - name: noop-jobs

That concludes the bare minimum openstack-infra/config changes necessary to
add a project to StackForge. You can commit these changes and submit
them to review.openstack.org at this point, or you can wait a little
longer and add your project to GerritBot first.

Request an Initial Gerrit Core Group Member
===========================================

StackForge uses Gerrit for group management. After the change to create
your StackForge project has merged, request an initial member for the
Gerrit group configured in your ACL (probably something like
``your-project-name-core``). Members of this team will have permissions
to approve code changes to your project as defined in your ACL, and to
add other Gerrit users to the group.

You can request an initial Gerrit group member by opening a bug at
https://bugs.launchpad.net/openstack-ci/+filebug (make sure to mention
the Gerrit full name or E-mail address of your initial member). See
https://wiki.openstack.org/wiki/Project_Group_Management for details on
project group management.

Configure StackForge Project to use GerritBot
=============================================

To have GerritBot send Gerrit events for your project to a Freenode IRC
channel edit
``modules/gerritbot/files/gerritbot_channel_config.yaml``.
If you want to configure GerritBot to leave alerts in a channel
GerritBot has always joined just add your project to the project list
for that channel::

  stackforge-dev:
      events:
        - patchset-created
        - change-merged
        - x-vrif-minus-2
      projects:
        - stackforge/foo
        - stackforge/python-fooclient
        - stackforge/project-name
      branches:
        - master

If you want to join GerritBot to a new channel add a new section to the
end of this file that looks like::

  project-name-dev:
      events:
        - patchset-created
        - change-merged
        - x-vrif-minus-2
      projects:
        - stackforge/project-name
      branches:
        - master

If you are defining a new channel, add it also in
``modules/openstack_project/files/accessbot/channels.yaml`` file, optionally
defining also its mask.
The mask will be used to define the access level for IRC users who are not
listed in that file in the ``global`` section or otherwise listed for the
channel.

For instance:

  - name: new_project
    mask: full_mask

For more information about channel requirements and IRC services provided by
the infrastructure team, visit :ref:`irc`

And that's it. At this point you will want to submit these edits as a
change to review.openstack.org. When you do so, please use the
``new-project`` topic.  You can do that using the ``-t`` option to
``git review``.

  $ git review -t new-project

Add .gitreview file to project
==============================

If the new project you have added has a specified upstream you will need
to add a ``.gitreview`` file to the project once it has been created. This
new file will allow you to use ``git review``.

The basic process is clone from stackforge, add file, push to Gerrit,
review and approve.::

  git clone https://git.openstack.org/stackforge/project-name
  cd project-name
  git checkout -b add-gitreview
  cat > .gitreview <<EOF
  [gerrit]
  host=review.openstack.org
  port=29418
  project=stackforge/project-name.git
  EOF
  git review -s
  git add .gitreview
  git commit -m 'Add .gitreview file.'
  git review
