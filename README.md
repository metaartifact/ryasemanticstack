# Introduction

Below process will configure Apache Rya 4.0.0 based on Accumulo2.0, Hadoop3.1.3 with Zookeeper3.4.13 on single machine. Hadoop would be configured simple single machine cluster (without YARN) for this configuration. This is only for setting your development environment and not for production run purposes
 
### Download stack components;
  
    download accumulo 1.9.3
    download zookeeper 3.4.13
    download hadoop 3.1.3
    download tomcat 8.15
    download RYA sources 4.0.0

### Start and Run Hadoop (single node cluster):

    cd hadoop/

Configure core-site.xml file with below new property;

    vi /etc/hadoop/core-site.xml
    <configuration>
        <property>
            <name>fs.defaultFS</name>
            <value>hdfs://localhost:9000</value>
        </property>
    </configuration>

and for hdfs-site.xml

    vi etc/hadoop/hdfs-site.xml:
    <configuration>
        <property>
            <name>dfs.replication</name>
            <value>1</value>
        </property>
        <property>
            <name>dfs.namenode.rpc-address</name>
            <value>localhost:8020</value>
        </property>
    </configuration>

Format the namenode, start the namenode and datanode in SingleNode Cluster 

      bin/hdfs namenode -format
      sbin/start-dfs.sh

test if hadoop is setup correctly:
copy files to HDFS

      bin/hdfs dfs -mkdir /user
      bin/hdfs dfs -mkdir /user/lw5523
      bin/hdfs dfs -mkdir input
      bin/hdfs dfs -put etc/hadoop/*.xml input

Run mapreduce job example provided

      bin/hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-3.1.3.jar grep input output 'dfs[a-z.]+'

move the output files to local directory
    
      bin/hdfs dfs -get output output

Read the output

      cat output/*

Check namenode web console
http://localhost:9870/

Check datanode web console
http://localhost:9864/

[optional] How to stop hadoop cluster (single node)
      sbin/stop-dfs.sh

### Start and Run Zookeeper (single node):

Start standalone Zookeeper. Its important that that dataDir is changed to a path other than /tmp/ 

      cd zookeeper/
      vi conf/zoo.cfg 

sample zoo.cfg file is

    tickTime=2000
    initLimit=10
    syncLimit=5
    dataDir=/opt/dataDir
    clientPort=2181

Star zookeeper

      bin/zkServer.sh start

Verify and connect to zookeeper CLI

    bin/zkCli.sh -server 127.0.0.1:2181

[optional]
On the CLI you can do some commands like help, ls /, create new_znode, delete new_znode, set /new_znode junk, get /new_znode

[optional] How to stop zookeeper
  
    606  bin/zkServer.sh stop

### Start and Run Accumulo (single node - v1.9.3):

Start Acccumulo as service;

initialize configuration; a few question/answers to follow after below command
    
    cd <directory of Accumulo>
    ./bin/build_native_library.sh
    ./bin/bootstrap_config.sh

Set accumulo-env.sh class paths for Hadoop and Zookeeper folders

    if [[ -z $HADOOP_HOME ]] ; then
        test -z "$HADOOP_PREFIX"      && export HADOOP_PREFIX=/opt/hadoop
    else
    HADOOP_PREFIX="$HADOOP_HOME"
    unset HADOOP_HOME
    fi

    # hadoop-2.0:
    test -z "$HADOOP_CONF_DIR"       && export HADOOP_CONF_DIR="$HADOOP_PREFIX/etc/hadoop"

    test -z "$JAVA_HOME"             && export JAVA_HOME=/path/to/java
    test -z "$ZOOKEEPER_HOME"        && export ZOOKEEPER_HOME=/opt/zookeeper

Set property `instance volumes` as below;

    <name>instance.volumes</name>
    <value>hdfs://localhost:8020/accumulo1</value>

Set property `instance zookeeper host` as below;

    <name>instance.zookeeper.host</name>
    <value>localhost:2181</value>

Modify `general classpaths` as below for hadoop 3 to add all dependencies

      <!-- Hadoop 3 requirements -->
      $HADOOP_PREFIX/share/hadoop/client/[^.].*.jar,
      $HADOOP_PREFIX/share/hadoop/common/[^.].*.jar,
      $HADOOP_PREFIX/share/hadoop/common/lib/(?!slf4j)[^.].*.jar,
      $HADOOP_PREFIX/share/hadoop/hdfs/[^.].*.jar,
      $HADOOP_PREFIX/share/hadoop/mapreduce/[^.].*.jar,

Answer some questions, give 'accumulo' when asked for instance name (or any of your choice) and root password as root (or any of your choice) when asked in below step!

    ./bin/accumulo classpath
    ./bin/accumulo init #asks to set accumulo clusterid and root user password
    ./bin/start-all.sh

Check the monitoring of accumulo here;]
`http://localhost:9995/`

access Accumulo CLI after setting `auth.principal=<username>` and `auth.token=<userpassword>` - incase of dev env I set it as root root for both

    accumulo shell -u root

[optional] How to stop the accumulo service;

    ./bin/stop-all.sh


### Start and Run RYA (single node):

Download Source from https://rya.apache.org/download/
you need to build RYA from sources as its source code distribution (no Binary)

once downloaded unzip and go to its root folder;
Need Java 8 to compile (as of 30/12/2020 )

    mvn clean install -DskipTests

If Java 8 is not set, set JAVA_HOME to JAVA 8 [below for MAC]
/usr/libexec/java_home -V

    /usr/libexec/java_home -V
    export JAVA_HOME=`/usr/libexec/java_home -v 1.8.0_181`
    java --version
    vi ~/.bashrc 

above `mvn clean install` will produce deployable war file for Rya at below location;
 `<rya_folder>/web/web.rya/target/web.rya.war`

### Place Deployable WAR to Tomcat:

Download and install TOMCAT (8.5) and put `web.rya.war` into `webapps` folder of tomcat (and extract the war)

Create a file in `<tomcat-home>/webapps/web.rya/WEB-INF/classes/` with the name `environment.properties` with following content

    instance.name=accumulo
    instance.zk=localhost:2181
    instance.username=root
    instance.password=root
    rya.tableprefix=rya_
    rya.displayqueryplan=true

Start TOMCAT;

    bin/startup.sh

Apache rya is now available at below link inside tomcat applications manager;

    http://localhost:8080/manager/

all web.rya functions are now available at URL (please note the link itself has no webpage);

    http://localhost:8080/web.rya

Load data REST endpoint;
    
    http://localhost:8080/web.rya/loadrdf

Query data REST endpoint;

    http://localhost:8080/web.rya/queryrdf?query=

For example code on load/query data and others please check github.com/metaartifact/ryaexamples
