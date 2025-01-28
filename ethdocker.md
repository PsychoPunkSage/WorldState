# Eth-Docker interaction

## Container info

> Required for interaction purpose

```sh
docker inspect <container_name> --format '{{json .NetworkSettings.Networks}}'
# For Consensus & Execution client
```

<details>
<summary>
Output
</summary>

```json
// Execution
{
    "eth-docker_default": {
        "IPAMConfig": {},
        "Links": null,
        "Aliases": [
            "eth-docker-execution-1",
            "execution",
            "eth1",
            "sepolia-execution",
            "6e9ef41acabd"
        ],
        "NetworkID": "bd4096ba7199bd3d2f71af26a89f0a4bcc57b45973fbe98ff05bb845c273f394",
        "EndpointID": "327db5f83110bd04b7a2cce445c9341e705aa7150856c1e720e5fe6900f19001",
        "Gateway": "172.18.0.1",
        "IPAddress": "172.18.0.2",
        "IPPrefixLen": 16,
        "IPv6Gateway": "",
        "GlobalIPv6Address": "",
        "GlobalIPv6PrefixLen": 0,
        "MacAddress": "02:42:ac:12:00:02",
        "DriverOpts": null
    }
}

// Consensus
{
    "eth-docker_default": {
        "IPAMConfig": {},
        "Links": null,
        "Aliases": [
            "eth-docker-consensus-1",
            "consensus",
            "eth2",
            "sepolia-consensus",
            "ba3d50ccc23d"
        ],
        "NetworkID": "bd4096ba7199bd3d2f71af26a89f0a4bcc57b45973fbe98ff05bb845c273f394",
        "EndpointID": "6611f3cbdecc0a28a4c4f70b2c5434cc4799f47498ecc9f69893f796300e0e46",
        "Gateway": "172.18.0.1",
        "IPAddress": "172.18.0.3",
        "IPPrefixLen": 16,
        "IPv6Gateway": "",
        "GlobalIPv6Address": "",
        "GlobalIPv6PrefixLen": 0,
        "MacAddress": "02:42:ac:12:00:03",
        "DriverOpts": null
    }
}
```

> Important part is the `IPAddress` and `Gateway`

</details>

## Network Info (Eth-docker)

```sh
docker network ls
```

```js
NETWORK ID     NAME                 DRIVER    SCOPE
685818d8d12e   bridge               bridge    local
bd4096ba7199   eth-docker_default   bridge    local ****
fbc07f87214e   host                 host      local
1c4523aaf953   none                 null      local
```

```sh
docker network inspect eth-docker_default
```

<details>
<summary>
Output
</summary>

```json
[
    {
        "Name": "eth-docker_default",
        "Id": "bd4096ba7199bd3d2f71af26a89f0a4bcc57b45973fbe98ff05bb845c273f394",
        "Created": "2025-01-24T13:36:48.183202449Z",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.18.0.0/16",
                    "Gateway": "172.18.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {
            "6e9ef41acabd839766bd60e3080e0cdce91cb52f7eb0a33d3c7d735ac8dd817e": {
                "Name": "eth-docker-execution-1",
                "EndpointID": "327db5f83110bd04b7a2cce445c9341e705aa7150856c1e720e5fe6900f19001",
                "MacAddress": "02:42:ac:12:00:02",
                "IPv4Address": "172.18.0.2/16",
                "IPv6Address": ""
            },
            "ba3d50ccc23def1cd01d6a7bc5eab23b355120fb168cf36bbc2701f6c229847d": {
                "Name": "eth-docker-consensus-1",
                "EndpointID": "6611f3cbdecc0a28a4c4f70b2c5434cc4799f47498ecc9f69893f796300e0e46",
                "MacAddress": "02:42:ac:12:00:03",
                "IPv4Address": "172.18.0.3/16",
                "IPv6Address": ""
            }
        },
        "Options": {},
        "Labels": {
            "com.docker.compose.network": "default",
            "com.docker.compose.project": "eth-docker",
            "com.docker.compose.version": "2.24.6"
        }
    }
]
```

</details>