# How To Release Apache Submarine
This document is for Apache Submarine committer and PMCs to do a new release.

## Preparation
If you have not already done so, generate your PGP key and append your [signing key](http://www.apache.org/dev/release-signing.html#keys-policy) to the [KEYS](https://dist.apache.org/repos/dist/release/submarine/KEYS) file. Once you commit your changes (ask for PMC's help if you cannot), they will automatically be propagated to the website. Also upload your key to a public key server if you haven't.
End users use the KEYS file to validate that releases were done by an Apache committer.

```
# generate key and upload to the public
gpg --gen-key
gpg --list-sigs <your name>
gpg --keyserver pgp.mit.edu --send-key <your publish key RSA string like B3097AC in the above cmd output>
gpg --refresh-keys --keyserver pgp.mit.edu
# adding your key to submarine KEYS
gpg --list-sigs <your name> >> ~/key.txt
gpg --armor --export <your name> >> ~/key.txt
svn co --depth immediates https://dist.apache.org/repos/dist apache-dist
cd apache-dist
svn update --set-depth infinity release/submarine
cd release/submarine
echo ~/key.txt >> KEYS
svn add KEYS
svn ci -m "Add <your name>'s key"
```

## Release Plan
It's better to send a release plan email informing code freeze date and release date.

## Clean up the JIRA
Bulk update JIRA to unassign from this release all issues that are open non-blockers.
Assuming we're releasing version X, use below advanced filter in [submarine issue page](https://issues.apache.org/jira/projects/SUBMARINE). For instance, if we're releasing 0.3.0.
```
project in ("Apache Submarine") AND  "Target Version" = 0.3.0 AND statusCategory != Done
```
Click "tools"-> "bulk update" to edit all issues:
1. Change the target version to X+1. Here it is 0.4.0
2. Add a comment to inform contributors. Like this "Bulk update due to releasing 0.3.0. Please change it back if you think this is a blocker."

Do a double-check to confirm that there's no issues found with the above filter. And send mail to the developer list informing that we should mark "Target version" to 0.4.0 when creating new JIRAs.

## Tagging
Once the JIRA is cleaned up, we can tag the candidate release with below steps:
```
export version=0.3.0
export cversion=0.3.0-RC0
export tag=release-$cversion
git tag -s release-$version -m "Release candidate - $version"
# Verify the tag is signed with your GPG key
git tag -v release-$cversion
# Push the tag to upstream
git push origin release-$cversion
```

## Build Artifacts

The submarine artifacts consists of GPG signed source code tarball, binary tarball and docker images.
```
cd submarine/dev-support/cicd/
./create_release.sh $version $tag
#Move the artifacts to a folder instead of /tmp/
mv /tmp/submarine-release ~/
```

## Upload Artifacts For Vote
Before the uploading, we need to do some basic testing for the release candidates. For instance, build from the source tarball and run feature test using the binary tarball.

### Staging Source and Binary Tarball to self FTP server
```
cd ~/submarine-release
sftp home.apache.org
cd public_html
mkdir submarine-0.3.0-RC0
put -r .
exit
```
### Staging Docker Images
#### Package An Existing Release Candidates
When doing release, the release manager might needs to package an artifact candidates in this docker image and public the image candidate for a vote.
In this scenario, we can do this:

Put submarine candidate artifacts to a folder like "~/releases/submarine-release"
```
$ ls $release_candidates_path
submarine-dist-0.3.0-hadoop-2.9.tar.gz        submarine-dist-0.3.0-src.tar.gz.asc
submarine-dist-0.3.0-hadoop-2.9.tar.gz.asc    submarine-dist-0.3.0-src.tar.gz.sha512
submarine-dist-0.3.0-hadoop-2.9.tar.gz.sha512 submarine-dist-0.3.0-src.tar.gz
```
```
export submarine_version=0.3.0
export release_candidates_path=~/releases/submarine-release
./build_mini-submarine.sh
#docker run -it -h submarine-dev --net=bridge --privileged -P local/mini-submarine:0.3.0 /bin/bash
docker tag local/mini-submarine:0.3.0 apache/submarine:mini-0.3.0-RC0
```
In the container, we can verify that the submarine jar version is the expected 0.3.0. Then we can upload this image with a "RC" tag for a vote.
```
docker push apache/mini-submarine:0.3.0:RC0
```
### Publish Jars To Apache Maven Staging Repository
```
export GPG_PASSPHRASE=yourPassphase
export ASF_USERID=yourApacheId
export ASF_PASSWORD=yourApachePwd
./publish_release.sh $version $tag
```
Then to view the staging repo, we can login the the https://repository.apache.org with the apache id. Click the "Staging Repositories" in the left side of the web page. And click "orgapachesubmarine-1001", then you will see the details of the repo including the URI. The URI is like this:
"https://repository.apache.org/content/repositories/orgapachesubmarine-1001"

### Call A Vote For The Release Candidate
After the artifacts and images are staged, we can send a vote email to the community. It's recommended that we paste the URI of RC tag, RC release artifacts, docker images, maven staging repository and the KEYS.
Refer to [here](https://www.mail-archive.com/dev@submarine.apache.org/msg01498.html) for an example.

## Release
In several days if the [vote passes](http://hadoop.apache.org/bylaws#Decision+Making), we can publish the release.
If the vote fails, then we need to start another RC from tagging to the staging.

1. Access [submarine project version page](https://issues.apache.org/jira/projects/SUBMARINE?selectedItem=com.atlassian.jira.jira-projects-plugin:release-page&status=unreleased). Click the version to be released, and then click the "Release" button. It will request the release date. We can fill it with the end-of-vote date.

2. Tag the release.
```
export version=0.3.0
git tag -s rel/release-${version} -m "Submarine ${version} release"
git push origin rel/release-${version}
```

3. Copy release artifacts to apache dist server
```
svn co --depth immediates https://dist.apache.org/repos/dist apache-dist
cd apache-dist
svn update --set-depth infinity dev/submarine
svn update --set-depth infinity release/submarine
cd dev/submarine
mkdir submarine-$version
cp ~/submarine-release/*  submarine-$version/
svn add submarine-$version
svn ci -m "Publishing the bits for submarine release ${version} to apache dist dev folder"
cd ../../
svn mv dev/submarine/submarine-$version release/submarine/
svn ci -m "Publishing the bits for submarine release ${version}"
```

Usually binary tarball becomes larger than 300MB, so it cannot be directly uploaded to the distribution directory. We can use the dev directory (https://dist.apache.org/repos/dist/dev/submarine/) first and then move it to the distribution directory by svn move

4. In [Nexus](https://repository.apache.org/), effect the release of artifacts by selecting the staged repository and then clicking Release. If there were multiple RCs, simply drop the staging repositories corresponding to failed RCs.

5. Upload the images
```
docker tag apache/submarine:mini-0.3.0-RC0 apache/submarine:mini-0.3.0
docker push apache/submarine:mini-0.3.0
```

6. Update the version in pom.xml
```
# if the new version a a point release
mvn versions:set -DgenerateBackupPoms=false -DnewVersion=X.(Y+1).Z-SNAPSHOT
git commit -a -m "Preparing for X.(Y+1).Z development"
```

7. Wait 24 hours for release to propagate to mirrors.

8. Update the website. The guide is [here](https://github.com/apache/submarine-site)
```
git clone https://github.com/apache/submarine-site.git
cd submarine-site
git checkout master
# Edit download.md to add the new release content
# The url of the binary and source tarball should be like
# "https://www.apache.org/dyn/closer.cgi/submarine/submarine-0.3.0/submarine-dist-0.3.0-hadoop-2.9.tar.gz"
# due to it will find a mirror server for different region.
# The url of the signature and checksum should be like
# "https://www.apache.org/dist/submarine/submarine-0.3.0/submarine-dist-0.3.0-hadoop-2.9.tar.gz.sha512)"
# due to they're not mirrored from apache dist server.
# For the release note web page, we need to create a MD file under "releases" directory
# like "submarine-release-0.3.0.md". The count of issues can be found
# from JIRA like "https://issues.apache.org/jira/projects/SUBMARINE/versions/12345556".
# And the details of issues can also get from "Release Notes" in that page.
cp release/submarine-release-0.3.0.md release/submarine-release-<new version>.md
vim release/submarine-release-<new version>.md
cd ..
docker run -it -p 4000:4000 -v $PWD/submarine-site:/submarine-site hadoopsubmarine/submarine-website:1.0.0 bash
cd /submarine-site
bundle exec jekyll serve --watch --host=0.0.0.0
# Open another terminal, you can edit MD files and refresh the webpage to see changes instantly.
```
9. Send announcements to the user and developer lists once the site changes are visible.
