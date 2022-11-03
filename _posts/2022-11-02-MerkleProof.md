---
layout: post
title:  "스마트 컨트렉트에서의 Merkle Proof 사용"
date:   2022-11-02 19:03:31 +0900
categories: Smart Contract
---

### Merkle Proof
특정한 주소 리스트들만 함수를 실행할 수 있도록 스마트 컨트렉트를 작성하려면 어떻게 해야할까? 이더리움의 저장 공간과 비용에 제약이 없다면 모든 주소 리스트들을 Storage에 저장하는 것이 하나의 방법일 것이다. 하지만 이더리움 네트워크는 수 많은 컴퓨터가 상태를 공유하는 월드 컴퓨터로서, 이러한 월드 컴퓨터에 데이터를 저장하는 것은 많은 비용이 소모된다 (32 bytes의 word를 저장하는 것은 통상적으로 20,000 gas를 소모, 글쓴 시점 기준 약 8천원).

우리는 Merkle Proof를 통해서 모든 주소 데이터를 Storage에 저장하지 않고도 해당 주소 데이터가 함수 실행 허용 주소 목록에 들어가는지 아닌지를 판단할 수 있다. 즉 모든 데이터 (여기선 주소 리스트)를 이더리움 온체인에 저장하지 않고도 특정 데이터의 무결성(Data integrity)을 검증할 수 있다.  

###  Merkle Tree
Merkle Proof는 머클 트리(Merkle Tree)를 사용한다. 아래 그림과 같이 머클 트리의 Leaf 노드는 데이터의 해시 값으로 구성되어 있으며,Leaf 노드가 아닌 모든 노드들은 자식 노드 둘의 해시 값을 가진다. Leaf 노드의 데이터 값이 하나라도 변경된다면, 해시 함수의 결과 값이 계층마다 연쇄적으로 달라지기 때문에 Root node의 해시 값이 완전히 달라지게 된다. 우리는 이 Root Hash 값만을 이더리움 온체인에 저장함으로써 특정 데이터가 해당 머클 트리에 포함되어 있는지를 검증할 수 있다. 예시를 들어 보자.

위 그림에서 Leaf 노드 C가 해당 머클 트리에 포함된다는 것을 스마트 컨트렉트에서 어떻게 증명할 수 있을까? 여기서 해당 머클 트리의 Root Hash 값은 온체인에 저장되어 있다고 가정해보자. 우리는 아래 그림에 포함된 것처럼, C 값과 함께 Root Hash 값을 구하기 위한 다른 해시 값을 포함하여 [H(A-B), C, D, H(E-H)]의 형태로 스마트 컨트렉트에 전달할 수 있다. 스마트 컨트렉트에서는 C와 D의 해시 값을 계산하고 (즉 H(C-D)를 도출), 이를 H(A-B)와 해싱한다. 마지막으로 이를 H(E-H)와 해싱함으로써 최종적인 Root Hash 값 H(A-H)를 구할 수 있다. 이렇게 계산된 H(A-H) 값이 미리 온체인에 저장되어 있던 H(A-H)값과 일치한다면 우리는 노드 C가 머클 트리에 포함되었다고 판단할 수 있다. 해당 과정에선 해시 데이터의 순서를 무시하였지만, H(H(A-B) + H(C-D))의 값은 H(H(C-D) + H(A-B)) 값과 전혀 다르기 때문에 실제 구현에선 순서(order)에 유의해야한다.

Merkle Proof는 위에서 살펴봤듯이 특정 데이터가 머클 트리에 속해있는지를 검증하는 방법이다. 스마트 컨트렉트에서는 스토리지에 머클 트리의 Root Hash 값을 미리 저장하고, 추후 사용자가 데이터가 해당 머클 트리에 속해있는지 증명하기 위해 Proof를 전달하면 된다. 여기서 Proof에는 모든 데이터를 포함 할 필요 없이, Root Hash를 구성하기 위해 Root 노드로 가는 경로상에 있는 특정 데이터만 제공하면 된다 (앞선 예시에서 C의 Proof는 모든 데이터 A,...,H 가 아닌 H(A-B), C, D, H(C-D)만 필요하였다).   

https://medium.com/crypto-0-nite/merkle-proofs-explained-6dd429623dc5

### OpenZeppelin의 Merkle Proof 라이브러리

```solidity
function testDoubleWithFuzzing(uint256 x) public {
  foo.set(x);
  require(foo.x() == x);
  foo.double();
  require(foo.x() == 4 * x);
}
```

위에서 x에 자동으로 여러 값을 대입해 테스트하고 에러가 발생할 시 카운터 예제 값인 x를 아래와같이 보여줍니다.

```
[FAIL. Counterexample: calldata=0x44735ef10000000000000000000000000000000000000000000000000000000000000001, args=[Uint(1)]] testDoubleWithFuzzingCounterExample (gas: [fuzztest])
```

이 외에도, VM 상태 값을 변경하는 기능과 라이브 네트워크의 상태를 반영한 테스트 기능들을 수행할 수 있습니다. 아래 예제는 block timestamp 값을 100으로 변경하여 테스트하는 예제입니다.

```solidity
address constant CHEATCODE_ADDRESS = 0x7cFA93148B0B13d88c1DcE8880bd4e175fb0DeDF;

interace Vm {
  // Sets the block.timestamp to `x`.
  function warp(uint256 x) external;
}

contract MyTest {
  Vm vm = Vm(CHEATCODE_ADDRESS);

  function testWarp() public {
    vm.warp(100);
    require(block.timestamp == 100);
  }
}
```
위에 코드에서 보듯이, wrap() 함수를 통해 block timestamp 값을 변경하여 테스트가 가능합니다. 이 외에도 block diffculty, block number, slot 값 변경 및 읽기 등 다양한 기능들을 제공하며 더 자세한 사항은 [Readme]를 참고하시면 됩니다. 다른 개발툴처럼, 실시간 네트워크와의 연동은 node URL을 명시해줌으로써 쉽게 가능합니다.  

```
forge test --fork-url <your node url> [--fork-block-number <the block number you want>].
```

이처럼 다양한 기능들을 지원하고 있기 때문에 많은 프로젝트들이 점차적으로 Typescript와 더불어 Foundry를 통한 테스트 코드를 작성하는 추세입니다. 저 같은 경우 TreasureDAO가 해당 툴을 통해 디버깅하는 것을 보고 접하게 되었습니다. 그럼 이제 어떻게 설치하는지 살펴봅시다.

### Foundry 설치
[Foundry]의 Github에서 설치 관련 내용을 확인할 수 있습니다. 저는 Window의 WSL2의 우분투환경이므로 아래 명령어를 통해 설치해보도록 하겠습니다.

`curl -L https://foundry.paradigm.xyz | bash`

docker를 통해 설치할 경우엔 아래와 같은 명령어를 입력해주면 됩니다. foundry의 도커 이미지를 통한 자세한 사용법은 [docker 섹션]에서 확인할 수 있습니다.

`docker pull ghcr.io/foundry-rs/foundry:latest`  

저처럼 Curl을 통해 다운로드할 경우 아래와 같은 결과가 나옵니다.

<p style="text-align: center;">
	<img src="{{ site.url }}/assets/images/foundry/foundry_install.png" alt="Drawing" style="max-width: 80%; height: auto;"/>
</p>

메시지에 나온 것처럼, 추가적으로 `source /home/사용자/.bashrc`를 입력해준 후 `foundryup` 명령어를 입력해줍시다. 그럼 아래와 같이 정상적으로 설치가 완료됩니다.

<p style="text-align: center;">
	<img src="{{ site.url }}/assets/images/foundry/foundry_install2.png" alt="Drawing" style="max-width: 80%; height: auto;"/>
</p>

설치가 완료된 후에는 `forge`, `cast`, `anvil` 명령어를 command 창에 입력하여 help 사용법을 확인할 수 있습니다. 아래는 `forge`를 입력했을 때 출력되는 help 메시지들입니다.

<p style="text-align: center;">
	<img src="{{ site.url }}/assets/images/foundry/forge_help.png" alt="Drawing" style="max-width: 80%; height: auto;"/>
</p>

### Gas Optimization (Assembly)

위의 OpenZeppelin이 작성한 스마트 컨트렉트 코드는 어셈블리를 사용하지 않고 동작합니다. 대부분의 프로젝트들은 해당 라이브러리를 사용하는 것을 추천드립니다. 다만 Gas 비용을 줄이기 위해서 어셈블리(Assembly)를 사용하여 같은 동작을 수행할 수 있습니다. Art Gobblers라는 NFT 프로젝트의 컨트렉트를 예시로 살펴보겠습니다.  

### Reference
1. [Ethereum Foundation's Blog Post for Merkle Proof]
2. [OpenZeppelin's Github for Merkle Tree]

[Merkle Tree]: https://en.wikipedia.org/wiki/Merkle_tree
[MERKLE PROOFS FOR OFFLINE DATA INTEGRITY]: https://ethereum.org/en/developers/tutorials/merkle-proofs-for-offline-data-integrity/
[OpenZeppelin]: https://github.com/OpenZeppelin/merkle-tree
[OpenZeppelin_MerkleProof]: https://docs.openzeppelin.com/contracts/4.x/api/utils#MerkleProof
[Art Gobblers]: https://github.com/artgobblers/art-gobblers/blob/master/src/ArtGobblers.sol#L347
[Solmate]: https://github.com/transmissions11/solmate
[Belavadi Prahalad's Medium]: https://medium.com/crypto-0-nite/merkle-proofs-explained-6dd429623dc5
