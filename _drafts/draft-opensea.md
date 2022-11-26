---
layout: post
title:  "Opensea: Wyvern 프로토콜과 Seaport 코드 분석"
categories: Smart Contract
---
<!-- jekyll serve --drafts -->

### Introduction

test
### Wyvern v2.2 취약점

2022년 1분기에, Gus, 오픈씨 보안 개발자, Wyvern 프로토콜 개발자 그리고 samczsun에 의해서 offer를 수행한 사용자들에게서 ETH를 탈취할 수 있는 취약점이 발견되었다. 해당 취약점은 리스팅과 offer를 수행한 유저들이 정상적인 서명과 행동을 수행했더라도, 별다른 추가 활동 없이 악용이 가능하였다. 그동안의 로그 상으론 해당 취약점으로 인한 피해는 없었다고 하였으나, 해당 시점에 오픈씨의 여러 해킹 사건이 발생했었는데 연관이 있는지는 추후 체크해보도록 하겠다.   

[[CTO of Opensa: A critical vulnerability in Wyvern Protocol]에서 이 취약점에 대해 자세히 설명하고있다. 해당 취약점은


이는 `abi.encodePacked`을 서명, 인증 및 데이터 무결성 체크에 확인할 때 주의해야하는 이유와 비슷하다.
`keccak256(abi.encodePacked(a, b))`를 계산할 때 a와 b가 string, bytes와 같은 동적 타입이라면 a와 b의 값 일부를 서로 간에 이동시켜도 동일한 결과 값을 얻을 수 있다. 예시로 변수 string a = "test", string b = "collision"이라고 해보자. `keccak256(abi.encodePacked(a, b)) = keccak256(abi.encodePacked("test", "collision")) = keccak256(abi.encodePacked("te", "stcollision"))`와 동일하다. 따라서 특별한 이유가 없다면 abi.encode 사용을 권고한다. 이러한 경고는 [Solidity Docs](https://docs.soliditylang.org/en/latest/abi-spec.html#non-standard-packed-mode)에 non-standard mode의 주의사항으로 나와있다.    

### Yul을 통한 Seaport의 가스 최적화

### Reference
1. [CTO of Opensa: A critical vulnerability in Wyvern Protocol]
2. [Haechi Audit: Wyvern v2.2 1-day Vulnerabilities]
3. [EIP712]

[Haechi Audit: Wyvern v2.2 1-day Vulnerabilities]: https://blog.audit.haechi.io/wyvern_v2_2_1_1day_vulnerabilities
[EIP712]:https://github.com/ethereum/EIPs/blob/master/EIPS/eip-712.md
[CTO of Opensa: A critical vulnerability in Wyvern Protocol]: https://nft.mirror.xyz/VdF3BYwuzXgLrJglw5xF6CHcQfAVbqeJVtueCr4BUzs
