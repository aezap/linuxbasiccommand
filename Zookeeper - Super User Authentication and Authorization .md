# Zookeeper - Super User Authentication and Authorization 

## Introduction

By default, Zookeeper runs without the option of becoming a superuser to administrate znodes in the ZK ensemble, for example, to fix ACLs, remove znodes that are not required anymore, or create new ones in specific locations. Zookeeper grants permissions through ACLs through different schemas or authentication methods, such as 'world', 'digest', or 'sasl' if we use Kerberos. We can potentially we locked out if we were to grant everyone just read permissions to a znode, as we would not be able to delete it or modify it anymore.

## xample of the problem

or example, we connect to Zookeeper through zkCli :

    /usr/hdp/current/zookeeper-client/bin/zkCli.sh -server `hostname -f`:2181

    [zk: sandbox.hortonworks.com:2181(CONNECTED) 1] getAcl /config/topics 
    'world,'anyone

    : r

If we need to modify that znode so that, for example, user 'kafka' can have access to it to create new topics :

    [zk: sandbox.hortonworks.com:2181(CONNECTED) 2] setAcl /config/topics world:anyone:r sasl:kafka:cdrwa
    Authentication is not valid : /config/topics

## Using superDigest to become a Zookeeper superuser

The following can be done to run as a Zookeeper superuser and be able to make ACL changes or delete/modify znodes. We can run the DigestAuthenticationProvider to get the digest of a given password. Foe example, if we want our superuser 'super' to have the password 'super123' we can:

    export ZK_CLASSPATH=/etc/zookeeper/conf/:/usr/hdp/current/zookeeper-server/lib/*:/usr/hdp/current/zookeeper-server/*

    java -cp $ZK_CLASSPATH org.apache.zookeeper.server.auth.DigestAuthenticationProvider super:super123

    OUTPUT: super:super123->super:UdxDQl4f9v5oITwcAsO9bmWgHSI= 


From the output we can just add the following to SERVER_JVMFLAGS and restart Zookeeper :

    SERVER_JVMFLAGS=-Dzookeeper.DigestAuthenticationProvider.superDigest=super:UdxDQl4f9v5oITwcAsO9bmWgHSI=

Then, in zkCli do:

    [zk: sandbox.hortonworks.com:2181(CONNECTED) 1] addauth digest super:super123

Now, we can perform the action we couldn't before :

    [zk: sandbox.hortonworks.com:2181(CONNECTED) 1] setAcl /config/topics world:anyone:cdrwa,sasl:kafka:cdrwa 

    cZxid = 0x29
    ctime = Tue Jul 21 18:46:36 UTC 2015 
    mZxid = 0x29
    mtime = Tue Jul 21 18:46:36 UTC 2015 
    pZxid = 0x22410 
    cversion = 4 
    dataVersion = 0 
    aclVersion = 9 
    ephemeralOwner = 0x0 
    dataLength = 0 
    numChildren = 4

    [zk: sandbox.hortonworks.com:2181(CONNECTED) 3] getAcl /config/topics
    'world,'anyone 
    : cdrwa 
    'sasl,'kafka 
    : cdrwa 