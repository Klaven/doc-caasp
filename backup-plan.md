# Backup / Disaster recovery

* https://trello.com/c/c0Cm37wO/39-disaster-recovery-emergency-procedures-backups
* https://trello-attachments.s3.amazonaws.com/5a8ef582df5995f183d1eb2b/5a8eff54bd0b46a5a303712d/7257859bd24b20005cf25d5f30100f10/Re__%5Bcaasp-devel%5D_CaaSP_Disaster_recovery.eml

## Requirements
* Must work with CaaSP v3
* Must work reliably and restore cluster to working state
* Must be complete in each direction, can not have any "if you reach this point, contact support" clauses, we must think ahead and cover it all
* Must provide verification for each step to ensure procedures are successful (or if not enable triage/debugging)

## Expectations

Enterprise customers expect to have a backup solution in place. They want "Disaster recovery" e.g. someone throws a bomb onto their datacenter(s) they can restore the business from some backup within the shortest possible time. That is the priority, being able to recover from unforseen events that cause your production environment to fail.

They use this to create and evaluate failover plans and "business continuitiy" to prepare for what to do in such an event. We need to provide them with the tools to do that for our software and at least somewhat understand what else needs to be done to preserve/restore connected systems. We don't need to know how to back up their databases but we need to be able to tell them how to re-connect them if we have some specialized process for that or how to reconfigure if IDs, endpoints, IPs, hostnames or whatever (the components need to function) change during restore procedures.

The second expectation is a backup / restore solution that provides a means to have this on the smaller scale for daily work. A node fails, you throw away the node and rebuild it, Kubernetes rebuilds deployments and applications while it is working properly.

What is needed is the steps when CaaSP stops working properly. E.g. hardware failure of a node causes it to drop completely from the cluster and now you need to bootstrap a new node and reconfigure.

For the first step we need to describe the worst case scenario with the most generalized solution. Backup everything, Lose everything, restore everything. Most important here is that all workload relevant information can be restored to a working state. E.g. you restore from backup, run some steps and then the cluster should act like it did before. This is especially important because external service will expect the cluster to behave in certain ways, be reachable at certain hostnames/ports etc.

Ideally, we have a solution that allows dumping the entire cluster state and just resurrecting from that state. With Kubernetes this is not really possible so we must try to get as close as possible.

## Scenarios

Permanent errors on masters or workers can be worked by removing those nodes and adding fresh ones, so that's why I think we'll probably only consider those two cases (only admin, or the whole cluster).

### Full cluster
Full cluster backup and restore. How to back up all important components and configurations, how often to do that to make sure you don't lose too much work (imagine how much money a bank loses if a stock trading app that works on a microsecond level goes down for hours or days).

#### Procedure

Add more details here

### Component / Node based backup
Backup of individual node states and configurations to resurrect in case they die unexpectedly. Ideally even to pick up work from where the old node died etc.

#### Procedure

Add more details here

## Planning

### First Iteration (Fullfil UBS requirements)

- [ ] Instructions to backup full cluster
  - [ ] Backup Admin node
  - [ ] Backup `etcd`

  - [ ] Backup cluster secrets / certs
    - Exception: Cluster certs get recreated during redeployment
  - [ ] Backup node configurations
  - [ ] Instructions to verify success for each step

- [ ] Instructions to restore full cluster
  - [ ] Restore Admin node
  - [ ] Restore `etcd`
  - [ ] Restore cluster secrets / certs
  - [ ] Restore node configurations
  - [ ] Instructions to verify success for each step

### Full instructions (long term)

- [ ] Instructions to backup full cluster
- [ ] Instructions to restore full cluster
- [ ] Instructions to backup individual master node
- [ ] Instructions to backup restore master node
- [ ] Instructions to backup individual admin node
- [ ] Instructions to backup restore admin node
- [ ] Instructions to backup individual worker node
- [ ] Instructions to backup restore worker node

## Unclear questions and discussion points

* How to remove old salt minion IDs from Admin? Just click "Remove"?
* Restore with same IP/hostnames?
 * Is it possible
 * Should you do it?
* Restore using new IP/hostnames (if "last" network is still down/taken in new env)
* Restore with same container names etc?
* Restore cluster secrets
 * Should you restore all possible secrets or recreate?
 * Will customers want to have full restore including the "previous" certificates?
 * Rafa: ```I don't think it's important to restore the very same passwords.
  When restoring a backup I don't think we want to reuse the same
  passwords in A1 and A2 as long as the different passwords that A1
  and A2 have been autogenerated make sense as a whole in each of
  those machines (that is: the set of passwords and settings in A1
  make sense from A1 point of view, as well as A2).```
* How to reconfigure external dashboard and API IP after restore? (Hostnames used must be available?)
* Which services / nodes must be turned off during backup and/or restore?
* What are the individual pieces that must be backed up to give more context
- When restoring only the admin node some questions might arise:
  - The IP of the admin will change? In this case, we need to make all existing salt-minions to point to the
   new IP address of the admin node (if it's by IP, or make the DNS resolution point to the new one). In this
   case the most important thing is to keep the salt-master key, so the minions cached salt-master public key match
   the restored salt-master private key (`/etc/salt/pki/master` in the admin node).
- When restoring all the machines in the cluster (admin, and the rest):
  - The IP of the admin will change? Same case as above.
  - Do we have to accept the nodes again? If their ID change we will have to. If we restore all the admin salt-master
  configuration, and all the Minion ID's, then there's no need to re-accept them. That being said, the salt-master
  private key in the admin, and the cached salt-master public key in all the minions must match after the restore, so
  they can talk to each other and nothing changes from their perspective.
  - Do we want to keep the salt-minion ID's on all the nodes? (`/etc/salt/minion.d/minion_id.conf`,
    and probably `/etc/salt/minion.d/` in general)
- Also, maybe it's worth it separating the backup/restore procedure in two different cases: when you only have to
backup/restore the admin node (it's a SPOF, and for whatever reason it died), or if you have to restore your complete
cluster (you had a really bad HW fail and everything went south in the whole cluster).
  - **Yes, but in the second iteration**

## Known problems

* Don't have a method to backup/restore CaaSP v3 (only v4)
 * need to integrate notes from Rafa to enable backup for v3
* We have no method to fully restore an admin node to a working state
* No feedback during procedure if the steps succeeded
