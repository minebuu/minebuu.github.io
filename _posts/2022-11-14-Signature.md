---
layout: post
title:  "지갑 데이터 서명(Signature)과 EIP712"
date:   2022-11-14 19:03:31 +0900
categories: Smart Contract
---

### Introduction
메타마스크를 사용하다보면 Web3 홈페이지에서 메타마스크를 통해 로그인을 수행하거나, 오픈씨와 같은 NFT 거래소에서 NFT를 사고 팔 때 아래 사진과 같은 서명 요청을 받은 경험이 많을 것이다. 여러 사이트에서 서명을 하다 보면 괜스래 어떤 보안 위협이 생기지는 않을까하는 막연한 두려움이 몰려오게 된다. 또한, 어떤 곳은 서명 요청 시 아래 두번째 사진처럼 메시지 내용 안에 특정 해시 값만 보여주는 반면, 오픈씨와 같이 Offerer, Token 값 등 다양한 데이터를 세부적으로 보여주는 곳이 있다. 오늘은 이러한 서명 방식의 차이들을 알아보고 오픈씨에서 사용 중인 서명 방식 (EIP721)에 대해 좀 더 자세히 알아보겠다. 이 글을 통해 그 동안 막연히 가졌던 서명에 대한 불안함을 해소하고, 어떤 방식으로 서명 방식이 좀 더 안전하도록 발전되어 왔는지를 확인했으면 한다.

<p style="text-align: center;">
	<img src="{{ site.url }}/assets/images/Signature/opensea_listing.png" alt="Drawing" style="max-width: 80%; height: auto;"/>
</p>

<p style="text-align: center;">
	<img src="{{ site.url }}/assets/images/Signature/authenticating.png" alt="Drawing" style="max-width: 80%; height: auto;"/>
</p>

### MetaMask에서의 서명
메타마스크와 같은 지갑은 사용자의 Key를 가지고 있고, 이를 통한 다양한 서명 방식을 지원한다. 메타마스크 로그인을 지원하는 웹사이트에서는 서명을 통해 주소에 대한 소유권 [인증(Authentication)](https://medium.com/hackernoon/writing-for-blockchain-wallet-signature-request-messages-6ede721160d5)을 수행할 수 있고, 오픈씨처럼 on-chain 프로토콜을 위해 off-chain 메시지 서명에 활용할 수 있다. 아래는 메타마스크의 지원 서명 함수들 목록입니다.

- `eth_sign`
- `personal_sign`
- `signTypedData` (currently identical to `signTypedData_v1`)
- `signTypedData_v1`
- `signTypedData_v3`
- `signTypedData_v4`

위에서 `eth_sign`은 MetaMask가 가장 처음으로 제공한 서명 함수로서, 임의의 해시 값에 단순히 서명하는 방식이다. 임의의 해시 값은 특정 트랜잭션이나 다른 데이터가 될 수 있기 때문에, 피싱 공격에 사용될 수 있어 매우 위험한 서명 함수로 더 이상 사용되지 않는다. 우리는 비록 가스를 소모하지 않더라도, 메타마스크에서 eth_sign를 통해 서명하는 것만으로 모든 자산이 탈취될 위험이 있다. 이와 관련된 예시는 다음 [트위터 쓰레드](https://twitter.com/CT_IOE/status/1534658825843683328?s=20&t=oTD2R7LJ3w5jZAUeLt2xog)에 나와있다.  

이러한 위험 때문에 `eth_sign`을 통해 사용자에게 서명 요청이 발생할 시 메타마스크는 아래와 같은 빨간색 경고 구문을 보여준다. 반대로 이러한 경고 구문이 없다면, `eth_sign`을 통한 서명 요청이 아니므로 사용자는 이러한 피싱 공격에 안심하고 서명을 수행해도 된다. 예시로, 본 포스팅의 위에 있는 두번째 사진은 아래 사진처럼 해시 값만 보여주지만 Warning이 표시되지 않는다. 이는 `eth_sign` 서명 요청이 아닌 것을 의미하므로 안심하고 서명을 수행해도 된다.  

<p style="text-align: center;">
	<img src="{{ site.url }}/assets/images/Signature/eth_sign_warning.png" alt="Drawing" style="max-width: 80%; height: auto;"/>
</p>

`eth_sign`의 이러한 피싱 공격 문제 때문에, `personal_sign`에서는 공격자가 메시지의 임의의 트랜잭션을 가장하여 넣을 수 없도록 개선되었다. `personal_sign`은 `eth_sign`이 해시 값만 보여주는 것과는 다르게 사람이 읽을 수 있는 형태(UTF-8)로 원하는 메시지를 설정할 수 있다. `personal_sign`의 예시는 아래와 같다.

```
handleSignMessage = ({ publicAddress, nonce }) => {
    return new Promise((resolve, reject) =>
      web3.personal.sign(
        web3.fromUtf8(`I am signing my one-time nonce: ${nonce}`),
        publicAddress,
        (err, signature) => {
          if (err) return reject(err);
          return resolve({ publicAddress, signature });
        }
      )
    );
  };
```   

`personal_sign`을 통해 raw data 서명을 통한 피싱 공격 위험은 사라졌지만, 여전히 사용자들은 자신이 무엇에 대해 서명하는지 명확히 알지 못한다. 이는 [0xProtocol]의 Github 이슈에서도 지적하고 있는데, `eth_sign`과 `personal_sign`도 아닌 유저가 서명하는 데이터를 `명시적`으로 보여줄 수 있는 새로운 서명 함수를 필요로 하고 있다. 이러한 요구에 맞춰, 0xProtocol 팀은 ethereum/EIPs#683에 다음과 같은 [코멘트 1](https://github.com/ethereum/EIPs/pull/683#issuecomment-327945854)와 [코멘트2](https://github.com/ethereum/EIPs/pull/683#issuecomment-328074258)를 남겼다. 이러한 요구가 이더리움 커뮤니티 전체의 호응을 얻으며 공식적으로 [EIP712](https://github.com/ethereum/EIPs/pull/712) 제안이 생성되었다. 이렇게 만들어진 EIP712 spec에서 가장 많이 사용되는 버전이 `signTypedData_v3`이며, 해당 spec의 가장 최신 버전이 `signTypedData_v4`이다. `signTypedData_v1`은 EIP712 spec의 가장 초기 버전으로, 추후에 나온 보안 개선 사항들이 반영되지 않았으므로 `signTypedData_v3`와 `signTypedData_v4`을 사용하는 것을 추천한다.        

### EIP712
그럼 EIP712는 `eth_sign`과`personal_sign`에 비해 어떠한 점들이 개선되었는지를 알아보자. 0xProtocol 팀이 요구한 사항처럼, 사용자들은 자신이 서명하고 있는 데이터를 명시적으로 알 필요가 있다.



자세한 내용들은 해당 [블로그](https://medium.com/metamask/eip712-is-coming-what-to-expect-and-how-to-use-it-bb92fd1a7a26) 포스팅에서 확인할 수 있다.  


### 서명을 이용한 공격 사례

### 결론

<!--
```
offerer:
0x4fbEde53b59D1a2Ab85F785ad76E1dcD66A1546A
offer:
0:
itemType:
2
token:
0x942BC2d3e7a589FE5bd4A5C6eF9727DFd82F5C8a
identifierOrCriteria:
906
startAmount:
1
endAmount:
1
consideration:
0:
itemType:
0
token:
0x0000000000000000000000000000000000000000
identifierOrCriteria:
0
startAmount:
108000000000000000
endAmount:
108000000000000000
recipient:
0x4fbEde53b59D1a2Ab85F785ad76E1dcD66A1546A
1:
itemType:
0
token:
0x0000000000000000000000000000000000000000
identifierOrCriteria:
0
startAmount:
3000000000000000
endAmount:
3000000000000000
recipient:
0x0000a26b00c1F0DF003000390027140000fAa719
2:
itemType:
0
token:
0x0000000000000000000000000000000000000000
identifierOrCriteria:
0
startAmount:
9000000000000000
endAmount:
9000000000000000
recipient:
0x6C093Fe8bc59e1e0cAe2Ec10F0B717D3D182056B
startTime:
1668422891
endTime:
1671014891
orderType:
2
zone:
0x004C00500000aD104D7DBd00e3ae0A5C00560C00
zoneHash:
0x0000000000000000000000000000000000000000000000000000000000000000
salt:
24446860302761739304752683030156737591518664810215442929811789059466231445624
conduitKey:
0x0000007b02230091a7ed01230072f7006a004d60a8d4e71d599b8104250f0000
counter:
0
``` -->


### Reference
1. [MetaMask's Guide for signing the data]
2. [체인의 정석: EIP712]
3. [Koh Wei Jie's Blog post for EIP712]

[MetaMask's Guide for signing the data]: https://docs.metamask.io/guide/signing-data.html#a-brief-history
[체인의 정석: EIP712]: https://it-timehacker.tistory.com/316
[]:https://medium.com/hackernoon/writing-for-blockchain-wallet-signature-request-messages-6ede721160d5
[Koh Wei Jie's Blog post for EIP712]:https://medium.com/metamask/eip712-is-coming-what-to-expect-and-how-to-use-it-bb92fd1a7a26
[0xProtocol]: https://github.com/0xProject/0x-monorepo/issues/162
