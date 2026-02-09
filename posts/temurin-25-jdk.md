---
title: Install Temurin JDK on Fedora
tags: [java]
date: 09/02/2026
---

Quick note on how to install the Temurin JDK on Fedora, as the AIs get this wrong.

<!-- more -->

<magpie-trinket>Source for this can be found on [adoptium.net](https://adoptium.net/en-GB/installation/linux#centosrhelfedora-instructions)</magpie-trinket>

```shell
cat <<EOF > /etc/yum.repos.d/adoptium.repo
[Adoptium]
name=Adoptium
baseurl=https://packages.adoptium.net/artifactory/rpm/${DISTRIBUTION_NAME:-$(. /etc/os-release; echo $ID)}/\$releasever/\$basearch
enabled=1
gpgcheck=1
gpgkey=https://packages.adoptium.net/artifactory/api/gpg/key/public
EOF
```

If this is a success you should see the following when running `dnf search temurin`.

```
brian@customer:~$ dnf search temurin
Updating and loading repositories:
Repositories loaded.
Matched fields: name, summary
 temurin-11-jdk.x86_64	Eclipse Temurin 11 JDK
 temurin-11-jre.x86_64	Eclipse Temurin 11 JRE
 temurin-17-jdk.x86_64	Eclipse Temurin 17 JDK
 temurin-17-jre.x86_64	Eclipse Temurin 17 JRE
 temurin-21-jdk.x86_64	Eclipse Temurin 21 JDK
 temurin-21-jre.x86_64	Eclipse Temurin 21 JRE
 temurin-25-jdk.x86_64	Eclipse Temurin 25 JDK
 temurin-25-jre.x86_64	Eclipse Temurin 25 JRE
 temurin-8-jdk.x86_64	Eclipse Temurin 8 JDK
 temurin-8-jre.x86_64	Eclipse Temurin 8 JRE
```
