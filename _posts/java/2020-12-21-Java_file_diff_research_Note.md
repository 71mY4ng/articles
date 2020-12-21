# Java file diff research Note

## get nexus server file sha value

* [How do I retrieve an artifact checksum from Nexus using their rest api via curl?](https://stackoverflow.com/questions/25047781/how-do-i-retrieve-an-artifact-checksum-from-nexus-using-their-rest-api-via-curl)

- Nexus API

like 

```
http://localhost:8081/nexus/service/local/artifact/maven/resolve?g=log4j&a=log4j&v=1.2.9&r=central
```

return following:

```xml
<artifact-resolution>
  <data>
    <presentLocally>true</presentLocally>
    <groupId>log4j</groupId>
    <artifactId>log4j</artifactId>
    <version>1.2.9</version>
    <extension>jar</extension>
    <snapshot>false</snapshot>
    <snapshotBuildNumber>0</snapshotBuildNumber>
    <snapshotTimeStamp>0</snapshotTimeStamp>
    <sha1>55856d711ab8b88f8c7b04fd85ff1643ffbfde7c</sha1>
    <repositoryPath>/log4j/log4j/1.2.9/log4j-1.2.9.jar</repositoryPath>
  </data>
</artifact-resolution>
```


- [Nexus Search api](https://help.sonatype.com/repomanager3/rest-and-integration-api/search-api#SearchAPI-SearchandDownloadAsset)

```shell
curl -sL 'https://mynexus/service/rest/v1/search/assets/download?repository=my-repo&sort=version&maven.groupId=com.example&maven.artifactId=example&maven.baseVersion=LATEST-SNAPSHOT&maven.extension=jar.sha1&maven.classifier='
```

