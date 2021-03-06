---
title: "Setting up a single server SLURM cluster"
layout: post
date: 2015-04-20 23:00 +0200
tags:
- SLURM
---
Shared computation resources can easily get crowded as everyone log on and start their jobs. Running above full load will cause excess context switching and in worst case swapping, resulting in sub-optimal performance. We would then benefit from running the tasks ordered in a more optimal manner. Coordinating this is a job which could be left to the machine itself.

In this post, I'll describe how to setup a single-node SLURM mini-cluster to implement such a queue system on a computation server. I'll assume that there is only one node, albeit with several processors. The computation server we use currently is a 4-way octocore E5-4627v2 3.3 GHz Dell PowerEdge M820 with 512 GiB RAM.

The setup will be described in the usual "read-along" style, with commands to follow. And yes, in case you wondered, you'll need sudo access to the machine!

Setting up Control Groups
-------------------------
We don't have a separate login node as is usual on supercomputers, so the entire machine is still usable directly from an SSH command-line. To discourage this, we need to take away the ability to get unfettered access to the machine.

Cgroups is a neat Linux feature to control process resource use, but it is not installed by default on a Ubuntu Trusty system, so we start out by installing the package which contains this feature. We add the ability to restrict memory usage too, which require kernel command-line option on Debian derivatives.

{% highlight sh %}
sudo apt-get install -y cgroup-bin
sudo sed -i 's,\(^GRUB_CMDLINE_LINUX_DEFAULT\)=\"\(.*\)",\1=\"\2 cgroup_enable=memory swapaccount=1\",' /etc/default/grub
sudo update-grub
{% endhighlight %}

A quick starter for those who haven't used Cgroups before: A *controller* is a resource that is restricted, such as CPU or memory, whereas a *cgroup* is a policy for some set of processes on how to use these resources. The controllers are built-in and the cgroups are setup by administrators.

We'll set up two cgroups, a root cgroup which have unrestricted access to the machine as before, and an "interactive" cgroup which we'll limit to only half of a single CPU and 2 GiB of RAM, more or less equivalent to an old workstation. (Note that comments in this config file must start at the very beginning of the line).

{% highlight sh %}
sudo sh -c "cat > /etc/cgconfig.conf" <<EOF
group interactive {
   cpu {
# length of the reference period is 1 second (across all cores)
      cpu.cfs_period_us = "1000000";
# allow this group to only use total of 0.25 cpusecond (across all cores)
# i.e. throttle the group to half of a single CPU
      cpu.cfs_quota_us = "500000";
   }
   memory {
# 2 GiB limit for interactive users
      memory.limit_in_bytes = "2G";
      memory.memsw.limit_in_bytes = "2G";
   }
}
EOF
{% endhighlight %}

Now, Ubuntu 12.04 has a `cgconfig` daemon that take care of setting up cgroups, and Ubuntu 16.04 will use systemd to do the same, but 14.04 falls in between and has no default way of setting up custom cgroups (!). We'll solve this by adding an Upstart script which does this based on the configuration file:

{% highlight sh %}
sudo sh -c "cat >> /etc/init/cgroup-config.conf" <<EOF
description "Setup initial cgroups"
task
start on started cgroup-lite
script
/usr/sbin/cgconfigparser -l /etc/cgconfig.conf
end script
EOF
{% endhighlight %}

Now we have our cgroups, next task is to setup how processes are assigned to them based on user groups. Here we let the root user (which starts all the daemons) and the slurm user, which will be setup later and which starts jobs from the queue, get the root cgroup. Everyone else (which has logged on with SSH) is supposed to be put in the "interactive" group.

{% highlight sh %}
sudo sh -c "cat >> /etc/cgrules.conf" <<EOF
root             cpu,memory    /
slurm            cpu,memory    /
*                cpu,memory    /interactive
EOF
{% endhighlight %}

But still there is nothing which actually performs these assignments. We solve this by installing a PAM plugin, which will move the login process of people SSHing into the system, into the right cgroup:

{% highlight sh %}
sudo apt-get install -y libpam-cgroup
sudo sh -c "cat >> /etc/pam.d/common-session" <<EOF
session optional        pam_cgroup.so
EOF
{% endhighlight %}

Picking Passwords
-----------------
SLURM will store the accounting information in a database. Unfortunately, Postgres-based accounting is not mature yet, SQLite-based accounting is non-existing and file-based accounting does not work properly and is deprecated. Thus, we are stuck with using MySQL, which I have to admin is not my favorite to have running on the server.

Anyway, we'll try to lock it down as good we can, and in that process we'll need two passwords: One to access the meta-database in MySQL itself, and one to access the accounting database. These passwords will be written down in (locked-down) configuration files (!), so we can generate more secure passwords since we don't have to memorize them:

{% highlight sh %}
sudo apt-get install -y pwgen
DB_ROOT_PASS=$(pwgen -s 16 1)
DB_SLURM_PASS=$(pwgen -s 16 1)
{% endhighlight %}

Since we need these passwords two places: once in MySQL itself and once in the configuration file, I'll put it in environment variables.

Setting up MySQL
----------------
First we setup the database daemon itself and make sure that it only listen on local connections:

{% highlight sh %}
sudo DEBIAN_FRONTEND=noninteractive apt-get install -y mysql-server
sudo sed -i 's/^\(bind-address[\ \t]*\)=\(.*\)/\1= 127.0.0.1/' /etc/mysql/my.cnf
{% endhighlight %}

Next, we'll enable the root user to access the database password-less (if you have access to the root user's account or files you have bigger problems than this anyway):

{% highlight sh %}
sudo touch ~root/.my.cnf
sudo chown root:root ~root/.my.cnf
sudo chmod 700 ~root/.my.cnf
sudo sh -c "cat > ~root/.my.cnf" <<EOF
[client]
user=root
password=$DB_ROOT_PASS
EOF
{% endhighlight %}

Set the root user password in MySQL, allow only local connections and remove the anonymous guest user and the test database:

{% highlight sh %}
sudo mysql --user=root --password="" <<EOF
update mysql.user set password=password('$DB_ROOT_PASS') where user='root';
delete from mysql.user where user='root' and host not in ('localhost', '127.0.0.1', '::1');
delete from mysql.user where user='';
flush privileges;
EOF
{% endhighlight %}

Now we have a satisfactory MySQL instance, let's create the database for SLURM accounting:

{% highlight sh %}
sudo HOME=~root mysql <<EOF
create database slurm_acct_db;
create user 'slurm'@'localhost';
set password for 'slurm'@'localhost' = password('$DB_SLURM_PASS');
grant usage on *.* to 'slurm'@'localhost';
grant all privileges on slurm_acct_db.* to 'slurm'@'localhost';
flush privileges;
EOF
{% endhighlight %}

Setting up the SLURM daemons
----------------------------
SLURM consists of four daemons: "munge", which will authenticate users to the cluster, "slurmdbd" which will do the authorization, i.e. checking which access the user has to the cluster, "slurmctld" which will accept requests to add things to the queue, and "slurmd" which actually launches the tasks on each computation node.

Let us start with the authentication daemon. Notice that munge does not think the default option for logging is secure (!), so we must change this to using syslog before it will start.

{% highlight sh %}
sudo apt-get install -y munge
sudo /usr/sbin/create-munge-key
sudo sed -i '/# OPTIONS/aOPTIONS="--syslog"' /etc/default/munge
{% endhighlight %}

A problem with Ubuntu 14.04 is that daemons started by SysV-init files will get the cgroup of the console that started them, so we cannot boot the daemons ourselves but must let Upstart do so. By installing first and then writing the config file, we stall the startup of the daemon until we have everything ready.

{% highlight sh %}
sudo apt-get install -y slurm-llnl-slurmdbd slurm-llnl-basic-plugins slurm-llnl slurm-llnl-torque libswitch-perl
{% endhighlight %}

Configuration
-------------
Basically, the configuration file for the authorization daemon just specifies how to connect to the database (we'll add entries later):

{% highlight sh %}
sudo touch /etc/slurm-llnl/slurmdbd.conf
sudo chown slurm:slurm /etc/slurm-llnl/slurmdbd.conf
sudo chmod 660 /etc/slurm-llnl/slurmdbd.conf
sudo sh -c "cat > /etc/slurm-llnl/slurmdbd.conf" <<EOF
# logging level
ArchiveEvents=no
ArchiveJobs=no
ArchiveSteps=no
ArchiveSuspend=no

# service
DbdHost=localhost
SlurmUser=slurm
AuthType=auth/munge

# logging; remove this to use syslog
LogFile=/var/log/slurm-llnl/slurmdbd.log

# database backend
StoragePass=$DB_SLURM_PASS
StorageUser=slurm
StorageType=accounting_storage/mysql
StorageLoc=slurm_acct_db
EOF
{% endhighlight %}

The main configuration file is where we setup how the cluster should behave. Notable options here are `proctrack/cgroup` to have SLURM use a cgroup to put the jobs in, and `sched/backfill` which indicates that the system should try to run several jobs to keep full utilization.

Queues in SLURM are called "partitions". A set of nodes can be shared between partitions, or they can belong exclusively to only one. Each partition have a priority in case a node is covered by more than one.

We'll set up three partitions, which all contains the one node that is the computation server.

Limitations to the cluster in SLURM is set up using "associations", which are tuples of: (cluster, account, user, partition) with some properties attached. When we add an association, we say that the user can submit a job to this queue (partition), charging the CPU time to that project (account). Use `sacctmgr show assoc format=account,user,partition` to see the current list.

Unfortunately, associations are hierarchial in the order listed above, meaning that we cannot grant access to a partition for an entire account, but must do it per-user. From version 15.08 and out we should be able to use an orthogonal "quality of service" property to specify the priorities, but in the current version 2.6, it cannot preempt jobs into standby, only cancellation. Thus, for now, we set PreemptType to partition_prio which means that the priority of the queue is used.

To have the cluster only accept submissions to queues that have been explicitly allowed, we set `AccountingStorageEnforce=limits`.

{% highlight sh %}
sudo touch /etc/slurm-llnl/slurm.conf
sudo chown slurm:slurm /etc/slurm-llnl/slurm.conf
sudo chmod 664 /etc/slurm-llnl/slurm.conf
sudo sh -c "cat > /etc/slurm-llnl/slurm.conf" <<EOF
# identification
ClusterName=$(hostname -s)
ControlMachine=$(hostname -s)

# authentication
AuthType=auth/munge
CacheGroups=0
CryptoType=crypto/munge

# service
# proctrack/cgroup controls the freezer cgroup
SlurmUser=slurm
SlurmctldPort=6817
SlurmdPort=6818
SlurmdSpoolDir=/var/lib/slurm-llnl/slurmd
StateSaveLocation=/var/lib/slurm-llnl/slurmctld
SlurmctldPidFile=/var/run/slurmctld.pid
SlurmdPidFile=/var/run/slurmd%n.pid
SwitchType=switch/none
ProctrackType=proctrack/cgroup
MpiDefault=none
RebootProgram=/sbin/reboot

# get back on track as soon as possible
ReturnToService=2

# logging
SlurmctldLogFile=/var/log/slurm-llnl/slurmctld.log
SlurmdLogFile=/var/log/slurm-llnl/slurmd.log

# accounting
AccountingStorageType=accounting_storage/slurmdbd
AccountingStorageEnforce=limits

# checkpointing
#CheckpointType=checkpoint/blcr

# scheduling
SchedulerType=sched/backfill
PriorityType=priority/multifactor
PriorityDecayHalfLife=3-0
PriorityMaxAge=7-0
PriorityFavorSmall=YES
PriorityWeightAge=1000
PriorityWeightFairshare=0
PriorityWeightJobSize=125
PriorityWeightPartition=1000
PriorityWeightQOS=0
PreemptMode=suspend,gang
PreemptType=preempt/partition_prio

# wait 30 minutes before assuming that a node is dead
SlurmdTimeout=1800

# core and memory is the scheduling units
# task/cgroup controls cpuset, memory and devices cgroups
SelectType=select/cons_res
SelectTypeParameters=CR_Core_Memory,CR_CORE_DEFAULT_DIST_BLOCK
TaskPlugin=task/affinity,task/cgroup

# computing nodes
NodeName=$(hostname -s) RealMemory=$(grep "^MemTotal:" /proc/meminfo | awk '{print int($2/1024)}') Sockets=$(grep "^physical id" /proc/cpuinfo | sort -uf | wc -l) CoresPerSocket=$(grep "^siblings" /proc/cpuinfo | head -n 1 | awk '{print $3}') ThreadsPerCore=1 State=UNKNOWN

# partitions
PartitionName=DEFAULT  Nodes=$(hostname -s) Shared=FORCE:1 MaxTime=INFINITE State=UP
PartitionName=student  Priority=10 Default=YES
PartitionName=academic Priority=20 Default=NO
PartitionName=industry Priority=30 Default=NO
EOF
{% endhighlight %}

{% highlight sh %}
sudo touch /etc/slurm-llnl/cgroup.conf
sudo chown slurm:slurm /etc/slurm-llnl/cgroup.conf
sudo chmod 664 /etc/slurm-llnl/cgroup.conf
sudo sh -c "cat > /etc/slurm-llnl/cgroup.conf" <<EOF
CgroupMountpoint=/sys/fs/cgroup
CgroupAutomount=no
CgroupReleaseAgentDir="/etc/slurm-llnl/cgroup"
ConstrainCores=yes
#TaskAffinity=yes
ConstrainRAMSpace=yes
EOF
{% endhighlight %}

Now that we're done with the configuration and we don't need them anymore, we'll clean the passwords from the environment.

{% highlight sh %}
unset DB_ROOT_PASS
unset DB_SLURM_PASS
{% endhighlight %}

All the configuration is now done, and we are ready to start the services. On Ubuntu Trusty there is a problem that services that are not started by Upstart will get the same cgroup as the login session (which is the restrained "interactive" session in our case). Anyway, since we have made changes to the kernel configuration, this is a good time to restart the machine.

{% highlight sh %}
sudo shutdown -r now
{% endhighlight %}

If you do minor changes to the configuration file, you can use the `scontrol reconfig` command to have the daemon reread slurm.conf.

Accounting
----------

Now that the programs have been installed (and should be running), we'll add some accounting features. We only have one cluster to manage, namely our own server:

{% highlight sh %}
sudo sacctmgr -i add cluster $(hostname -s)
{% endhighlight %}

We'll then add "accounts"; accounts in SLURM parlance is really grouping of users into research groups and/or projects. Here is an example on how to add two such groups:

{% highlight sh %}
sudo sacctmgr -i add account researcher Description="Researcher" Organization="Uni"
sudo sacctmgr -i add account student Description="Student" Organization="UiB"
{% endhighlight %}

Finally, we need some users that can fill the accounts. The names of the users should be the system name that you logon with.

{% highlight sh %}
sudo sacctmgr -i create user scott defaultaccount=researcher adminlevel=Operator partition=student
sudo sacctmgr -i create user scott account=researcher partition=academic

sudo sacctmgr show user name=scott
sudo sacctmgr show assoc format=account,user,partition
{% endhighlight %}

Notice that we first create the user and give default account and partition. The next association is set up by "creating" the user once more, but with a different account and/or partition.

Testing the installation
------------------------
Use these commands to check that the cluster is live and ready to accept commands:

{% highlight sh %}
sinfo
sacct
{% endhighlight %}

To test the installation, I'll download a benchmark test, modify it slightly to run a little longer, and then submit it to a queue. Using `htop`, you can verify that it now runs with full CPU utilization.

{% highlight sh %}
sudo apt-get install -y build-essential
mkdir ~/stream
pushd ~/stream
wget http://www.cs.virginia.edu/stream/FTP/Code/stream.c
sed "s/^\(\#   define NTIMES[\ \t]\+\)\(.*\)/\11000/" stream.c >
large.c
gcc -fopenmp large.c -o large
sed "s/^\(\#   define NTIMES[\ \t]\+\)\(.*\)/\1100/" stream.c > medium.c
gcc -fopenmp medium.c -o medium
{% endhighlight %}

We'll submit a background workload to the queue with least priority:

{% highlight sh %}
cat > ~/stream/large.job <<EOF
#!/bin/bash
#SBATCH --partition=student
#SBATCH --account=researcher
#SBATCH --ntasks=1
#SBATCH --ntasks-per-node=1
#SBATCH --cpus-per-task=4
#SBATCH --mem-per-cpu=128M
#SBATCH --job-name="Large"
OMP_NUM_THREADS=\$SLURM_JOB_CPUS_PER_NODE ./large
EOF
sbatch ~/stream/large.job
{% endhighlight %}

"Task" is the term for a process, and "cpu" is used for a core. An MPI process will typically have several tasks with one cpu per task, and an OpenMP process will have one task but use several cpus per task.

We then submit a smaller job to a queue with higher priority, here shown using the PBS/TORQUE compatibility layer for illustration. Notice that the TORQUE wrapper does not set the PBS_NUM_PPN environment variable, so you will have to set the number of cores as a constant in the script.

{% highlight sh %}
cat > ~/stream/medium.job <<EOF
#!/bin/bash
#PBS -q academic
#PBS -W researcher
#PBS -l nodes=1:ppn=4
#PBS -l pmem=128MB
#PBS -N Medium
OMP_NUM_THREADS=4 ./medium
EOF
qsub ~/stream/medium.job
{% endhighlight %}

