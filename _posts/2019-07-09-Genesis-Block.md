---
layout: article
title: Genesis Block @ Hyperledger Fabric
mathjax: true
---

# Internal of Genesis Block

Blockchain 의 제일 첫 블록을 *genesis block* 이라고 부르며, Hyperledger Fabric 또한 마찬가지이다.
그렇다면 Hyperledger Fabric의 genesis block에는 무슨 내용이 있을까?

## 확인에 필요한 도구

- `jq`
- `git`

## Genesis block 만들기

무슨 내용을 품었는지 확인하려면 우선 만들어 보아야 하지 않을까?
다행히 우리에게는 `fabric-samples` 가 있으니 쉽게 만들어 보자.

우선 `fabric-samples`과 binary 파일들을 받아보자.

```bash
# Download fabric-samples & binaries
curl -sSL http://bit.ly/2ysbOFE | bash -s -- -d
```

이제 ~~Hyperledger Fabric을 다루는 사람이라면 수십번 해보았을~~ `first-network` 으로 가서 Genesis block을 만들어 보자.

```bash
# Go to first-network
cd fabric-samples/first-network
# Generate a genesis block
./byfn.sh generate -c testchannel
```

## Genesis block 내용 확인하기

`configtxgen` tool을 사용하면 Genesis block에 있는 내용을 parsing하여 json 형식으로 보여준다.
이 내용을 계속 확인해야하니 json 파일로 저장하여 확인해보자.

```bash
# Parse genesis.block
../bin/configtxgen -inspectBlock ./channel-artifacts/genesis.block | jq -r > ./channel-artifacts/genesis.json
```

### Genesis block 내용 확인하기 - depth 1

우선 가장 윗 부분을 살펴보면, 아래와 같다.

```
{
    "header": {...},
    "data": {...},
    "metadata": {...}
}
```

#### `header` 에는 어떤 값이 있을까?

```bash
# Get 'header' data
jq '.header' ./channel-artifacts/genesis.json
{
  "data_hash": "PDUJ8J0akd8uYVw9Zr7xWIDglDVkkTaOmR6Vgqo3zng=",
  "number": "0",
  "previous_hash": null
}
```

- `number` : 현재 블록의 번호 (genesis block이니 값은 `0`)
- `data_hash` : block의 내용인 `data` 부분의 hash 값
- `previous_hash` : Hyperledger Fabric을 blockchain으로 만들어 주는 이전 block의 hash 값(genesis block이니 값이 `null`)

#### `metadata` 에는 어떤 값이 있을까?

```bash
# Get 'metadata' data
jq '.metadata' ./channel-artifacts/genesis.json
{
  "metadata": [
    "",
    "",
    "",
    ""
  ]
}
```

보이다시피 아무것도 없다. 사실 이 부분은 block을 생성한 자의 identity 정보 및 signature 등이 들어가는 자리이지만 genesis block이기에 아무 것도 없다. 왜냐하면 genesis block은 orderer가 아니라 사용자가 직접 만들기 때문이다.

#### `data` 에는 어떤 값이 있을까?

```bash
# Get 'data' data
jq '.data' ./channel-artifacts/genesis.json
{
    "data": [
        {
            "payload": {...},
            "signature": null
        }
    ]
}
```

- `payload` : 실제 data가 들어가는 부분
- `signature` : 문자 그대로 signature가 들어갈 부분이지만 사용자가 직접 만들기 때문에 `null`

### Genesis block 내용 확인하기 - depth 2

#### `payload` 를 살펴보자

```bash
# Get 'payload.header' data
jq '.data.data[].payload.header' ./channel-artifacts/genesis.json
{
  "channel_header": {
    "channel_id": "byfn-sys-channel",
    "epoch": "0",
    "extension": null,
    "timestamp": "2019-07-09T03:43:59Z",
    "tls_cert_hash": null,
    "tx_id": "93428e403eedb19fd4dff8e9a2c72ffb51dad8c1f091692247e920a259ab791e",
    "type": 1,
    "version": 1
  },
  "signature_header": {
    "creator": null,
    "nonce": "O2ZyieHUv/OKEH0pvxsxiwXIQQ/aWOzn"
  }
}
```

- `channel_header.channel_id` : Orderer들이 사용하는 channel의 이름 (`configtxgen`으로 직접 genesis block을 생성하면 이름을 지정할 수 있다.)
- `channel_header.epoch` : ~~언제 사용될 지 모르는~~ 현재는 무조건 0인 부분
- `channel_header.extension` : extension에 사용되는 부분 (`channel_header.type`에 따라 다름)
- `channel_header.tls_cert_hash` : mutual TLS를 사용할 때, client certificate의 hash값이 들어가는 부분
- `channel_header.type` : data의 type 값 (자세한 내용은 `github.com/hyperledger/fabric/protos/common/common.proto`에서 확인 가능)
- `signature_header.nonce` : replay attack을 막기위해 들어가는 random 값

### Genesis block 내용 확인하기 - more depth

#### `data`를 더 파보자

`data` 부분에는 크게 Consortium과 Orderer의 구성정보가 들어가 있다.
다 보려면 너무 많으니 중복되는 부분은 빼고 ~~내가 생각하는~~ 중요한 부분만 보도록 하자.

##### `channel_group`

```bash
# Get 'channel_group' data
jq '.data.data[].payload.data.config.channel_group' ./channel-artifacts/genesis.json
{
    "groups": {...},
    "mod_policy": "Admins",
    "policies": {...},
    "values": {...},
    "version": 0
}
```

- `groups` : channel을 생성할 때 사용되는 consortium의 정보가 저장되는 부분 (아래에서 좀더 자세히 알아보자)
- `mod_policy` : channel_group 부분을 수정할 권한을 가진 policy가 지정되는 부분 (현재는 `Admins` 이며, 내용은 아래에서..)
- `policies` : 어떤 policy 들이 있는지 저장되는 부분
- `values` : 이런저런 값들(?)이 들어가는 부분

##### `channel_group.values`

```bash
jq -r '.data.data[].payload.data.config.channel_group.values' ./channel-artifacts/genesis.json
{
  "BlockDataHashingStructure": {
    "mod_policy": "Admins",
    "value": {
      "width": 4294967295
    },
    "version": "0"
  },
  "Capabilities": {
    "mod_policy": "Admins",
    "value": {
      "capabilities": {
        "V1_3": {}
      }
    },
    "version": "0"
  },
  "HashingAlgorithm": {
    "mod_policy": "Admins",
    "value": {
      "name": "SHA256"
    },
    "version": "0"
  },
  "OrdererAddresses": {
    "mod_policy": "/Channel/Orderer/Admins",
    "value": {
      "addresses": [
        "orderer.example.com:7050"
      ]
    },
    "version": "0"
  }
}
```

- `BlockDataHashingStructure` : Merkle tree 사용 시 width 값을 지정하는 부분(이지만 Hyperledger Fabric에서는 Merkle tree를 아직 사용하지 않음)
- `Capabilities` : `configtx.yaml` 파일에서 ChannelCapabilities 부분이 바로 여기 저장됨
- `HashingAlgorithm` : hash 값을 구할 때 사용할 algorithm (Hyperledger Fabric에서는 SHA256만 사용함)
- `OrdererAddresses` : `configtx.yaml` 파일에서 `Orderer.Addresses` 부분이 이곳에 저장됨

##### `channel_group.policies`

```bash
jq -r '.data.data[].payload.data.config.channel_group.policies' ./channel-artifacts/genesis.json
{
  "Admins": {
    "mod_policy": "Admins",
    "policy": {
      "type": 3,
      "value": {
        "rule": "MAJORITY",
        "sub_policy": "Admins"
      }
    },
    "version": "0"
  },
  "Readers": {
    "mod_policy": "Admins",
    "policy": {
      "type": 3,
      "value": {
        "rule": "ANY",
        "sub_policy": "Readers"
      }
    },
    "version": "0"
  },
  "Writers": {
    "mod_policy": "Admins",
    "policy": {
      "type": 3,
      "value": {
        "rule": "ANY",
        "sub_policy": "Writers"
      }
    },
    "version": "0"
  }
}
```

위에 보이다시피 Hyperledger Fabric v1.2 이후 부터 추가된 ACL 기능에서 사용하는 policy가 저장되어 있다. 여기에 저장된 ACL의 path는 `/Channel/<PolicyName>` 이다.
각 policy의 구조는 동일하므로 `Readers` 만 살펴보도록 하자.
- `Readers.mod_policy` : `Readers` policy를 수정할 권한을 가진 policy
- `Readers.policy.type` : 이 policy의 type(`3` 이면 `ImplicitMeta` policy, 자세한 내용은 `github.com/hyperledger/fabric/protos/common/policies.pb.go`에서 확인가능)
- `Readers.policy.value.rule` && `Readers.policy.value.sub_policy` : ImplicitMeta policy의 내용이 들어가는 부분 (`ANY Readers`라는 것은 누구나 확인가능)

##### `channel_group.groups`

```bash
jq -r '.data.data[].payload.data.config.channel_group.groups' ./channel-artifacts/genesis.json
{
    "Consortiums": {...},
    "Orderer": {...}
}
```

Consorium과 Orderer 정보를 담고 있을 수 밖에 없다.

##### `channel_group.groups.Orderer`

```bash
jq -r '.data.data[].payload.data.config.channel_group.groups.Orderer' ./channel-artifacts/genesis.json
{
    "groups": {...},
    "mod_policy": "Admins",
    "policies": {...},
    "values": {...},
    "version": 0
}
```

- `groups` : Orderer를 구성하고 있는 MSP 정보가 저장되는 부분
- `mod_policy` : 설명은 위와 같으므로 앞으로는 넘어가도록 하자
- `policies` : 위의 `channel_group.policies`와 구조가 같으니 넘어가도록 하자
- `policies.BlockValidation` : Orderer에만 적용되는 특수 policy이며, 이 부분을 만족해야 block으로써 인정받는다.
- `values` : `configtx.yaml` 내에 Orderer 관련 설정값이 들어가는 부분

##### `channel_group.groups.Orderer.values`

```bash
jq -r '.data.data[].payload.data.config.channel_group.groups.Orderer.values' ./channel-artifacts/genesis.json
{
  "BatchSize": {
    "mod_policy": "Admins",
    "value": {
      "absolute_max_bytes": 103809024,
      "max_message_count": 10,
      "preferred_max_bytes": 524288
    },
    "version": "0"
  },
  "BatchTimeout": {
    "mod_policy": "Admins",
    "value": {
      "timeout": "2s"
    },
    "version": "0"
  },
  "Capabilities": {
    "mod_policy": "Admins",
    "value": {
      "capabilities": {
        "V1_1": {}
      }
    },
    "version": "0"
  },
  "ChannelRestrictions": {
    "mod_policy": "Admins",
    "value": null,
    "version": "0"
  },
  "ConsensusType": {
    "mod_policy": "Admins",
    "value": {
      "metadata": null,
      "migration_context": "0",
      "migration_state": "MIG_STATE_NONE",
      "type": "solo"
    },
    "version": "0"
  }
}
```

- `Batchsize`, `BatchTimeout`, `Capabilities`는 많이 보던 것들이니 pass
- `ConsensusType.value.migration*` : Kafka에서 RAFT로 변경할 때 사용되는 부분
- `ChannelRestrictions` : `configtx.yaml` 파일에서 `Orderer.MaxChannels` 부분이 이곳에 저장됨

##### `channel_group.groups.Consortiums`

```bash
jq -r '.data.data[].payload.data.config.channel_group.groups.Consortiums' ./channel-artifacts/genesis.json
{
    "groups": {...},
    "mod_policy": "/Channel/Orderer/Admins",
    "policies": {...},
    "values": {},
    "version": "0"
}
```

- `mod_policy` : `/Channel/Orderer/Admins` 라는 값으로 되어 있다. 이는 `configtx.yaml` 에 `Orderer.Policies.Admins`를 만족해야 이 부분을 수정할 수 있다는 의미이다.

##### `channel_group.groups.Consortiums.groups`

```bash
jq -r '.data.data[].payload.data.config.channel_group.groups.Consortiums.groups' ./channel-artifacts/genesis.json
{
    "SampleConsortium": {...}
}
```

`configtx.yaml`에서 `Profile` 부분에서 지정되는 consortium 관련 정보가 이곳에 저장된다.

##### `channel_group.groups.Consortiums.groups.SampleConsortium`

```bash
jq -r '.data.data[].payload.data.config.channel_group.groups.Consortiums.groups.SampleConsortium' ./channel-artifacts/genesis.json
{
    "groups": {...},
    "mod_policy": "/Channel/Orderer/Admins",
    "policies": {},
    "values": {
        "ChannelCreationPolicy": {
            "mod_policy": "/Channel/Orderer/Admins",
            "value": {
                "type": 3,
                "value": {
                    "rule": "ANY",
                    "sub_policy": "Admins"
                }
            },
            "version": "0"
        }
    },
    "version": "0"
}
```

- `groups` : Consortium을 구성하고 있는 Member Organization의 정보가 저장됨
- `values.ChannelCreationPolicy` : `channel create` 요청을 할 수 있는 권한을 지정한 부분 (현재 값에 의하면 각 Org의 아무 admin이 할 수 있음)

##### `channel_group.groups.Consortiums.groups.SampleConsortium.groups`

```bash
jq -r '.data.data[].payload.data.config.channel_group.groups.Consortiums.groups.SampleConsortium.groups' ./channel-artifacts/genesis.json
{
    "Org1MSP": {...},
    "Org2MSP": {...}
}
```

보이다시피 Consortium을 구성하고 있는 Member Organization의 정보가 있다.
각 Member Organization 정보의 구조는 동일하니 하나만 살펴보자.

##### `channel_group.groups.Consortiums.groups.SampleConsortium.groups.Org1MSP`

```bash
jq -r '.data.data[].payload.data.config.channel_group.groups.Consortiums.groups.SampleConsortium.groups.Org1MSP' ./channel-artifacts/genesis.json
{
    "groups": {},
    "mod_policy": "Admins",
    "policies": {...},
    "values": {...},
    "version": "0"
}
```

특별한 내용은 없다. 그러니 `values` 부분을 보도록 하자.

##### `channel_group.groups.Consortiums.groups.SampleConsortium.groups.Org1MSP.values`

```bash
jq -r '.data.data[].payload.data.config.channel_group.groups.Consortiums.groups.SampleConsortium.groups.Org1MSP.values' ./channel-artifacts/genesis.json
{
    "MSP": {
        "mod_policy": "Admins",
        "value": {
            "config": {
                "admins": ["..."],
                "crypto_config": {
                    "identity_identifier_hash_function": "SHA256",
                    "signature_hash_family": "SHA2"
                },
                "fabric_node_ous": {
                    "client_ou_identifier": {
                        "certificate": "...",
                        "organizational_unit_identifier": "client"
                    },
                    "enable": true,
                    "peer_ou_identifier": {
                        "certificate": "...",
                        "organizational_unit_identifier": "peer"
                    }
                },
                "intermediate_certs": [],
                "name": "Org1MSP",
                "organizational_unit_identifiers": [],
                "revocation_list": [],
                "root_certs": ["..."],
                "signing_identity": null,
                "tls_intermediate_certs": [],
                "tls_root_certs": ["..."]
            },
            "type": 0
        },
        "version": "0"
    }
}
```

~~드디어 마지막 부분이다.~~
다른 부분은 이전에 설명된 부분으로 대부분 가늠할 수 있을 것이라고 보고, `MSP.value` 부분을 살펴보고자 한다.
- `MSP.value.type` : MSP의 type을 설정하는 부분(`0` 이니 `FABRIC` type 임)
- `MSP.value.config.admins` : 해당 Organization의 Admin 의 certificate 정보를 모아두는 부분
- `MSP.value.config.crypto_config` : identity나 signature 관련 hash algorithm이 지정되는 부분
- `MSP.value.config.fabric_node_ous` : identity classification 사용과 관련된 정보다 들어가는 부분(자세한 내용은 [Membership Service Provider](https://hyperledger-fabric.readthedocs.io/en/release-1.4/msp.html)에서 확인)
- `MSP.value.config.intermediate_certs` : intermediate CA 사용 시 관련 certificate가 저장되는 부분
- `MSP.value.config.name` : MSP 의 이름
- `MSP.value.config.organizational_unit_identifiers` : Organizational unit identifier가 사용될 때 들어가는 부분(자세한 내용은 [Membership Service Provider](https://hyperledger-fabric.readthedocs.io/en/release-1.4/msp.html)에서 확인)
- `MSP.value.config.revocation_list` : certificate revocation list 정보가 저장되는 부분
- `MSP.value.config.root_certs` : Root CA의 certificate가 저장되는 부분
- `MSP.value.config.tls*` : TLS 관련 certificate들이 저장되는 부분

## 결론

지금까지 genesis block의 내용에 대해 한번 훑어 보았다.
개인적으로 genesis block의 내용을 보고나면, `configtx.yaml` 이나 public certificate 파일들이 어떤 식으로 blockchain 에 전달되는지 확인할 수 있고, 향후 update 작업(Org 추가, channel 추가, consortium 변경 등)을 할 때 '왜' 이렇게 진행되는지 쉽게 이해할 수 있게 된다고 생각한다.
쉽게 한번 보고, 나중에 필요할 때 더 자세히, 더 정확히 확인할 수 있는 기본정보가 되었기를 바란다.
