---
layout: article
title: Hyperledger Fabric on k8s - 1st
lang: ko
---

# Hyperledger Fabric on k8s - 1st

 Hyperledger Fabric 관련 일을 하게되면서 docker에 대해 관심을 가지게 되었다. 그리고 docker가 굉장히 유용한 도구라는 것을 깨닳음과 동시에 k8s에 대해 알게 되었다.
2019년 현재 container orchestration 계의 *de facto standard* 이며, cloud-native operating system 이라 일컬어지는 k8s를 사용하면 더 쉽고 편하게 Hyperledger Fabric network을 관리할 수 있지 않을까?
이와 같은 물음에서 출발하여 Hyperledger Fabric을 k8s에 쉽게 배포하는 방안을 구상하고 확인하는 과정을 이 글로 보이고자 한다.

## How to deploy? And Which objects to use?

 k8s에 대해 궁금해하고, 무엇인지 확인하기 위해 한번이라도 [k8s Documentation](https://kubernetes.io/docs/home/)에 방문한 적이 있다면 그 방대하고 다양한 object들에게 압도된 적이 있을 것 같다.
그 다양한 object들을 한번에 모두 소화할 수는 ~~당연히~~ 없기에, 불 꺼진 방에서 핸드폰 찾듯이 더듬더듬 필요한 것들만 찾고 추려서 사용해보려고 한다.

### Goals

일단 내가 Hyperledger Fabric을 k8s에 배포하려는 목적에 대해 생각해 보도록 하자.

1. 관리가 편했으면 좋겠다. 그리고 기왕 k8s를 사용하는 것이니 MSA 정신에 부합하여 쉽게 scaling이 되었으면 좋겠다. --> G1. 손 쉬운 scaling
2. Hyperledger Fabric이 여러 동등한 기관이 모여 하나의 blockchain 네트워크를 이루는 것이니 이 정신도 계승되는 것이 좋겠다. --> G2. 각 Organiztion 의 독립성 유지

### Ways

#### Problem#1 - G1.scaling

k8s에서 Hyperledger Fabric의 component를 쉽게 관리할 수 있으려면 어떻게 해야 할까? 일단은 Peer Organization을 기준으로 생각해보자.
Hyperledger Fabric에서 peer들은 각자 최신의 block을 찾아 적용하도록 설계되어있다.
 즉 Gossip protocol을 통해 같은/다른 Organiztion의 블록상태를 주기적으로 확인하여 내가 가진 블록숫자보다 높은 숫자를 발견하면 해당 블록을 가져와 그 내부의 transaction들을 적용시키는 식이다.
이 부분을 뒤집어 생각하면 모든 peer가 항상 동일한 상황에 놓여있지 않으며, 한 사용자가 동시에 여러 peer에게 요청을 보냈을 때 서로 다른 응답을 받을 수 있다는 의미이다.
그렇다면 Hyperledger Fabric의 peer는 stateful application이라고 쉽게 유추해 볼 수 있다.
 엇.. 그런데 k8s에서 손쉽게 scaling을 다룰 수 있는 이유가 stateless application 이기 때문이라고 알고 있는데??
 하지만 걱정하지 마라. 전세계의 개발자들은 내가 필요로 하기 전에 자신이 필요한 것들을 이미 만들어 놓았다.
 k8s에는 [`StatefulSets`](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)라는, stateful application를 관리하기 위한 API가 이미 개발되어 있다. (v1.9 GA, v1.5 Beta)
이 API를 사용하면 stateless application 못지 않게 쉽게 scaling 관리가 가능할 것이다.

Solution#1 - Use `StatefulSets`

#### Problem#2 - G2.independency
 
