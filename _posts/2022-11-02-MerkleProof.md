---
layout: post
title:  "스마트 컨트렉트에서의 Merkle Proof 사용"
date:   2022-11-02 19:03:31 +0900
categories: Smart Contract
---

### Merkle Proof
특정한 주소 리스트들만 함수를 실행할 수 있도록 스마트 컨트렉트를 작성하려면 어떻게 해야할까? 이더리움의 저장 공간과 비용에 제약이 없다면 모든 주소 리스트들을 Storage에 저장하는 것이 하나의 방법일 것이다. 하지만 이더리움 네트워크는 수 많은 컴퓨터가 상태를 공유하는 월드 컴퓨터로서, 이러한 월드 컴퓨터에 데이터를 저장하는 것은 많은 비용이 소모된다 (32 bytes의 word를 저장하는 것은 통상적으로 20,000 gas를 소모, 글쓴 시점 기준 약 8천원).

우리는 Merkle Proof를 통해서 모든 주소 데이터를 Storage에 저장하지 않고도 해당 주소 데이터가 함수 실행 허용 주소 목록에 들어가는지 아닌지를 판단할 수 있다. 즉 모든 데이터 (여기선 주소 리스트)를 이더리움 온체인에 저장하지 않고도 특정 데이터의 무결성(Data integrity)을 검증할 수 있다.  

### 동작 과정 
테스트를 하다보면 생각하지도 못한 에러사항이 발생하게됩니다. 사람이 모든 테스트 영역을 커버하기는 쉽지 않으므로 무작위의 input을 통해 자동화된 테스트를 수행하는 퍼징 기능(무작위 데이터 입력)이 필요할 수 있습니다. 아래는 Foundry에서 지원하는 퍼징 기능의 한 예시입니다.

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

### 예제

그럼 이제 간단히 `forge`를 어떻게 사용하는지 살펴보도록 하겠습니다.  

### Reference
1. [Ethereum Foundation's Blog Post for Merkle Proof]
2. [OpenZeppelin]

[MERKLE PROOFS FOR OFFLINE DATA INTEGRITY]: https://ethereum.org/en/developers/tutorials/merkle-proofs-for-offline-data-integrity/
[OpenZeppelin]: https://www.paradigm.xyz/2021/12/introducing-the-foundry-ethereum-development-toolbox
[Art Gobblers]: https://github.com/artgobblers/art-gobblers/blob/master/src/ArtGobblers.sol#L347
[Solmate]: https://github.com/transmissions11/solmate
