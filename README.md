# EDA-Scenarios

Requirements:

1, All rulebooks and playbooks in the repository are developed for using under Red Hat Ansible Automation Platform(AAP) 2.5+;
   1) rulebooks would run as Rulebook Activations on AAP;
   2) playbooks would run as Job Templates on AAP;

2, Please create and enable Event Stream on AAP at first, then can attach each rulebook activations to the Event Stream;

3, Please make sure collection "community.general" installed in your project dirctory(Manual) by running:

        # ansible-galaxy collection install community.general -p collections
        
   or build a new execution environment which includes community.general

4, Please install sysstat, iotop, perf on each of your managed Linux servers;

