---
layout: post
title:  "지갑 데이터 서명(Signature)과 EIP712"
date:   2022-11-14 19:03:31 +0900
categories: Smart Contract
---

### Introduction
메타마스크를 사용하다보면 Web3 홈페이지에서 메타마스크를 통해 로그인을 수행하거나, 오픈씨와 같은 NFT 거래소에서 NFT를 사고 팔 때 아래 사진과 같은 서명 요청을 받은 경험이 많을 것이다. 여러 사이트에서 서명을 하다 보면 괜스래 어떤 보안 위협이 생기지는 않을까하는 막연한 두려움이 몰려오게 된다. 또한, 어떤 곳은 서명 요청 시 아래 Figure 2 사진처럼 메시지 내용 안에 특정 해시 값만 보여주는 반면, Figure 1 사진처럼 오픈씨와 같이 Offerer, Token 값 등 다양한 데이터를 세부적으로 보여주는 곳이 있다. 오늘은 이러한 서명 방식의 차이들을 알아보고 오픈씨에서 사용 중인 서명 방식 (EIP721)에 대해 좀 더 자세히 알아보겠다. 이 글을 통해 그 동안 막연히 가졌던 서명에 대한 불안함을 해소하고, 어떤 방식으로 서명 방식이 좀 더 안전하도록 발전되어 왔는지를 확인했으면 한다.

<p style="text-align: center;">
	<img src="{{ site.url }}/assets/images/Signature/opensea_listing.png" alt="Drawing" style="max-width: 80%; height: auto;"/>
	<figcaption align = "center"><b>Figure 1. Opensea 리스팅 시 서명 요청</b></figcaption>
</p>


<p style="text-align: center;">
	<img src="{{ site.url }}/assets/images/Signature/authenticating.png" alt="Drawing" style="max-width: 80%; height: auto;"/>
	<figcaption align = "center"><b>Figure 2. Web3 지갑 로그인 시 서명 요청</b></figcaption>
</p>

### MetaMask에서의 서명
메타마스크와 같은 지갑은 사용자의 Key를 가지고 있고, 이를 통한 다양한 서명 방식을 지원한다. 메타마스크 로그인을 지원하는 웹사이트에서는 서명을 통해 주소에 대한 소유권 [인증(Authentication)](https://medium.com/hackernoon/writing-for-blockchain-wallet-signature-request-messages-6ede721160d5)을 수행할 수 있고, 오픈씨처럼 on-chain 프로토콜을 위해 off-chain 메시지 서명에 활용할 수 있다. 아래는 메타마스크의 지원 서명 함수들 목록이다.

- `eth_sign`
- `personal_sign`
- `signTypedData` (현재 `signTypedData_v1`와 동일)
- `signTypedData_v1`
- `signTypedData_v3`
- `signTypedData_v4`

위에서 `eth_sign`은 MetaMask가 가장 처음으로 제공한 서명 함수로서, 임의의 해시 값에 단순히 서명하는 방식이다. 임의의 해시 값은 특정 트랜잭션이나 다른 데이터가 될 수 있기 때문에, 피싱 공격에 사용될 수 있어 매우 위험한 서명 함수로 더 이상 사용되지 않는다. 우리는 비록 가스를 소모하지 않더라도, 메타마스크에서 `eth_sign`를 통해 서명하는 것만으로 모든 자산이 탈취될 위험이 있다. 이와 관련된 예시는 다음 [트위터 쓰레드](https://twitter.com/CT_IOE/status/1534658825843683328?s=20&t=oTD2R7LJ3w5jZAUeLt2xog)에 나와있다. 간단히 말해, 공격자는 "공격자의 주소로 10 ETH를 전송합니다"와 같은 메시지를 만들어 `eth_sign`을 요청할 수 있으며, 이러한 메시지는 사용자가 보기엔 단순한 bytes 배열이기 때문에 위험성을 모르고 서명을 할 위험이 있다. 서명을 수락한다면 이는 이더리움 네트워크에 제출할 수 있는 트랜잭션으로 완성되고, 공격자는 서명된 메시지를 이더리움 블록체인상에 제출하여 10 ETH를 탈취할 수 있다.  

<code><b>이러한 위험성 때문에 `eth_sign`을 통해 사용자에게 서명 요청이 발생할 시 메타마스크는 아래 Figure 3과 같이 빨간색 경고 구문을 보여준다. 반대로 이러한 경고 구문이 없다면, `eth_sign`을 통한 서명 요청이 아니므로 사용자는 이러한 피싱 공격에 안심하고 서명을 수행해도 된다.</b></code> 예시로, Figure 2에서는 Figure 3과 같이 메시지에 해시 값만 보여주지만 Warning이 표시되지 않는다. 이는 `eth_sign` 서명 요청이 아니라 위의 피싱 문제를 해결한 다른 서명 함수를 사용하는 것을 의미하므로 피싱 공격에 안심하고 서명을 수행해도 된다.  

<p style="text-align: center;">
	<img src="{{ site.url }}/assets/images/Signature/eth_sign_warning.png" alt="Drawing" style="max-width: 80%; height: auto;"/>
	<figcaption align = "center"><b>Figure 3. `eth_sign` 서명 요청 시 메타마스크에서의 경고 문구</b></figcaption>
</p>

위의 `eth_sign`를 사용하는 Dapp에선 사용자가 전적으로 Dapp을 신뢰해야한다. 전문가가 아니고선 특정 바이트가 나의 자산을 탈취하는 메시지인지 올바르게 의도된 메시지인지 판단하기 어렵기 때문이다. 이런 문제를 해결하고자 `personal_sign`에서는 공격자가 메시지에 임의의 트랜잭션을 가장하여 넣을 수 없도록 개선되었다. 해당 서명 함수에선 `"\x19Ethereum Signed Message:\n" + len(message)`의 메시지가 해싱 전 메시지 앞 부분에 추가된다. 즉 `personal_sign`은 다음과 같이 계산된다: `sign(keccak256("\x19Ethereum Signed Message:\n" + len(message) + message))`. 그렇기 때문에 공격자가 "공격자의 주소로 10 ETH를 전송합니다"와 같은 메시지를 넣더라도, 앞에 `"\x19Ethereum Signed Message:\n" + len(message)`가 추가되어 해싱되기 때문에 실제 이더리움 네트워크에 트랜잭션으로서 유효하지 않다. 또한, `personal_sign`은 `eth_sign`이 바이트 값 (해시 값)만 보여주는 것과는 다르게 사람이 읽을 수 있는 형태(UTF-8)로 메시지를 메타마스크에 출력할 수 있다. `personal_sign`의 예시는 아래와 같다.

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

`personal_sign`을 통해 raw data 서명을 통한 피싱 공격 위험은 사라졌지만, 여전히 사용자들은 자신이 무엇에 대해 서명하는지 명확히 알지 못한다. Figure 1처럼 우리는 특정 거래에 대한 서명을 수행할 때 거래에 대한 정보를 명시적으로 확인할 수 있어야한다. 이는 [0xProtocol]의 Github 이슈에서도 지적하고 있는데, `eth_sign`과 `personal_sign`도 아닌 유저가 서명하는 데이터를 `명시적`으로 보여줄 수 있는 새로운 서명 함수를 필요로 하고 있다. 이러한 요구에 맞춰, 0xProtocol 팀은 ethereum/EIPs#683에 다음과 같은 [코멘트 1](https://github.com/ethereum/EIPs/pull/683#issuecomment-327945854)와 [코멘트2](https://github.com/ethereum/EIPs/pull/683#issuecomment-328074258)를 남겼다. 이러한 요구가 이더리움 커뮤니티 전체의 호응을 얻으며 공식적으로 [EIP712](https://github.com/ethereum/EIPs/pull/712) 제안이 생성되었다. 이렇게 만들어진 EIP712 spec에서 가장 많이 사용되는 버전이 `signTypedData_v3`이며, 해당 spec의 가장 최신 버전이 `signTypedData_v4`이다. `signTypedData_v1`은 EIP712 spec의 가장 초기 버전으로, 추후에 나온 보안 개선 사항들이 반영되지 않았으므로 `signTypedData_v3`와 `signTypedData_v4`을 사용하는 것을 추천한다.        

### EIP712
그럼 EIP712는 `eth_sign`과`personal_sign`에 비해 어떠한 점들이 개선되었는지를 알아보자. 0xProtocol 팀이 요구한 사항처럼, 사용자들은 자신이 서명하고 있는 데이터를 명시적으로 알 필요가 있으며, 피싱 공격을 방지할 수 있어야 한다.

[EIP712](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-712.md) 공식 문서를 보면, 해당 제안의 타이틀은 "구조화된 데이터의 해싱과 서명 (Typed structured data hashing and signing)"이다. 즉, EIP712의 구현에서는 단순한 바이트 스트링이 아닌, 아래 Figure 4와 같이 웹사이트/컨트렉트 이름, 체인 ID, 컨트렉트 주소, Dapp에서 필요로하는 메시지 정보 등이 구조화되어 사용자에게 표시되고, 이러한 구조화된 데이터를 해싱하고 서명을 수행한다.


정리하자면, 우리는 EIP 712를 통해 다음과 같은 이점을 얻을 수 있다.
- `domain`을 통해 서명된 데이터가 의도된 컨트렉트와 다른 곳에서 사용될 수 없다.
- `chainID`를 통해 서명이 다른 체인에서 사용되지 않는다. Rinkeby에서 생성된 서명은 이더리움 메인넷에서 사용될 수 없다.
- 기타 등등 추가 예정 



자세한 내용들은 해당 [블로그](https://medium.com/metamask/eip712-is-coming-what-to-expect-and-how-to-use-it-bb92fd1a7a26) 포스팅에서 확인할 수 있다.  


### 서명을 이용한 공격 사례

### 결론

### Appendix
`personal_sign`은 메타마스크에서만 사용되는 함수로서 `eth_sign`에서 트랜잭션을 가장한 공격을 막기 위해 `"\x19Ethereum Signed Message:\n" + len(message)`가 해싱 전 메시지 앞 부분에 추가되었다고 앞에서 설명하였다. 즉 기존에 문제가 되던 `eth_sign`은 그대로 유지하고
하지만 이는 Ethereum JSON-RPC API의 `eth_sign`을 보면 혼동될 수 있다. 왜냐하면 이더리움 API에서는 `eth_sign`이 메타마스크의 `personal_sign`과 동일하게 동작하도록 변경되었기 때문이다. 정리하자면, `eth_sign`은 이더리움 공식 API에선 `"\x19Ethereum Signed Message:\n" + len(message)`가 항상 메시지 앞에 붙도록 변경되었고, 메타마스크의 구현에선 옛날에 사용하던 `eth_sign`는 그대로 두고, `"\x19Ethereum Signed Message:\n" + len(message)`를 메시지 앞에 붙이는 `personal_sign` 함수를 별도로 구현하였다. 메타마스크의 `eth_sign`과 Ethereum JSON-RPC API의 `eth_sign`에 대한 혼동이 없길 바란다.    


### Reference
1. [MetaMask's Guide for signing the data]
2. [체인의 정석: EIP712]
3. [Koh Wei Jie's Blog post for EIP712]

[MetaMask's Guide for signing the data]: https://docs.metamask.io/guide/signing-data.html#a-brief-history
[체인의 정석:EIP712]: https://it-timehacker.tistory.com/316
[hackernoon]:https://medium.com/hackernoon/writing-for-blockchain-wallet-signature-request-messages-6ede721160d5
[Koh Wei Jie's Blog post for EIP712]:https://medium.com/metamask/eip712-is-coming-what-to-expect-and-how-to-use-it-bb92fd1a7a26
[0xProtocol]: https://github.com/0xProject/0x-monorepo/issues/162#issuecomment-328263512
