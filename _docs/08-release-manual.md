---
title: "Release Manual"
permalink: /docs/release-manual
excerpt: "Apache RocketMQ Release Manual"
modified: 2017-02-7T15:01:43-04:00
---

{% include toc %}

This is a guide to making a release of Apache RocketMQ (incubating). Please follow the steps below.

## Preliminaries
### Apache Release Documentation
There are some release documentations provided by The ASF, including the incubator release documentation, can be found here:

* [Apache Release Guide](http://www.apache.org/dev/release-publishing)
* [Apache Release Policy](http://www.apache.org/dev/release.html)
* [Apache Incubator Release Guidelines](http://incubator.apache.org/guides/releasemanagement.html)
* [Apache Incubator Release Policy](http://incubator.apache.org/incubation/Incubation_Policy.html#Releases)
* [Maven Release Info](http://www.apache.org/dev/publishing-maven-artifacts.html)

### Code Signing Key
Create a code signing gpg key for release signing, use **\<your Apache ID\>@apache.org** as your primary ID for the code signing key. See the [Apache Release Signing documentation](https://www.apache.org/dev/release-signing) for more details.

* Create new pgp key. How to use pgp please refer to [here](http://www.apache.org/dev/openpgp.html).
* Generate a new key via `gpg --gen-key`, and answer 4096 bits with no expiration time.
* Upload your key to a public key server, like `gpg --keyserver pgpkeys.mit.edu --send-key <your key id>`.
* Get the key signed by other committers(Optional).
* Add the key to the RocketMQ [KEYS file](https://dist.apache.org/repos/dist/dev/incubator/rocketmq/KEYS).

**Tips:** If you have more than one key in your gpg, set the code signing key to `~/.gnupg/gpg.conf` as default key is recommended.
 
### Prepare Your Maven Settings
Make sure that your Maven settings.xml file contains the following:

```xml
<settings>
   <profiles>
         <profile>
           <id>signed_release</id>
           <properties>
               <mavenExecutorId>forked-path</mavenExecutorId>
               <gpg.keyname>yourKeyName</gpg.keyname>
               <username>yourApacheID</username>
               <deploy.url>https://dist.apache.org/repos/dist/dev/incubator/rocketmq/</deploy.url>
           </properties>
       </profile>
 </profiles>
  <servers>
    <!-- To publish a snapshot of some part of Maven -->
    <server>
      <id>apache.snapshots.https</id>
      <username>yourApacheID</username>
      <password>yourApachePassword</password>
    </server>
    <!-- To stage a release of some part of Maven -->
    <server>
      <id>apache.releases.https</id>
      <username>yourApacheID</username>
      <password>yourApachePassword</password>
    </server>
    <server>
      <id>gpg.passphrase</id>
      <passphrase>yourKeyPassword</passphrase>
    </server>
  </servers>
</settings>
```

**Tips:** It is highly recommended to use [Maven's password encryption capabilities](http://maven.apache.org/guides/mini/guide-encryption.html) for your passwords.

### Cleanup JIRA issues
Cleanup JIRA issues related to this release version, and check all the issues has been marked with right version in the `FixVersion` field.

### Publish the Release Notes
Generate the release notes via [RocketMQ JIRA](https://issues.apache.org/jira/browse/ROCKETMQ/) and publish it to the [rocketmq-site](https://github.com/apache/incubator-rocketmq-site), there is a [release notes](http://rocketmq.incubator.apache.org/release_notes/release-notes-4.0.0-incubating/) of `4.0.0-incubating` available for reference, include the link to the release notes in the voting emails.

## Build the Release Candidate
Firstly, checkout a new branch from `master` with its name equal to the release version, like `release-4.0.0-incubating`.

### Build the Candidate Release Artifacts
Before building the release artifacts, do some verifications below:

* Ensure that now your are in the candidate release branch.
* Ensure that all the unit tests can pass via `mvn clean install`.
* Ensure that all the integration tests can pass via `mvn clean test -Pit-test`.

Perform the following to generate and stage the artifacts:

1. `mvn clean release:clean`
2. `mvn release:prepare -Psigned_release -Darguments="-DskipTests"`, answer the right release version, SCM release tag, and the new development version.
3. `mvn -Psigned_release release:perform -Darguments="-DskipTests"`, generate the artifacts and push them to the [Nexus repo](https://repository.apache.org/#stagingRepositories). If you would like to perform a dry run first (without pushing the artifacts to the repo), add the arg -DdryRun=true

Now, the candidate release artifacts can be found in the [Nexus staging repo](https://repository.apache.org/#stagingRepositories) and in the `target` folder of your local branch.

**Tips:** If you are performing a source-only release, please remove all artifacts from the staging repo except for the .zip file containing the source and the javadocs jar file. In the Nexus GUI, you can right click on each artifact to be deleted and then select `Delete`.

### Validate the Release Candidate
Now the release candidate is ready, before calling a vote, the artifacts must satisfy the following checklist:

* Checksums and PGP signatures are valid.
* Build is successful including unit and integration tests.
* DISCLAIMER is correct, filenames include “incubating”.
* LICENSE and NOTICE files are correct and dependency licenses are acceptable.
* All source files have license headers where appropriate, RAT checks pass
* Javadocs have been generated correctly and are accurate.
* The provenance of all source files is clear (ASF or software grants).

Please follow the steps below to verify the checksums and PGP signatures:

1. Download the release artifacts, PGP signature file, MD5/SHA hash files.
2. On unix platforms the following command can be executed:
  
  ```shell
  for file in `find . -type f -iname '*.asc'`
  do
      gpg --verify ${file} 
  done
  ```
  
  or
  
  ```shell
  gpg --verify rocketmq-all-%version-number%-incubating-source-release.zip.asc rocketmq-all-%version-number%-incubating-source-release.zip
  ```
  Check the output to ensure it contains only good signatures:
  
  ```text
  gpg: Good signature from ... gpg: Signature made ...
  ```

3. Compare MD5, SHA hash generated from the below command with the downloaded hash files.

  ```shell
  gpg --print-mds rocketmq-all-%version-number%-incubating-source-release.zip 
  ```

### Release Artifacts to Dev-Repository
If the release candidate appears to pass the validation checklist, close the staging repository in Nexus by selecting the staging repository `orgapacherocketmq-XXX` and clicking on the `Close` icon.

Nexus will now run through a series of checksum and signature validations.

If the checks pass, Nexus will close the repository and give a URL to the closed staging repo (which contains the candidate artifacts). Include this URL in the voting email so that folks can find the staged candidate release artifacts.

If the checks do not pass, fix the issues, roll back and restart the release process. 

If everything is ok, use svn to copy the candidate release artifacts to RocketMQ repo: https://dist.apache.org/repos/dist/dev/incubator/rocketmq/${release version}.

## Vote on the Release

As per the Apache Incubator [release guidelines](http://incubator.apache.org/incubation/Incubation_Policy.html#Releases), all releases for incubating projects must go through a two-step voting process. First, release voting must successfully pass within the Apache RocketMQ community via the **dev@rocketmq.incubator.apache.org** mailing list. Then, release voting must successfully pass within the Apache Incubator PMC via the **general@incubator.apache.org** mailing list.

General information regarding the Apache voting process can be found [here](http://www.apache.org/foundation/voting.html).

### Apache RocketMQ Community Vote
To vote on a candidate release, send an email to the [dev list](mailto:dev@rocketmq.apache.incubator.org) with subject **[VOTE]: Release Apache RocketMQ \<release version\>(incubating) RC\<RC Number\>** and a body along the lines of:

> Hello RocketMQ Community,  
>
> This is the vote for \<release version\> of Apache RocketMQ (incubating).  
>
> **The artifacts:**  
> https://dist.apache.org/repos/dist/dev/incubator/rocketmq/${release version}
>
> **The staging repo:**  
> https://repository.apache.org/content/repositories/orgapacherocketmq-XXX/
>
> **Git tag for the release:**  
> \<link to the tag of GitHub repo\>  
>
> **Hash for the release tag:**  
> \<Hash value of the release tag\>  
>
> **Release Notes:**  
> \<insert link to the rocketmq release notes\>  
>
> The artifacts have been signed with Key : \<ID of signing key\>, which can be found in the keys file:  
> https://dist.apache.org/repos/dist/dev/incubator/rocketmq/KEYS  
>
> The vote will be open for at least 72 hours or until necessary number of votes are reached.  
>
> Please vote accordingly:  
>
> [ ] +1  approve    
> [ ] +0  no opinion    
> [ ] -1  disapprove with the reason    
>
> Thanks,  
> The Apache RocketMQ Team  

Once 72 hours has passed (which is generally preferred) and/or at least three +1 (binding) votes have been cast with no -1 (binding) votes, send an email closing the vote and pronouncing the release candidate a success. Please use the subject: **[RESULT][VOTE]: Release Apache RocketMQ \<release version\>(incubating) RC\<RC Number\>** :  

> Hello RocketMQ Community,  
>
> The Apache RocketMQ <release version> vote is now closed and has passed with [number] binding +1s, [number] non-binding +1s and no 0 or -1:  
>
> **Binding votes +1s:**  
> User Name (Apache ID)    
> User Name (Apache ID)    
> User Name (Apache ID)    
> ....
>
> **Non-binding votes +1s:**  
> User Name (Apache ID)  
> ....  
>
> A vote Apache RocketMQ \<release version\> will now be called on general@incubator.apache.org.  
>
> Thanks,   
> The Apache RocketMQ Team

If we do not pass the VOTE, fix the related issues, roll back, restart the release process and increase RC number. When we call a new vote, we must use the updated mail subject: **[RESTART][VOTE][#\<Attempt Number\>]: Release Apache RocketMQ \<release version\>(incubating) RC\<RC Number\>**

### Incubator PMC Vote
Once the candidate release vote passes on dev@rocketmq, send an email to [IMPC](mailto:general@incubator.apache.org) with subject **[VOTE]: Release Apache RocketMQ \<release version\>(incubating) RC\<RC Number\>** and a body along the lines of:

> Hello Incubator PMC,  
>
> The Apache RocketMQ community has voted and approved the proposal to release Apache RocketMQ \<release version\> (incubating). We now kindly request the IPMC review and vote on this incubator release.  
>
> **[VOTE] Thread:**  
> \<link to the dev voting mail-archive\>  
>
> **[RESULT][VOTE] Thread:**  
> \<link to the dev voting mail-archive\>  
>
> **The artifacts:**  
> https://dist.apache.org/repos/dist/dev/incubator/rocketmq/${release version}  
>
> **The staging repo:**  
> https://repository.apache.org/content/repositories/orgapacherocketmq-XXX/  
>
> **Git tag for the release:**  
> \<link to the tag of GitHub repo\>  
> 
> **Hash for the release tag:**  
> \<Hash value of the release tag\>  
>
> **Release Notes:**  
> \<insert link to the rocketmq release notes\>  
>
> The artifacts have been signed with Key : \<ID of signing key\>, which can be found in the keys file:  
> https://dist.apache.org/repos/dist/dev/incubator/rocketmq/KEYS  
>
> The vote will be open for at least 72 hours or until necessary number of votes are reached.  
>
> Please vote accordingly:  
>
> [ ] +1  approve   
> [ ] +0  no opinion   
> [ ] -1  disapprove with the reason   
>
> Thanks,  
> The Apache RocketMQ Team

Also don't forget announce the vote result:

> Hello Incubator PMC,  
>
> The Apache RocketMQ <release version> vote is now closed and has passed wit [number] binding +1s, [number] non-binding +1s and no 0 or -1:  
>
> **Binding votes +1s:**  
> User Name (Apache ID)   
> User Name (Apache ID)   
> User Name (Apache ID)   
> ....  
>
> **Non-binding votes +1s:**  
> User Name (Apache ID)   
> ....  
>
> The Apache RocketMQ (incubating) community will proceed with the release.  
>
> Thanks,  
> The Apache RocketMQ Team  

## Publish the Release
Once the Apache RocketMQ PPMC and IPMC votes both pass, publish the release artifacts to the Nexus Maven repository and to the Apache release repository.

1. Publish the Maven Artifacts, release the Maven artifacts in Nexus by selecting the staging repository **orgapacherocketmq-XXX** and clicking on the `Release` icon.
2. Publish the Artifacts to the Apache Release Repository, use svn copy candidate release artifacts to https://dist.apache.org/repos/dist/release/incubator/rocketmq/${release version}

## Announce the Release
Send an email to **announce@apache.org**, **general@incubator.apache.org**, and **dev@rocketmq.apache.incubator.org** with the subject **[ANNOUNCE] Release Apache RocketMQ \<release version\>(incubating)** and a body along the lines of:

> Hi all,
>
> The Apache RocketMQ team would like to announce the release of Apache RocketMQ \<release version\> (incubating).  
>
> More details regarding Apache RocketMQ can be found at:  
> http://rocketmq.incubator.apache.org/  
>
> The release artifacts can be downloaded here:  
> https://dist.apache.org/repos/dist/release/incubator/rocketmq/${release version}  
>
> The release notes can be found here:  
> \<insert link to the rocketmq release notes\>  
>
> Thanks,  
> The Apache RocketMQ Team  
>
> --- DISCLAIMER  Apache RocketMQ is an effort undergoing incubation at The Apache Software Foundation (ASF), sponsored by the Apache Incubator PMC. Incubation is required of all newly accepted projects until a further review indicates that the infrastructure, communications, and decision making process have stabilized in a manner consistent with other successful ASF projects. While incubation status is not necessarily a reflection of the completeness or stability of the code,it does indicate that the project has yet to be fully endorsed by the ASF.

## References

[1]. http://pirk.incubator.apache.org/releasing  
[2]. http://htrace.incubator.apache.org/building.html  
[3]. http://slider.incubator.apache.org/developing/releasing.html  
[4]. http://streams.incubator.apache.org/release-management.html  

