# Backup / Disaster recovery

* https://trello.com/c/c0Cm37wO/39-disaster-recovery-emergency-procedures-backups
* https://trello-attachments.s3.amazonaws.com/5a8ef582df5995f183d1eb2b/5a8eff54bd0b46a5a303712d/7257859bd24b20005cf25d5f30100f10/Re__%5Bcaasp-devel%5D_CaaSP_Disaster_recovery.eml

## Requirements
* Must work with CaaSP v3
* Must work reliably and restore cluster to working state
* Must be complete in each direction, can not have any "if you reach this point, contact support" clauses, we must think ahead and cover it all

## Expectations

Enterprise customers expect to have a backup solution in place. They want "Disaster recovery" e.g. someone throws a bomb onto their datacenter(s) they can restore the business from some backup within the shortest possible time. That is the priority, being able to recover from unforseen events that cause your production environment to fail.

They use this to create and evaluate failover plans and "business continuitiy" to prepare for what to do in such an event. We need to provide them with the tools to do that for our software and at least somewhat understand what else needs to be done to preserve/restore connected systems. We don't need to know how to back up their databases but we need to be able to tell them how to re-connect them if we have some specialized process for that or how to reconfigure if IDs, endpoints, IPs, hostnames or whatever (the components need to function) change during restore procedures.

The second expectation is a backup / restore solution that provides a means to have this on the smaller scale for daily work. A node fails, you throw away the node and rebuild it, Kubernetes rebuilds deployments and applications while it is working properly.

What is needed is the steps when CaaSP stops working properly. E.g. hardware failure of a node causes it to drop completely from the cluster and now you need to bootstrap a new node and reconfigure.

For the first step we need to describe the worst case scenario with the most generalized solution. Backup everything, Lose everything, restore everything. Most important here is that all workload relevant information can be restored to a working state. E.g. you restore from backup, run some steps and then the cluster should act like it did before. This is especially important because external service will expect the cluster to behave in certain ways, be reachable at certain hostnames/ports etc.

Ideally, we have a solution that allows dumping the entire cluster state and just resurrecting from that state. With Kubernetes this is not really possible so we must try to get as close as possible.

## Scenarios

### Full cluster
Full cluster backup and restore. How to back up all important components and configurations, how often to do that to make sure you don't lose too much work (imagine how much money a bank loses if a stock trading app that works on a microsecond level goes down for hours or days).

Add more details here

### Component / Node based backup
Backup of individual node states and configurations to resurrect in case they die unexpectedly. Ideally even to pick up work from where the old node died etc.

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

## Unclear

* How to remove old salt minion IDs from Admin? Just click "Remove"?
* Restore with same IP/hostnames?
 * Is it possible
 * Should you do it?
* Restore using new IP/hostnames (if "last" network is still down/taken in new env)
* Restore with same container names etc?
* Restore cluster secrets
 * Should you restore all possible secrets or recreate?
 * Will customers want to have full restore including the "previous" certificates?
* How to reconfigure external dashboard and API IP after restore? (Hostnames used must be available?)
* Which services / nodes must be turned off during backup and/or restore?
* What are the individual pieces that must be backed up to give more context

## Known problems

* We have no method to fully restore an admin node to a working state
