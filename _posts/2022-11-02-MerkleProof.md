---
layout: post
title:  "스마트 컨트렉트에서의 Merkle Proof 사용"
date:   2022-11-02 19:03:31 +0900
categories: Smart Contract
---

### Introduction
NFT 프로젝트에서 화이트 리스트 명단만 NFT 민팅이 가능하게 만드는 시나리오처럼, 특정한 주소 리스트들만 함수를 실행할 수 있도록 스마트 컨트렉트를 작성하려면 어떻게 해야할까? 이더리움의 저장 공간과 비용에 제약이 없다면 모든 주소 리스트들을 Storage에 매핑 형태로 저장한 후, 트랜잭션 요청 시 msg.sender가 매핑 값을 갖는지 확인하는 것이 하나의 방법일 것이다. 하지만 이더리움 네트워크는 수 많은 컴퓨터가 상태를 공유하는 월드 컴퓨터로서, 이러한 월드 컴퓨터에 데이터를 저장하는 것은 많은 비용이 소모된다 (32 bytes의 word를 저장하는 것은 통상적으로 20,000 gas를 소모, 글쓴 시점 기준 약 8천원).

우리는 Merkle Proof를 통해서 모든 주소 데이터를 Storage에 저장하지 않고도 해당 주소 데이터가 함수 실행 허용 주소 목록에 들어가는지 아닌지를 판단할 수 있다. 즉 모든 데이터 (여기선 주소 리스트)를 이더리움 온체인에 저장하지 않고도 특정 데이터의 무결성(Data integrity)을 검증할 수 있다.  

### Merkle Proof and Merkle Tree
Merkle Proof는 머클 트리(Merkle Tree)를 사용한다. 아래 그림과 같이 머클 트리의 Leaf 노드는 데이터의 해시 값으로 구성되어 있으며, Leaf 노드가 아닌 모든 노드들은 자식 노드 둘의 해시 값을 가진다. Leaf 노드의 데이터 값이 하나라도 변경된다면, 해시 함수의 결과 값이 계층마다 연쇄적으로 달라지기 때문에 Root node의 해시 값이 완전히 달라지게 된다. 우리는 이 Root Hash 값만을 이더리움 온체인에 저장함으로써 특정 데이터가 해당 머클 트리에 포함되어 있는지를 검증할 수 있다. 예시를 들어 보자.

<p style="text-align: center;">
	<img src="{{ site.url }}/assets/images/MerkleProof/tree.png" alt="Drawing" style="max-width: 80%; height: auto;"/>
</p>

위 그림에서 Leaf 노드 H(K)가 해당 머클 트리에 포함된다는 것을 스마트 컨트렉트에서 어떻게 증명할 수 있을까? 여기서 해당 머클 트리의 Root Hash 값 H(ABCDEFGHIJKLMNOP)은 온체인에 저장되어 퍼블릭하게 공개되어 있다고 가정해보자. 우리는 위 그림에서 색칠된 노드들만 있다면 해시 계산을 통해 Root Hash 값을 도출할 수 있다. 계산은 다음과 같다.

1. H(K)을 H(L)와 해싱하여 H(KL)을 계산
2. H(KL)을 H(IJ)와 해싱하여 H(IJKL)을 계산
3. H(IJKL)을 H(MNOP)와 해싱하여 H(IJKLMNOP)을 계산
4. H(IJKLMNOP)을 H(ABCDEFGH)와 해싱하여 Root Hash인 H(ABCDEFGHIJKLMNOP)을 계산  

따라서 H(K)로부터 Root Hash 값을 구하기 위해선 [ H(K), H(L), H(IJ), H(MNOP), H(ABCDEFGH)] 값들이 필요하다. 위 과정을 통해 계산된 Root Hash 값이 미리 온체인에 저장되어 있던 Root Hash 값과 일치한다면 우리는 Leaf H(K)가 H(ABCDEFGHIJKLMNOP)를 Root Hash로 가지는 머클 트리에 포함되었다고 판단할 수 있다.

Warning: 해당 과정에선 해시함수 파라미터들의 순서(order)를 무시하였지만, H(H(KL), H(IJ))의 값은 H(H(IJ), H(KL))) 값과 전혀 다르기 때문에 실제 구현에선 파라미터의 순서 혹은 Concat 순서에 유의해야한다. 아래 살펴볼 OpenZeppelin 라이브러리에서는 H(IJ)와 H(KL)의 값의 크기를 비교하여 정렬한다. 즉 H(IJ)가 H(KL)보다 클 때, H(concat(H(IJ),H(KL)))을 수행하고, 반대의 경우엔 H(concat(H(KL),H(IJ)))를 수행한다.    

위의 예제처럼 특정 데이터(A to P)들의 집합이 있을 때, 우리는 이 데이터들의 해시 값을 Leaf 노드로 하여 머클 트리를 구성할 수 있다. 이 때 Merkle Proof는 특정 데이터가 해당 머클 트리에 속해있는지를 검증하는 방법이다. 스마트 컨트렉트 스토리지에 머클 트리의 Root Hash 값을 미리 저장하고, 추후 사용자가 특정 데이터가 해당 머클 트리에 속해있는지 증명하기 위해 Proof를 전달할 수 있다. 여기서 Proof는 모든 데이터를 포함 할 필요 없이, Root Hash를 재구성하기 위한 최소한의 값들만을 포함하면 된다. (앞선 예시에서 Proof는 모든 데이터 A,...,P 가 아닌 H(K), H(L), H(IJ), H(MNOP), H(ABCDEFGH)만 필요하였다).   

### OpenZeppelin의 Merkle Proof 라이브러리

오픈제플린(OpenZeppelin)에서는 Merkle Proof를 안전하고, 쉽게 구현할 수 있도록 라이브러리를 제공하고 있다. Merkle Tree와 Proof를 생성하기 위한 Javascript 라이브러리와, 이를 온체인상에서 검증하기 위한 Merkle Proof 솔리디티 라이브러리가 별개로 존재한다. 


```
[FAIL. Counterexample: calldata=0x44735ef10000000000000000000000000000000000000000000000000000000000000001, args=[Uint(1)]] testDoubleWithFuzzingCounterExample (gas: [fuzztest])
```


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

### Gas Optimization (Assembly)

위의 OpenZeppelin이 작성한 스마트 컨트렉트 코드는 어셈블리를 사용하지 않고 동작합니다. 대부분의 프로젝트들은 해당 라이브러리를 사용하는 것을 추천드립니다. 다만 Gas 비용을 줄이기 위해서 어셈블리(Assembly)를 사용하여 같은 동작을 수행할 수 있습니다. Art Gobblers라는 NFT 프로젝트의 컨트렉트를 예시로 살펴보겠습니다.  

### Reference
1. [Ethereum Foundation's Blog Post for Merkle Proof]
2. [OpenZeppelin's Github for Merkle Tree]
3. [Belavadi Prahalad's Medium Post for Merkle Proof]

[Merkle Tree]: https://en.wikipedia.org/wiki/Merkle_tree
[Ethereum Foundation's Blog Post for Merkle Proof]: https://ethereum.org/en/developers/tutorials/merkle-proofs-for-offline-data-integrity/
[OpenZeppelin's Github for Merkle Tree]: https://github.com/OpenZeppelin/merkle-tree
[OpenZeppelin MerkleProof]: https://docs.openzeppelin.com/contracts/4.x/api/utils#MerkleProof
[Art Gobblers]: https://github.com/artgobblers/art-gobblers/blob/master/src/ArtGobblers.sol#L347
[Solmate]: https://github.com/transmissions11/solmate
[Belavadi Prahalad's Medium Post for Merkle Proof]: https://medium.com/crypto-0-nite/merkle-proofs-explained-6dd429623dc5