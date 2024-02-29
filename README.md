# **Hadoop DFS**: Deploy a Distributed File System via Vagrant + Ansible

HDFS cluster deployment using VMs as platform

This repository shows how to create a cluster with distributed file system based on Hadoop DFS technology. The deployment is done with VirtualBox machines and Ansible for provisioning.

> You can clone this repository to deploy it or ever you can replicate the steps to improve your learning. <span style="color:yellow">*In clone case*</span> you must create the ssh key and download the hadoop binary file.

## Hadoop DFS (HDFS)
Hadoop Distributed File System (HDFS), often referred to as Hadoop DFS, is a distributed file system designed to run on commodity hardware. It serves as the primary storage system for Hadoop applications, especially those dealing with big data.

HDFS, divides the files into blocks and distribute it in different nodes of the cluster. Each block is replicated in multiple nodes for guarantee the fault tolerance and data availability.

### Architecture:

- NameNode: This maintains the register and location of the blocks and coordinate the operations of the file system.
- DataNode: This stores the data blocks and manage the read/write operations.

### Blocks and replication:

The files are divided into blocks (128MB or 64MB, by default). Each block is replicated in multiple nodes.

## Objectives

- Remember the distributed file system parts.
- Understand how a distributed file system works.
- Apply Hadoop DFS in a cluster.

## Requirements for this practice:

- Previously installed Virtualbox
- Previously installed Vagrant
- Previously installed Ansible

## Deploy Hadoop DFS

### 1. Pre requirements

In the host machine, create the project directory and change into it:

```bash
$ mkdir hdfs
$ cd hdfs
```

Create a data directory and change into it:

```bash
$ mkdir src
$ cd src
```

Download the hadoop package and unarchive into src folder

```bash
$ wget https://dlcdn.apache.org/hadoop/common/hadoop-3.3.6/hadoop-3.3.6.tar.gz
```

<!-- $ tar -xf hadoop-3.3.6.tar.gz -->

### 2. Create a Public/Private key

This key will be used for SSH access:

```bash
ssh-keygen -t rsa -N "" -f ./id_rsa -C hadoop
```

Two files must be created:
- **id_rsa**: private key, used for SSH login
- **id_rsa.pub**: public key, used for GCP VM configuration

### 3. Create templates for the project

Change into the folder `hdfs` and create `templates` folder, then change into it:

```bash
$ cd hdfs
$ mkdir templates
$ cd templates
```

Create the generic template `core-site.xml.j2`, as follows:

<details>
  <summary>core-site.xml.j2</summary>

```bash
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>

<configuration>
  <property>
    <name>fs.defaultFS</name>
    <value>hdfs://{{NAMENODE_NAME}}:9000</value>
  </property>
</configuration>
```

</details>

Create the NameNode specific template `hdfs-site-namenode.xml.j2`, as follows:

<details>
  <summary>hdfs-site.xml.j2</summary>

```bash
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>

<configuration>
  <property>
    <name>dfs.namenode.name.dir</name>
    <value>file://{{HDFS_HOME}}/dfs/name</value>
  </property>
  <property>
    <name>dfs.datanode.data.dir</name>
    <value>file://{{HDFS_HOME}}/dfs/data</value>
  </property>
  <property>
    <name>dfs.namenode.rpc-address</name>
    <value>0.0.0.0:9000</value>
  </property>
</configuration>
```
</details>

Create the DataNode specific template `hdfs-site-datanode.xml.j2` template, as follows:

<details>
  <summary>hdfs-site-datanode.xml.j2</summary>

```bash
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>

<configuration>
  <property>
    <name>{{RESOURCE_NAME_DIR}}</name>
    <value>file://{{HDFS_HOME}}/dfs/data</value>
  </property>
</configuration>
```

</details>

Now return to `hdfs` folder

```bash
templates $ cd ..
hdfs $ |
```

### 4. Create Vagrant project

Make sure if you are placed in the `hdfs` directory and continue the next steps.

#### Initialize the directory
Vagrant has a built-in command for initializing a project, `vagrant init`, which can take a box name and URL as arguments. Initialize the directory and specify the `generic/debian11` box.

```bash
$ vagrant init generic/debian11

A `Vagrantfile` has been placed in this directory. You are now ready to `vagrant up` your first virtual environment! Please read the comments in the Vagrantfile as well as documentation on `vagrantup.com` for more information on using Vagrant.
```

You now have a Vagrantfile in your current directory. Open the Vagrantfile, which contains some pre-populated comments and examples. In following tutorials you will modify this file.

Expand the file below to view the entire contents of the example Vagrantfile.

<details>
  <summary><b>Show Vagrantfile</b></summary>

```bash
# -*- mode: ruby -*-
# vi: set ft=ruby :

# All Vagrant configuration is done below. The "2" in Vagrant.configure 
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.
Vagrant.configure("2") do |config|
    # The most common configuration options are documented and commented below.
    # For a complete reference, please see the online documentation at
    # https://docs.vagrantup.com.

  config.vm.define :namenode do |config|
    config.vm.box = "generic/debian11"
    config.vm.hostname = "namenode"
    config.vm.network "private_network", ip: "192.168.56.10"
    config.vm.provider "virtualbox" do |v|
      v.memory = 1024  # Set RAM to 1 GB (adjust as needed)
      v.cpus = 1       # Set CPU cores to 1 (adjust as needed)
    end
    config.vm.provision "ansible" do |ansible|
      ansible.playbook = "namenode.yml"
    end
  end

  [ "datanode1", "datanode2" ].to_enum.with_index(1).each do |host, i|
    config.vm.define "#{host}" do |config|
      config.vm.box = "generic/debian11"
      config.vm.hostname = "#{host}"
      config.vm.network "private_network", ip: "192.168.56.1#{i}"
      config.vm.provider "virtualbox" do |v|
        v.memory = 1024  # Set RAM to 1 GB (adjust as needed)
        v.cpus = 1       # Set CPU cores to 1 (adjust as needed)
      end
      config.vm.provision "ansible" do |ansible|
        ansible.playbook = "datanode.yml"
      end
    end
  end
end
```
</details>

The previous configuration as it, creates three instances, one NameNode (master) and two DataNodes (slaves). The number of DataNodes can be changed by adding or removing items in this array `[ "datanode1", "datanode2" ]` where the number at the of `datanodeX` is the datanode index.

### 5. Create Ansible files
On the path `hdfs`, create three playbook files named `namenode.yml`, `datanode.yml` and `common_tasks.yml`

The `namenode.yml` file, is the specific ansible playbook for the NameNode instance.

<details>
  <summary><b>Show namenode.yml</b></summary>

```yaml
- name: CC NameNode
  hosts: namenode
    #  connection: local
  become: true

  vars:
    LOCAL_SRC_HOME: "./src"
    SRC_HOME: "/opt/hadoop"
    HDFS_HOME: "/home/hdfs"
    HDFS_USER: "hdfs"
    HDFS_PASSWORD: "hdfs"
    NAMENODE_NAME: 0.0.0.0
    RESOURCE_NAME_DIR: dfs.namenode.name.dir

  tasks:
  - import_tasks: common_tasks.yml

  - name: Config hdfs-site.xml
    template:
      src: "hdfs-site-namenode.xml.j2"
      dest: "/opt/hadoop-3.3.6/etc/hadoop/hdfs-site.xml"
      owner: "{{HDFS_USER}}"
      group: "{{HDFS_USER}}"

  # - name: Format filesystem
  #   ansible.builtin.command:
  #     become: hdfs
  #     cmd: /opt/hadoop-3.3.6/bin/hdfs namenode -format
```

</details>

In the other hand, `datanode.yml` is the specific ansible playbook for the datanodes.

> In this context, the `hosts` variable depends of the datanodes declared in the VagrantFile.

<details>
  <summary><b>Show datanode.yml</b></summary>

```yaml
- name: CC DataNode
  hosts: datanode1,datanode2
    #  connection: local
  become: true

  vars:
    LOCAL_SRC_HOME: "./src"
    SRC_HOME: "/opt/hadoop"
    HDFS_HOME: "/home/hdfs"
    HDFS_USER: "hdfs"
    HDFS_PASSWORD: "hdfs"
    NAMENODE_NAME: 192.168.56.10
    RESOURCE_NAME_DIR: dfs.datanode.data.dir

  tasks:
  - import_tasks: common_tasks.yml

  - name: Config hdfs-site.xml
    template:
      src: "hdfs-site-datanode.xml.j2"
      dest: "/opt/hadoop-3.3.6/etc/hadoop/hdfs-site.xml"
      owner: "{{HDFS_USER}}"
      group: "{{HDFS_USER}}"

  # - name: Format filesystem
  #   ansible.builtin.command:
  #     become: hdfs
  #     cmd: /opt/hadoop-3.3.6/bin/hdfs datanode -format
```

</details>

At the last, `common_tasks.yml` is the common tasks ansible playbook, this is be cause the base for NameNode and DataNode is actually the same for both. Only configurations are specific in each case. So, create `common_tasks.yml` with the next content:

<details>
  <summary><b>Show common_tasks.yml</b></summary>

```yaml
- name: Insert namenode in /etc/hosts
  ansible.builtin.shell:
    cmd: echo '192.168.56.10  namenode' >> /etc/hosts

- name: Insert datanode1 in /etc/hosts
  ansible.builtin.shell:
    cmd: echo '192.168.56.11  datanode1' >> /etc/hosts

- name: Insert datanode2 in /etc/hosts
  ansible.builtin.shell:
    cmd: echo '192.168.56.12  datanode2' >> /etc/hosts

- name: Create hdfs user
  user:
    name: "{{HDFS_USER}}"
    state: present
    password: "{{ HDFS_PASSWORD | password_hash('sha512','A512') }}"
    shell: /bin/bash
    groups: users, sudo

- name: Create SSH directory
  file: path="{{HDFS_HOME}}/.ssh" state=directory owner=hdfs group=hdfs

- name: Create dfs directory
  file: path="{{HDFS_HOME}}/dfs" state=directory owner=hdfs group=hdfs

- name: Create name directory
  file: path="{{HDFS_HOME}}/dfs/name" state=directory owner=hdfs group=hdfs

- name: Create data directory
  file: path="{{HDFS_HOME}}/dfs/data" state=directory owner=hdfs group=hdfs

- name: apt install required packages
  apt:
    update_cache: yes
    name:
      - openjdk-11-jdk
    state: present

# - name: Create remote directory
#   file: path={{SRC_HOME}} state=directory

# - name: Copy hadoop to vm
#   copy: src="{{LOCAL_SRC_HOME}}/hadoop-3.3.6/" dest={{SRC_HOME}}/

- name: Extract hadoop-3.3.6.tar.gz into /opt/hadoop
  ansible.builtin.unarchive:
    src: "{{LOCAL_SRC_HOME}}/hadoop-3.3.6.tar.gz"
    dest: /opt
    owner: "hdfs"
    group: "hdfs"

- name: Copy id_rsa private key and public one are present
  ansible.builtin.copy:
    src: "{{LOCAL_SRC_HOME}}/{{item}}"
    dest: "{{HDFS_HOME}}/.ssh/{{item}}"
    mode: 0600
    owner: hdfs
    group: hdfs
  with_items:
    - id_rsa
    - id_rsa.pub

- name: Copy Key in authorized_key
  authorized_key:
    user: hdfs
    key: "{{ lookup('file', '{{LOCAL_SRC_HOME}}/id_rsa.pub') }}"
    state: present

- name: Insert JAVA_HOME in .bashrc
  ansible.builtin.shell:
    cmd: echo 'export JAVA_HOME="/usr/lib/jvm/java-11-openjdk-amd64"' >> {{HDFS_HOME}}/.bashrc

- name: Insert HADOOP_HOME in .bashrc
  ansible.builtin.shell:
    cmd: echo 'export HADOOP_HOME="/opt/hadoop-3.3.6"' >> {{HDFS_HOME}}/.bashrc

- name: Insert HADOOP_HOME in PATH if .bashrc
  ansible.builtin.shell:
    cmd: echo 'export PATH=$PATH:$HADOOP_HOME/bin' >> {{HDFS_HOME}}/.bashrc

- name: Hadoop Environmental variables
  replace:
    path: /opt/hadoop-3.3.6/etc/hadoop/hadoop-env.sh
    regexp: '#(\s+)export JAVA_HOME='
    replace: 'export JAVA_HOME="/usr/lib/jvm/java-11-openjdk-amd64"'
    backup: yes

- name: Hadoop Environmental variables
  replace:
    path: /opt/hadoop-3.3.6/etc/hadoop/hadoop-env.sh
    regexp: '#(\s+)export HADOOP_HOME='
    replace: 'export HADOOP_HOME="/opt/hadoop-3.3.6"'
    backup: yes

- name: Config core-site.xml
  template:
    src: "core-site.xml.j2"
    dest: "/opt/hadoop-3.3.6/etc/hadoop/core-site.xml"
    owner: "{{HDFS_USER}}"
    group: "{{HDFS_USER}}"
```

</details>

> Please pay attention in the blocks corresponding to `Insert datanodeX in /etc/hosts` at the head lines. `X` is the index of the datanode, so if you change the datanode quanity, you must modify this blocks according to this.

### 6. Deploy the vagrant+ansible project
Now run `vagrant up` to deploy the virtual machines

```bash
$ vagrant up
```

> This takes a while, so get time to relax and drink a coffee.

### 7. Hadoop Configuration

> By default, this repository already have pre-configured the files for each instance, once Vagrant+Ansible are deployed. But here is the content for your information.

Depending of the type of machine, it is necessary to edit the configuration files for Hadoop. This files are located in /opt/hadoop-3.3.6/etc/hadoop, and this are:

- hadoop-env.sh
- core-site.xml
- hdfs-site.xml

#### 7.1. Common configuration

For all instances, the environmental variables must be changed in hadoop-env.sh, locating the next lines and change it (the ansible script in this practice already includes this part, but always we could do this as follows):

Previous
```bash
...
#export JAVA_HOME=
...
#export HADOOP_HOME=
...
```

Modified
```bash
...
export JAVA_HOME="/usr/lib/jvm/java-17-openjdk-amd64"
...
export HADOOP_HOME="/opt/hadoop"
...
```

#### 7.2. NameNode configuration

For this instance, you must have the config files core-site.xml and hdfs-site.xml whit the next content:

<details>
  <summary>core-site.xml</summary>

```bash
<configuration>
  <property>
    <name>fs.defaultFS</name>
    <value>hdfs://0.0.0.0:9000</value>
  </property>
</configuration>
```
</details>

<details>
  <summary>hdfs-site.xml</summary>

```bash
<configuration>
  <property>
    <name>dfs.namenode.name.dir</name>
    <value>file:///home/hdfs/dfs/name</value>
  </property>
  <property>
    <name>dfs.datanode.data.dir</name>
    <value>file:///home/hdfs/dfs/data</value>
  </property>
  <property>
    <name>dfs.namenode.rpc-address</name>
    <value>0.0.0.0:9000</value>
  </property>
</configuration>
```

</details>

#### 7.3. DataNode configuration

For the datanode instances, the config files core-site.xml and hdfs-site.xml has the next:

<details>
  <summary>core-site.xml</summary>

```bash
<configuration>
  <property>
    <name>fs.defaultFS</name>
    <value>hdfs://192.168.56.10:9000</value>
  </property>
</configuration>
```
</details>

<details>
  <summary>hdfs-site.xml</summary>

```bash
<configuration>
  <property>
    <name>dfs.datanode.data.dir</name>
    <value>file:///home/hdfs/dfs/data</value>
  </property>
</configuration>
```

</details>

### 8. Hadoop Startup

It is necessary to get into each virtual machine and run the daemon manually (this is be cause this practice doesn't include a systemctl service file implementation).

For each instance, you can get into through SSH as follows:

```bash
hdfs$ ssh hdfs@192.168.56.1x -i ./src/id_rsa -o StrictHostKeyChecking=no
```

> Remember: \
`x` in the IP address is the index of the node (0 for namenode) \
`-i ./src/id_rsa` previously we have created a ssh key named id_rsa \
`-o StrictHostKeyChecking=no` is the parameter used for avoid SSH to validate the kwon_hosts inclusion.

#### 8.1. NameNode startup

Get into NameNode through SSH and in the first time you bring up HDFS, it must be formatted. In the NameNode instance, format a new distributed filesystem as follow:

```bash
$hdfs namenode -format
```

Once is formatted, you can run the daemon with:

```bash
$hdfs --daemon start namenode
```

In the NameNode case, you could (optionaly) start a datanode daemon, this will be shown after.

#### 8.2. DataNode startup

Get into the DataNodes through SSH and run the daemon as follows:

```bash
$hdfs --daemon start datanode
```

### 9. Check the results

You can browse to 192.168.56.10:9870 and check the results.

## Working with the file system

Let's execute the commands available, do this in the NameNode. Get into this through SSH and test the commands listed below, to work with this file system:

> Important: at the firts time is necessary to create the user folder for work with the file system, as follows:
```bash
$ hdfs dfs -mkdir /user
$ hdfs dfs -mkdir /user/hdfs
```
> In this context, hdfs is the username of your host computer.

- hdfs dfs -ls \<path>
- hdfs dfs -mkdir \<folder name>
- hdfs dfs -touchz \<file_path>
- hdfs dfs -put \<local file path> \<dest(present on hdfs)>
- hdfs dfs -cat \<path>
- hdfs dfs -get \<srcfile(on hdfs)> \<local file dest>
- hdfs dfs -cp \<src(on hdfs)> \<dest(on hdfs)>
- hdfs dfs -mv \<src(on hdfs)> \<dest(on hdfs)>
- hdfs dfs -rmr \<filename/directoryName>
- hdfs dfs -du \<dirName>
- hdfs dfs -dus \<dirName>
- hdfs dfs -stat \<hdfs file>
- hdfs dfs -setrep -R -w 6 \<file>

## References
> https://www.bmc.com/blogs/hadoop-architecture/ \
> https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/ClusterSetup.html