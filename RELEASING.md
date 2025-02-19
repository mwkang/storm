# Release

This document includes information about the Storm release process.

---

# Release Policy

Apache Storm follows the basic idea of [Semantic Versioning](https://semver.org/). Given a version number MAJOR.MINOR.PATCH, increment the:
 1. MAJOR version when you make incompatible API changes,
 2. MINOR version when you add functionality in a backwards compatible manner, and
 3. PATCH version when you make backwards compatible bug fixes.
 
# Release process

## Preparation

- We strongly encourage you to read the [Apache release signing page](http://www.apache.org/dev/release-signing.html), the [release distribution page](http://www.apache.org/dev/release-distribution.html#sigs-and-sums), as well as the [release publishing](http://www.apache.org/dev/release-publishing), [release policy](http://www.apache.org/legal/release-policy.html) and [Maven publishing](https://infra.apache.org/publishing-maven-artifacts.html) pages. ASF has common guidelines that apply to all projects.
- Ensure you can log in to http://repository.apache.org. You should use your Apache ID username and password.
- Install a SVN client, and ensure you can access the https://dist.apache.org/repos/dist/dev/storm/ and https://dist.apache.org/repos/dist/release/storm/ repositories. You should be able to access these with your Apache ID username and password.
- During the release phase, artifacts will be uploaded to https://repository.apache.org. This means Maven needs to know your LDAP credentials. It is recommended that you use Maven's mechanism for [password encryption](https://maven.apache.org/guides/mini/guide-encryption.html). Please configure this in your `${user.home}/.m2/settings.xml`:
  
```
    <settings>
  ...
    <servers>
      <!-- To publish a snapshot of your project -->
      <server>
        <id>apache.snapshots.https</id>
        <username> <!-- YOUR APACHE LDAP USERNAME --> </username>
        <password> <!-- YOUR APACHE LDAP PASSWORD (encrypted) --> </password>
      </server>
      <!-- To stage a release of your project -->
      <server>
        <id>apache.releases.https</id>
        <username> <!-- YOUR APACHE LDAP USERNAME --> </username>
        <password> <!-- YOUR APACHE LDAP PASSWORD (encrypted) --> </password>
      </server>
     ...
    </servers>
  </settings>
``` 
- Ensure you have a signed GPG key, and that the GPG key is listed in the Storm KEYS file at https://dist.apache.org/repos/dist/release/storm/KEYS. The key should be hooked into the Apache web of trust (https://keyserver.ubuntu.com for example)
   - set up the key as the default one to be used during the signing operations that the GPG Maven plugin will request (your OS GPG agent can do this)
- Compile environment:
  - some tests currently rely on the following packages being available locally:
    - NodeJS
    - Python3
  - some tests will require Docker to be running since they will create/launch containers (make sure `/var/run/docker.sock` has the correct permissions, otherwise you migh see the error `Could not find a valid Docker environment`)



If you are setting up a new MINOR version release, create a new branch based on `master` branch, e.g. `2.6.x-branch`. Then on master branch, set the version to a higher MINOR version (with SNAPSHOT), e.g. `mvn versions:set -DnewVersion=2.3.0-SNAPSHOT -P dist,rat,externals,examples`.
In this way, you create a new release line and then you can create PATCH version releases from it, e.g. `2.8.1`.

## Setting up a vote

1. Checkout to the branch to be released.

2. Run `mvn release:prepare -P dist,rat,externals,examples` followed `mvn release:perform -P dist,rat,externals,examples`. 
This will create all the artifacts that will eventually be available in maven central. This step may seem simple, 
but a lot can go wrong (mainly flaky tests). Note that this will create and push two commits with the commit message 
starting with "[maven-release-plugin]" and it will also create and publish a git tag, e.g. `v2.8.1`. Note: the full build can take up to 30 minutes to complete.

3. Once you get a successful maven release, a “staging repository” will be created at http://repository.apache.org 
in the “open” state, meaning it is still writable. You will need to close it, making it read-only. You can find more 
information on this step [here](https://infra.apache.org/publishing-maven-artifacts.html).

4. Checkout to the git tag that was published by Step 1 above, e.g. `git checkout tags/v2.8.1 -b v2.8.1`. 
Then build it with `mvn clean install -DskipTests`. Run `mvn package` for `storm-dist/binary` and `storm-dist/source` 
to create the actual distributions.

5. Generate checksums for the *.tar.gz and *.zip distribution files, e.g.
```bash
pushd storm-dist/source/target
sha512sum apache-storm-2.8.1-src.zip > apache-storm-2.8.1-src.zip.sha512
sha512sum apache-storm-2.8.1-src.tar.gz > apache-storm-2.8.1-src.tar.gz.sha512
popd

pushd storm-dist/binary/final-package/target
sha512sum apache-storm-2.8.1.zip > apache-storm-2.8.1.zip.sha512
sha512sum apache-storm-2.8.1.tar.gz > apache-storm-2.8.1.tar.gz.sha512
popd
```

6. Create a directory in the dist svn repo for the release candidate: https://dist.apache.org/repos/dist/dev/storm/apache-storm-x.x.x-rcx

7. Run `dev-tools/release_notes.py` for the release version, piping the output to a RELEASE_NOTES.html file. Move that file to the svn release directory, sign it, and generate checksums, e.g.
```bash
export GITHUB_TOKEN=MY_PERSONAL_ACCESS_TOKEN_FOR_GI
python3 dev-tools/release_notes.py <id-of-the-github-milestone> > RELEASE_NOTES.html
gpg --armor --output RELEASE_NOTES.html.asc --detach-sig RELEASE_NOTES.html
sha512sum RELEASE_NOTES.html > RELEASE_NOTES.html.sha512
```

To create a personal access token:

- Go to your GitHub account settings. 
- Navigate to **Developer Settings** > **Personal Access Tokens** > **Tokens (classic)**. 
- Generate a token with the `public_repo` scope.

To obtain the ID of a GitHub milestone:
- Visit the [milestone overview](https://github.com/apache/storm/milestones). 
- Click on the milestone you want to create release notes for. 
- Look at the URL in your browser. It will look like this: `https://github.com/apache/storm/milestone/40`, where the last number is the milestone ID.

8. Move the release files from steps 4,5 and 7 to the svn directory from Step 6. Add and commit the files. 
This makes them available in the Apache staging repo.

9. Start the VOTE thread. The vote should follow the [ASF voting process](https://www.apache.org/foundation/voting.html). 
Sample Template sent to dev@storm.apache.org

```
Subject: [VOTE] Release Apache Storm [VERSION]] (rcN)

Hi folks,

I have posted a [Nth] release candidate for the Apache Storm [VERSION] release and it is ready for testing.

The Nexus staging repository is here:
    https://repository.apache.org/content/repositories/orgapachestorm-[REPO_NUM]

Storm Source and Binary Release with sha512 signature files are here:
    https://dist.apache.org/repos/dist/dev/storm/apache-storm-[VERSION]-rcN/
The release artifacts are signed with the following key:
    https://keyserver.ubuntu.com/pks/lookup?op=index&fingerprint=on&search=[KEY]
    in this file https://www.apache.org/dist/storm/KEYS

The release was made from the Apache Storm [VERSION] tag at:
    https://github.com/apache/storm/tree/v[VERSION]

Full list of changes in this release: 
    https://dist.apache.org/repos/dist/dev/storm/apache-storm-[VERSION]-rcN/RELEASE_NOTES.html

To use it in a maven build set the version for Storm to [VERSION] and add the following URL to your settings.xml file:
https://repository.apache.org/content/repositories/orgapachestorm-[REPO_NUM]

The release was made using the Storm release process, documented on the GitHub repository:
https://github.com/apache/storm/blob/master/RELEASING.md

Please vote on releasing these packages as Apache Storm [VERSION]. The vote is open for at least the next 72 hours.
"How to vote" is described here: https://github.com/apache/storm/blob/master/RELEASING.md#how-to-vote-on-a-release-candidate
When voting, please list the actions taken to verify the release.

Only votes from the Storm PMC are binding, but everyone is welcome to check the release candidate and vote.
The vote passes if at least three binding +1 votes are cast.

[ ] +1 Release this package as Apache Storm [VERSION]
[ ]  0 No opinion
[ ] -1 Do not release this package because...

Thanks to everyone who contributed to this release.

Thanks!
[Release Manager Name]
```


## Releasing if the vote succeeds

0. Announce the results. Use the following template:

```agsl
Subject: [VOTE][RESULT] Storm [VERSION] Release Candidate [N]

Dear Community,

The voting for releasing Apache Storm [VERSION] Release Candidate [N] has passed with # +1 votes (# binding) and # +0 or -1 votes.
+1 votes:
* ###### / binding
* ############

Vote thread can be found at:
https://lists.apache.org/thread.html/##############

Thanks everyone for taking the time to review and vote for the release!
We will continue the rest of release process and send out the announcement email in the coming days.

Thanks to everyone who contributed to this release.

[RELEASE MANAGER NAME]
```

1. `svn mv https://dist.apache.org/repos/dist/dev/storm/apache-storm-x.x.x-rcx https://dist.apache.org/repos/dist/release/storm/apache-storm-x.x.x`. This will make the release artifacts available on dist.apache.org and the artifacts will start replicating to mirrors.

2. Go to http://repository.apache.org and release the staging repository

3. Wait at least 24 hrs. for the mirrors to catch up.

4. Check out the [storm-site](https://github.com/apache/storm-site) repository, and follow the README to generate release specific documentation for 
the site. Compose a new blog post announcement for the new release. Update the downloads page. Finally, commit and push 
the site as described in the storm-site README to publish the site.

5. Update `doap_Storm.rdf` with the new release version.

6. Announce the new release to dev@storm.apache.org, user@storm.apache.org, and announce@apache.org. You will need to use your @apache.org email to do this.

7. Delete any outdated releases from the https://dist.apache.org/repos/dist/release/storm/ repository. See [when to archive](http://www.apache.org/legal/release-policy.html#when-to-archive). 

8. Delete any outdated releases from the storm-site releases directory, and republish the site.

10. Create a release on [GitHub](https://github.com/apache/storm/releases). Generate the release notes with the GitHub tooling.

11. Post, promote, celebrate. ;) Annoucement email can be sent to announce@apache.org using the following template:

```agsl
Subject: [ANNOUNCE] Apache Storm [VERSION] Released

The Apache Storm community is pleased to announce the release of Apache
Storm version [VERSION].

Apache Storm is a distributed, fault-tolerant, and high-performance
realtime computation system that provides strong guarantees on the
processing of data. You can read more about Apache Storm on the project
website:

https://storm.apache.org/

Downloads of source and binary distributions are listed in our download
section:

https://storm.apache.org/downloads.html

You can read more about this release in the following blog post:

https://storm.apache.org/[YEAR]/[MONTH]/[DAY]/storm[VERSION]-released.html

Distribution artifacts are available in Maven Central at the following
coordinates:

groupId: org.apache.storm
artifactId: storm-{component}
version: [VERSION]

The full list of changes is available here [1]. Please let us know [2] if
you encounter any problems.

Regards,
The Apache Storm Team

[1] https://downloads.apache.org/storm/apache-storm-[VERSION]/RELEASE_NOTES.html
[2] https://github.com/apache/storm/issues
```

## Cleaning up if the vote fails

1. Sent email to dev@storm.apache.org 

2. Go to http://repository.apache.org and drop the staging repository.

3. Delete the staged distribution files from https://dist.apache.org/repos/dist/dev/storm/

4. Delete the git tag.

5. Send a [VOTE][CANCELED] message using the following format:

```agsl
Subject: [VOTE][CANCELED] Storm [VERSION] Release Candidate [N]

This release candidate Storm Release candidate [VERSION] rcN https://dist.apache.org/repos/dist/dev/storm/apache-storm-[VERSION]-rcN/ has been canceled.
New vote request will be sent out on RC[N+1] with further updates.

[RELEASE MANAGER NAME]
```

# How to vote on a release candidate

We encourage everyone to review and vote on a release candidate to make an Apache Storm release more reliable and trustworthy.

Below is a checklist that one could do to review a release candidate. 
Please note this list is not exhaustive and only includes some of the common steps. Feel free to add your own tests.

1. Verify files such as *.asc, *.sha512; some scripts are available under `dev-tools/rc` to help with it;
2. Build Apache Storm source code and run unit tests, create an Apache Storm distribution;
3. Set up a standalone cluster using apache-storm-xxx.zip, apache-storm-xxx.tar.gz, the Apache Storm distribution created from step 2, separately;
4. Launch WordCountTopology and ThroughputVsLatency topology and check logs, UI metrics, etc;
5. Test basic UI functionalities such as jstack, heap dump, deactivate, activate, rebalance, change log level, log search, kill topology;
6. Test basic CLI such as kill, list, deactivate, deactivate, rebalance, etc.

It's also preferable to set up a standalone secure Apache Storm cluster and test basic funcionalities on it.

Don't feel the pressure to do everything listed above. After you finish your review, reply to the corresponding email thread with your vote, summarize the work you have performed and elaborate the issues
you have found if any. Also please feel free to update the checklist if you think anything important is missing there. 

Your contribution is very much appreciated.  
