---
layout: post
title:  "Deploying to Maven Central using GitHub Actions"
date:   2021-02-03 20:52:00+01:00
categories: Java
background: "/images/central-gh-actions.png"
language: en
---

I have [this library that I work on][1] from time to time, that I published on Maven Central. A few days ago I released 
a new version, and since I always forget what the exact `mvn` command is, what my GPG passphrase is, etc. I thought it 
would be a good opportunity to automate the release process using [GitHub Actions][2].

I came across several outdated tutorials that create `settings.xml` and import GPG keys manually, but as of Feb. 2021 most
of this is not needed anymore, thanks to [recent changes in the `setup-java` action][setup-java-gpg].

## Prerequisites

From now on, I will assume that you already know how to [deploy on Maven Central via Sonatype OSSRH][ossrh]. This means
you created an account on Sonatype's Jira, your local `settings.xml` is already configured and your `pom.xml` has a
`<plugins>` section that looks more or less like this:

```xml
<build>
    <plugins>
        ...
        <plugin>
            <groupId>org.sonatype.plugins</groupId>
            <artifactId>nexus-staging-maven-plugin</artifactId>
            <version>1.6.8</version>
            <extensions>true</extensions>
            <configuration>
                <serverId>ossrh</serverId>
                <nexusUrl>https://oss.sonatype.org/</nexusUrl>
                <autoReleaseAfterClose>true</autoReleaseAfterClose>
            </configuration>
        </plugin>
        ...
    </plugins>
</build>

<profiles>
    <profile>
        <id>release</id>
        <build>
            <plugins>
                ...
                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-gpg-plugin</artifactId>
                    <version>1.6</version>
                    <executions>
                        <execution>
                            <id>sign-artifacts</id>
                            <phase>verify</phase>
                            <goals>
                                <goal>sign</goal>
                            </goals>
                        </execution>
                    </executions>
                </plugin>
                ...
            </plugins>
        </build>
    </profile>
</profiles>
```

## Entering GitHub Actions

Java [Actions workflows][workflow] often use a `setup-java` action which... well... sets up Java in the build runner:

```yaml
...
    - name: Set up JDK 11
      uses: actions/setup-java@v1
      with:
        java-version: 11
```

I thought this action was only used to download a JDK, but it turns out it can do more than that: it also knows
how to set up the runner to [publish artifacts on Maven Central][publish-central] (or any `<distributionManagement>` 
configured in your `pom.xml`, for that matter):

```yaml
jobs:
  build:

    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v2
    - name: Set up JDK 1.8
      uses: actions/setup-java@v1
      with:
        java-version: 1.8

    - name: Build with Maven
      run: mvn -B package --file pom.xml

    - name: Set up Apache Maven Central
      uses: actions/setup-java@v1
      with: # running setup-java again overwrites the settings.xml
        java-version: 1.8
        server-id: maven # Value of the distributionManagement/repository/id field of the pom.xml
        server-username: MAVEN_USERNAME # env variable for username in deploy
        server-password: MAVEN_CENTRAL_TOKEN # env variable for token in deploy
        gpg-private-key: ${{ secrets.MAVEN_GPG_PRIVATE_KEY }} # Value of the GPG private key to import
        gpg-passphrase: MAVEN_GPG_PASSPHRASE # env variable for GPG private key passphrase

    - name: Publish to Apache Maven Central
      run: mvn deploy
      env:
        MAVEN_USERNAME: maven_username123
        MAVEN_CENTRAL_TOKEN: ${{ secrets.MAVEN_CENTRAL_TOKEN }}
        MAVEN_GPG_PASSPHRASE: ${{ secrets.MAVEN_GPG_PASSPHRASE }}
```

As explained in the readme, the second invocation of `actions/setup-java@v1` will overwrite the runner's 
`settings.xml` with your Sonatype credentials and GPG passphrase, using environment variables:

```xml
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 https://maven.apache.org/xsd/settings-1.0.0.xsd">
  <servers>
    <server>
      <id>maven</id>
      <username>${env.MAVEN_USERNAME}</username>
      <password>${env.MAVEN_CENTRAL_TOKEN}</password>
    </server>
    <server>
      <id>gpg.passphrase</id>
      <passphrase>${env.MAVEN_GPG_PASSPHRASE}</passphrase>
    </server>
  </servers>
</settings>
```

The private GPG key stored in the `MAVEN_GPG_PRIVATE_KEY` secret will also be imported in a GPG keychain, allowing
`maven-gpg-plugin` to sign your artifacts correctly.

## Adding the missing pieces

In order to make the workflow run smoothly, I needed two additional pieces of information that are not (yet) mentionned
in the readme:

* to fill the `MAVEN_GPG_PRIVATE_KEY` secret, you need to [export your private GPG key using this command][export-gpg]:
 `gpg --armor --export-secret-keys KEY_ID`
* to avoid a `gpg: signing failed: Inappropriate ioctl for device` error, you need to [configure `maven-gpg-plugin`][ioctl]
like this:

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-gpg-plugin</artifactId>
    <version>1.6</version>
    <configuration>
        <!-- Prevent gpg from using pinentry programs -->
        <gpgArguments>
            <arg>--pinentry-mode</arg>
            <arg>loopback</arg>
        </gpgArguments>
    </configuration>
    ...
</plugin>
```

## Final note

Setting up this workflow was not that difficult, except for the two missing pieces that I fortunately fixed very easily,
thanks to issues and comments from other people who had the same problem before me. 10/10, would recommend 😜.

My final [release workflow][my-workflow] is available on GitHub.

[1]: https://github.com/bjansen/swagger-schema-validator/
[2]: https://github.com/features/actions
[ossrh]: https://central.sonatype.org/pages/ossrh-guide.html
[setup-java-gpg]: https://github.com/actions/setup-java/issues/43
[publish-central]: https://github.com/actions/setup-java#publishing-using-apache-maven
[export-gpg]: https://github.com/actions/setup-java/issues/100
[ioctl]: https://github.com/actions/setup-java/issues/83
[workflow]: https://docs.github.com/en/actions/learn-github-actions/introduction-to-github-actions#workflows
[my-workflow]: https://github.com/bjansen/swagger-schema-validator/blob/master/.github/workflows/release.yml
