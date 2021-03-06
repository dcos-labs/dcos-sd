# A collection of service discovery utils for DC/OS

DC/OS comes with a set of powerful yet un-opinionated [service discovery](https://dcos.io/docs/1.7/usage/service-discovery/) mechanisms. While powerful, it can sometimes be hard for newcomers to quickly get results, so here a few tips and tricks in the following.

## Prerequisites 

1. Many shell commands below assume you've got [jq](https://stedolan.github.io/jq/download/) installed.
2. All bash shell commands below assume that you've added your private SSH key already; if you haven't done this yet do the following: `$ ssh-add ~/.ssh/MYPRIVATEKEY`

## 1. Using the DC/OS command line interface

### IP address of node a  Marathon app is running on

Let's say you have a Marathon app with the app ID `/example` and want to know on the public routable IP address it is running (if it's on a public node):

    $ export APP_NODE=$(echo "curl -s ifconfig.ca" | dcos node ssh --master-proxy --mesos-id=$(dcos task --json | jq --raw-output '.[] | select(.name == "example") | .slave_id') 2>/dev/null)

### Listing public nodes

To get the cluster-internal IP address of the public node:

    $ export PUBLIC_NODE_INTERNAL_IP=$(dcos node --json | jq "map(select(.attributes.public_ip).hostname)" | grep \" | cut -d"\"" -f 2)

To get the public Internet-routable IP address of the public node:

    $ export PUBLIC_NODE_PUBLIC_IP=$(echo "curl -s ifconfig.ca" | dcos node ssh --master-proxy --mesos-id=$(dcos node --json | jq --raw-output '.[] | select(.reserved_resources.slave_public != null) | .id') 2>/dev/null)

### Manual service discovery

To get the (cluster-internal) IP address and port of a Marathon app:

    $ export APP_ENDPOINT=$(dcos marathon task list --json | jq "map(select(.appId==\"/peek\").host, select(.appId==\"/peek\").ports)") 

Using the same base command, manual:

    $ dcos marathon task list
    APP       HEALTHY  STARTED                   HOST       ID
    /example  True     2016-04-22T13:00:57.458Z  10.0.3.34  example.3459106a-088a-11e6-ba3b-9eefd772c507
    
    $ dcos marathon task show example.3459106a-088a-11e6-ba3b-9eefd772c507 | jq .host,.ports
    "10.0.3.34"
    [
      31105
    ]

Or, alternatively, using a different sub-command:

    $ export APP_ENDPOINT=$(dcos marathon app show peek | jq .tasks[0].host,.tasks[0].ports[0])

## 2. Using dig

Assuming you want a Marathon app with the ID `/example` and you have `dig` installed. Then you can—from within the cluster, that is, after SSHing into a node—do the following:

    $ app_port=$(dig _peek._tcp.marathon.mesos SRV | grep ^_peek._tcp.marathon.mesos. | cut -d " " -f 4)
    $ app_ip=$(dig _peek._tcp.marathon.mesos SRV | grep ^$(dig _peek._tcp.marathon.mesos SRV | grep ^_peek._tcp.marathon.mesos. | cut -d " " -f 5) | cut -d " " -f 3 | cut -d$'\t' -f 3)
    $ export APP_ENDPOINT=$app_ip:$app_port


## 3. Programming Languages

### In Go

    type SRVRecord struct {
    	Service string
    	Host    string
    	IP      string
    	Port    string
    }

    func serviceDiscovery(lookup string) string {
    	mesosdns := "http://leader.mesos:8123" // the Mesos-DNS HTTP API within the cluster
    	query := "_" + lookup + "._tcp.marathon.mesos."
    	resp, err := http.Get(mesosdns + "/v1/services/" + query)
    	if err != nil {
    		log.Fatalf("Can't look up my address due to %v", err)
    	}
    	defer resp.Body.Close()
    	body, err := ioutil.ReadAll(resp.Body)
    	if err != nil {
    		log.Fatalf("Error reading response from Mesos-DNS due to %v", err)
    	}
    	var srvrecords []SRVRecord
    	err = json.Unmarshal(body, &srvrecords)
    	if err != nil {
    		log.Fatalf("Error decoding JSON object due to %v", err)
    	}
    	return "http://" + srvrecords[0].Host + ":" + srvrecords[0].Port
    }

### In JavaScript

* [front-end of the m-shop](https://github.com/mhausenblas/m-shop/blob/master/frontend-static/content/m-shop.js): See the example.
* [mesosdns-cli](https://www.npmjs.com/package/mesosdns-cli): A CLI for querying Mesos DNS service names.
* [mesosdns-client](https://www.npmjs.com/package/mesosdns-client): A simple client for Mesos DNS which makes it possible to receive actual endpoints from Mesos service names.
* [mesosdns-http-agent](https://www.npmjs.com/package/mesosdns-http-agent):  A http.Agent which makes it possible to consume Mesos service resources by simply using the Mesos DNS service names.

### In Bash scripting

* [tobilg/mesosdns-resolver](https://github.com/tobilg/mesosdns-resolver): A bash script to resolve Mesos DNS SRV records to actual host:port endpoints

### A Python snippet

See the [mc](https://github.com/mhausenblas/mc) project.
