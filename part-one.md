
# Cluster Administration Course Part I

## Command line used during talk

### Setup a cluster with the aragodb starter 

3 Machines named `a1`, `a2` and `a3`

Install arangodb 3.3.7 on all machines

    a1,a2,a3
      echo 'deb https://download.arangodb.com/arangodb33/xUbuntu_17.04/ /' | \
          sudo tee /etc/apt/sources.list.d/arangodb.list
      sudo apt-get install apt-transport-https
      sudo apt-get update
      sudo apt-get install arangodb3=3.3.73#
    
Start the cluster

    a1
      cd db && arangodb
    a2
      cd db && arangodb --starter.join a1
    a3
      cd db && arangodb --starter.join a1

If you would like to startup with authentication, add
`--auth.jwt-secret <secret>` to the command lines. If you would like
to operate the cluster on secure communications you have a trove of
choices with `--ssl.auto-key`, `--ssl.cafile`, `--ssl.keyfile` ...

### The agency

Dump the agent's cofigurations:

    curl a1:8531/_api/agency/config
    curl a2:8531/_api/agency/config
    curl a3:8531/_api/agency/config

One of the agents will stand out by having a populated 

    {
        "term": 1,
        "leaderId": "AGNT-7237ed7e-d6e0-44e1-813d-b8430dd597b7",
        "commitIndex": 254,
        "lastAcked": {
            "AGNT-0e8d7823-276b-41ad-b466-4196ebc98cd9": 0.231,
            "AGNT-4c6d0319-dfdf-4714-98f6-0d01f769338d": 0.231,
            "AGNT-7237ed7e-d6e0-44e1-813d-b8430dd597b7": 0
        },
        "configuration": {
            "pool": {
                "AGNT-4c6d0319-dfdf-4714-98f6-0d01f769338d": "tcp://a1:4002",
                "AGNT-7237ed7e-d6e0-44e1-813d-b8430dd597b7": "tcp://a2:4001",
                "AGNT-0e8d7823-276b-41ad-b466-4196ebc98cd9": "tcp://a3:4003"
            },
            "active": [
                "AGNT-4c6d0319-dfdf-4714-98f6-0d01f769338d",
                "AGNT-7237ed7e-d6e0-44e1-813d-b8430dd597b7",
                "AGNT-0e8d7823-276b-41ad-b466-4196ebc98cd9"
            ],
            "id": "AGNT-7237ed7e-d6e0-44e1-813d-b8430dd597b7",
            "agency size": 3,
            "pool size": 3,
            "endpoint": "tcp://[::1]:4001",
            "min ping": 1,
            "max ping": 5,
            "timeoutMult": 1,
            "supervision": true,
            "supervision frequency": 1,
            "compaction step size": 2000,
            "compaction keep size": 1000,
            "supervision grace period": 5,
            "version": 3,
            "startup": "origin"
        }
    }
    
Under authentication regime, you'd have to access the agency with the
additional curl headers `-H"Authorization: bearer $(jwtgen -a HS256 -c
iss=arangodb -c server_id=startup -s <secret>)"` 
    
### Resilience

Kill any entire machine to watch the resilience take over. Follow up
on Jobs, which have handled collections affectedby that failure by
accessing the `Target` region in the agency:

    curl a1:8531/_api/agency/read -sLd'[["/arango/Target"]]'

Supervision Jobs that handled/are handling resilience are found in
sections `ToDo`, `Pending`, `Failed`, `Finished`. All jobs should
reside in `Finished` after a short while.

During the outage the cluster operations should continue.

## References

https://raft.gihub.io  
https://arangodb.com/why-arangodb/cluster  
https://arangodb.com/why-arangodb/arangodb-enterprise/arangodb-enterprise-datacenter-datacenter-replication  
