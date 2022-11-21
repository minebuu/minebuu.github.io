---
layout: post
title:  "지갑 데이터 서명(Signature)과 EIP712"
date:   2022-11-14 19:03:31 +0900
categories: Smart Contract
---

### Introduction
메타마스크를 사용하다보면 Web3 홈페이지에서 메타마스크를 통해 로그인을 수행하거나, 오픈씨와 같은 NFT 거래소에서 NFT를 사고 팔 때 아래 사진과 같은 서명 요청을 받은 경험이 많을 것이다. 여러 사이트에서 서명을 하다 보면 괜스래 어떤 보안 위협이 생기지는 않을까하는 막연한 두려움이 몰려오게 된다. 또한, 어떤 곳은 서명 요청 시 아래 Figure 2 사진처럼 메시지 내용 안에 특정 해시 값만 보여주는 반면, Figure 1 사진처럼 오픈씨와 같이 Offerer, Token 값 등 다양한 데이터를 세부적으로 보여주는 곳이 있다. 오늘은 이러한 서명 방식의 차이들을 알아보고 오픈씨에서 사용 중인 서명 방식 [EIP712]에 대해 좀 더 자세히 알아보겠다. 이 글을 통해 그 동안 막연히 가졌던 서명에 대한 불안함을 해소하고, 어떤 방식으로 서명 방식이 좀 더 안전하도록 발전되어 왔는지를 확인했으면 한다.

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

[EIP712] 공식 문서를 보면, 해당 제안의 타이틀은 "구조화된 데이터의 해싱과 서명 (Typed structured data hashing and signing)"이다. 즉, EIP712의 구현에서는 단순한 바이트 스트링이 아닌, 아래 Figure 4와 같이 웹사이트/컨트렉트 이름, 체인 ID, 컨트렉트 주소, Dapp에서 필요로하는 메시지 정보 등이 구조화되어 사용자에게 표시되고, 이러한 구조화된 데이터를 해싱하고 서명을 수행한다.

<p style="text-align: center;">
	<img src="{{ site.url }}/assets/images/Signature/eip712.png" alt="Drawing" style="max-width: 80%; height: auto;"/>
	<figcaption align = "center"><b>Figure 4. `EIP712`를 사용한 메시지 서명 요청</b></figcaption>
</p>

EIP712을 따르는 서명의 데이터 구조는 `Domain`이 필수적으로 포함되어야하고, 선택적으로 설계할 수 있는 `Message` 부분으로 구성되어있다. Dapp들마다 필요로 하는 데이터 구조가 다를 수 있으므로, 각 Dapp은 자신들의 요구사항에 맞춰 Domain과 Message 부분을 설계할 수 있다.

`Domain`은 EIP712에서 필수적으로 포함되야하는 부분으로, 아래 필드들이 하나 이상 포함되어야한다. 프로토콜 디자이너는 서명 도메인에 맞춰 아래 필드 중 필요 없는 것들을 제거할 수 있다 (다만 필드의 순서는 지켜야 한다). 보통은 [Uniswap v2](https://github.com/Uniswap/v2-core/blob/master/contracts/UniswapV2ERC20.sol#L31)에서와 같이 `salt` 필드를 제외한 나머지 필드들은 모두 포함하여 Domain을 구성한다.  

- `string name`: 사용자가 알아볼 수 있는 Dapp 혹은 프로토콜 이름
- `string version`: 현재 도메인 객체의 버전. 서로 다른 버전을 갖는 서명들은 호환되지 않음
- `uint256 chainId`: EIP-155에서 제안된 chain id. 예를 들어 Rinkeby의 chain id를 갖는 서명은 이더리움 메인넷에서 동작하지 않음
- `address verifyingContract`: 서명이 사용될 컨트렉트의 주소. 서명이 사용될 컨트렉트를 명확히 명시함으로써 피싱 공격을 방지할 수 있음.
- `bytes32 salt`: 컨트랙트와 dApp 모두에 하드코딩된 고유한 32바이트 값으로, dApp을 다른 dApp과 구별하기 위한 최후의 수단

<br/>
`Message`는 Domain과 다르게 모든 부분이 Dapp 선택적으로 프로토콜에 적절하게 구현될 수 있다. 아래와 같은 우리는 특정 주소가 특정 주소에게 메일을 보내는 프로토콜에서의 데이터 구조를 생각해보자. 여기서 우리는 Solidity와의 호환성을 위해 Solidity의 표기법을 따른다.

```
struct Mail {
    address from;
    address to;
    string contents;
}
```

이제 `Domain`과 `Message`에 대해 알아보았으니, 위 정보들을 가지고 서명을 어떻게 구현하는지를 알아보겠다. 우선 다시 한번 [EIP712]를 보자면, `eth_signTypedData` 함수는 다음과 같이 계산된다: `sign(keccak256("\x19\x01" ‖ domainSeparator ‖ hashStruct(message)))`.
이는 EIP-191과 호환되는 인코딩을 따라는데, 여기서 `\x19`는 실제 이더리움 트랜잭션과 서명을 구별하는 역할을 함으로써, `eth_sign`에서 발생 가능했던 피싱 공격을 방지한다. `0x01`은 `version byte` 값으로 EIP712를 위한 고정된 값이다. domainSeparator와 hashStruct에 대한 설명은 아래에 자세히 나와 있으며 두 값 모두 32 바이트의 길이를 가진다.

> 글쓰는 시점 EIP712의 #L168에서는 eth_signTypedData 함수가 `sign(keccak256("\x19Ethereum Signed Message:\n" + len(message) + message)))`로 정의되어 있다고 나오지만, 이는 잘못 표기된 것이다 (기존 eth_sign 내용의 잘못된 copy). 이러한 오류가 [깃헙 이슈](https://github.com/ethereum/EIPs/pull/5457)에서 다뤄지고 있고, 곧 위에서 쓴 정의인 `sign(keccak256("\x19\x01" ‖ domainSeparator ‖ hashStruct(message)))` 로 수정될 것으로 보인다.

#### HashStruct
`domainSeparator`와 `hashStruct`의 정의를 이어서 살펴보자. 우선 hashStruct의 정의는 다음과 같다.  

<code>hashStruct(s : structured data 𝕊) = keccak256(typeHash ‖ encodeData(s)) where typeHash = keccak256(encodeType(typeOf(s))) </code>

여기서 `encodeData`는 `enc(value₁) ‖ enc(value₂) ‖ … ‖ enc(valueₙ)`를 의미하는데, 각 value는 정확히 32 바이트 길이여야한다. 즉 bytes 혹은 string과 같은 dynamic 값들은 값마다 길이가 다르므로 keccak256 함수를 적용해 32 바이트 길이로 통일시켜줘야한다. 배열의 경우에는 `keccak256( arr[0] || arr[1] || arr[2])`와 같이 배열 값들을 concate하여 계산한다. 마지막으로 구조체(struct)는 재귀적으로 인코딩된다.

`encodeType`은 구조체 데이터 𝕊를 다음과 같이 인코딩한다: `name ‖ "(" ‖ member₁ ‖ "," ‖ member₂ ‖ "," ‖ … ‖ memberₙ ")"`. 여기서 member `type ‖ " " ‖ name.`이다. 위의 Mail 구조체 데이터 𝕊를 예시로 encodeType을 적용할 경우 `Mail(address from,address to,string contents)`와 같다. `address from`처럼 변수의 타입과 변수명에는 스페이스 공백이 들어가야되며, 변수명 다음에는 바로 `,`가 붙는다. 또한 이러한 멤버들을 구조체 이름인 Member가 `()`으로 감싼 형태이다. 이렇게 인코딩 된 데이터에 keccak256 해시함수에 적용한 결과가 `typeHash` 값이 된다. 데이터를 인코딩할 때 스페이스 하나만 달라져도 해시함수의 결과값이 달리지기 때문에, 위에서 정의한 `encodeType`을 지키는게 매우 중요하다.

>구조체 데이터가 또 다른 구조체 데이터를 포함하는 경우에는 다음과 같이 인코딩한다. 가장 먼저 참조를 하는 데이터 구조를 위치시킨다. 그리고 참조하는 구조체들을 알파벳순으로 정렬하여 뒤에 추가한다. 예시로, `Transaction(Person from,Person to,Asset tx)`과 같은 구조체가 있을 때 Asset 구조체가 Person 구조체보다 알파벳 순서가 빠르므로 다음과 같이 인코딩 될 수 있다: `Transaction(Person from,Person to,Asset tx)Asset(address token,uint256 amount)Person(address wallet,string name)`



위의 수식들이 잘 이해가 안될수도 있으니, 위에서 언급한 Mail 구조체를 예시로 `hashStruct(Mail mail)`을 솔리디티 코드로 구현해보며 차근차근 이해해보자.

```
pragma solidity ^0.8.0;

contract Example {

	//structured Data 𝕊
	struct Mail {
	    address from;
	    address to;
	    string contents;
	}

	//1. Calculate TypeHash = keccak256(encodeType(typeOf(s)))
	bytes32 public constant MAIL_TYPEHASH = keccak256(
			"Mail(address from,address to,string contents)"
	);

	//2. hashStruct(s) = keccak256(typeHash ‖ encodeData(s))
	//3. encodeData = enc(value₁) ‖ enc(value₂) ‖ … ‖ enc(valueₙ)
	function hashStruct(Mail mail) internal pure returns (bytes32) {
			return keccak256(abi.encode(
					MAIL_TYPEHASH,
					mail.from,
					mail.to,
					//bytes, string 값에는 keccak256 함수를 적용해 32 바이트 길이로 통일
					keccak256(bytes(mail.contents))
			));
	}
}
```
위 코드를 보면, Mail 구조체의 hashStruct를 어떻게 구하는지 쉽게 이해할 수 있을 것이다. 우선 TypeHash는 앞서 설명한대로 1. 정해진 규칙에 맞춰 인코딩 된 값에 keccak256 해시함수를 적용하여 계산한다. 2. hashStruct는 typeHash와 encodeData(Mail)을 concate하여 keccak256 해시 함수를 적용함으로써 계산한다. 여기서 3. encodeData는 위에서 언급했듯이 구조체의 모든 값들에 대해 `enc(value₁) ‖ enc(value₂) ‖ … ‖ enc(valueₙ)`를 수행한다. 다만 string과 같은 dynamic 값들은 keccak256를 적용함으로써 32바이트 길이로 통일 시켜줘야한다.     

> hashStruct 함수에서 왜 해시함수 전에 abi.encode를 쓸까? 이에 대한 [Openzeppelin 포럼](https://forum.openzeppelin.com/t/abi-encode-vs-abi-encodepacked/2948) QnA

#### domainSeparator
위에서 hashStruct(Mail mail) 값을 구해봤지만, 우리는 Domain Sapartor라는 값도 구해야한다. Domain Saparator역시 위 과정과 매우 유사하게 구할 수 있다. 정의는 다음과 같다: `domainSeparator = hashStruct(eip712Domain)`

즉 hashStruct 함수의 인자에 임의의 메시지가 아닌 EIP712에서 정의하는 Domain을 넣는 점이 다르다. 예시로 salt를 제외한 모든 필드(name, version, chainId, verifyingContract)를 사용하는 상황을 가정해보겠다.

```solidity
bytes32 public DOMAIN_SEPARATOR;
bytes32 public constant EIP712DOMAIN_TYPEHASH = keccak256("EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)");

constructor (string memory name, string memory version) {
		//hashStruct(s) = keccak256(typeHash ‖ encodeData(s))
		DOMAIN_SEPARATOR = keccak256(
				abi.encode(
						//typeHash(eip712Domain)
						EIP712DOMAIN_TYPEHASH,
						//name
						keccak256(bytes(name)),
						//version
						keccak256(bytes(version)),
						//chaindid
						block.chainid,
						//verifyingContract
						address(this)
				)
		);
}
```  
hashStruct(s) = keccak256(typeHash ‖ encodeData(s)) 임을 상기시켜보자. 여기서도 마찬가지로, domain에 대한 typeHash 값을 구한다. 이 typeHash 값과 Domain을 인코딩한 값을 concate하여 keccak256 해싱을 수행한다. 이로써 Domain Separator 값을 구할 수 있다. 여기서 우리는 별도의 함수가 아닌, constructor() 생성자에서 Domain Separator을 구한다. 이는 컨트렉트의 이름과 주소는 대부분 고정된 값으기 때문에 일반적으로 생성자 시점에 계산한다. 또한 가스비 절감을 위해 immutable 변수에 저장하였다.

>하지만, proxy를 통한 upgradeable 컨트렉트의 경우에는 로직 컨트렉트의 주소가 변경될 수 있기 때문에 위의 방식을 사용할 수 없다. upgradeable 컨트렉트의 경우엔 [Openzeppelin의 EIP712Upgradeable](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/master/contracts/utils/cryptography/EIP712Upgradeable.sol)를 참고해라

이제 Domain Separator값과 hashStruct(Message) 값을 이용해 `eth_signTypedData`의 최종 결과 값인  `sign(keccak256("\x19\x01" ‖ domainSeparator ‖ hashStruct(message)))`을 구할 수 있다. 위의 컨트렉트들을 아래와 같이 합쳐 Remix에서 테스트해보자. 풀 코드는 아래와 같다.

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity ^0.8.10;

contract Example {

    //structured Data 𝕊
	struct Mail {
	    address from;
	    address to;
	    string contents;
	}

    bytes32 public immutable DOMAIN_SEPARATOR;
    bytes32 public constant EIP712DOMAIN_TYPEHASH = keccak256("EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)");
	bytes32 public constant MAIL_TYPEHASH = keccak256("Mail(address from,address to,string contents)");

    constructor (string memory name, string memory version) {
		DOMAIN_SEPARATOR = keccak256(
			abi.encode(
				//typehash(eip712Domain)
				EIP712DOMAIN_TYPEHASH,
				//name
				keccak256(bytes(name)),
				//version
				keccak256(bytes(version)),
				//chaindid
				block.chainid,
				//verifyingContract
				address(this)
			)
	    );
    }

	function hashStruct(Mail memory mail) internal pure returns (bytes32) {
		return keccak256(
            abi.encode(
			    MAIL_TYPEHASH,
			    mail.from,
			    mail.to,
			    keccak256(bytes(mail.contents))
		    )
        );
	}

    function signTypedData_v4(Mail memory mail) internal view returns (bytes32) {
        return keccak256(
            abi.encodePacked(
                '\x19\x01',
                DOMAIN_SEPARATOR,
                hashStruct(mail)
            )
        );
    }

    function verify(Mail memory mail, uint8 v, bytes32 r, bytes32 s) internal view returns (bool) {
    // Note: we need to use `encodePacked` here instead of `encode`.
        bytes32 digest = keccak256(abi.encodePacked(
            "\x19\x01",
            DOMAIN_SEPARATOR,
            hashStruct(mail)
        ));
        return ecrecover(digest, v, r, s) == mail.from;
    }


    function test() public view returns (bool) {
        // Example signed message
        Mail memory mail = Mail({
            from: 0xCD2a3d9F938E13CD947Ec05AbC7FE734Df8DD826,
            to: 0xbBbBBBBbbBBBbbbBbbBbbbbBBbBbbbbBbBbbBBbB,
            contents: "Hello, Bob!"
        });
        uint8 v = 28;
        bytes32 r = 0x4355c47d63924e8a72e509b65029052eb6c299d53a04e167c5775fd466751c9d;
        bytes32 s = 0x07299936d304c153f6443dfa05f40ff007d72911b6f72307f996231605b91562;

        // assert(DOMAIN_SEPARATOR == 0xf2cee375fa42b42143804025fc449deafd50cc031ca257e0b194a650a912090f);
        // assert(hashStruct(mail) == 0xc52c0ee5d84264471806290a3f2c4cecfc5490626bf912d01f240d7a274b371e);
        assert(verify(mail, v, r, s));
        return true;
    }
}
```

위 코드에서 signTypedData_v4은 `\x19\x01` 값, DOMAIN_SEPARATOR 값, hashStruct(mail)값


### 서명을 이용한 공격 사례

### 결론

---
### Appendix

* Personal_sign

`personal_sign`은 메타마스크에서만 사용되는 함수로서 `eth_sign`에서 트랜잭션을 가장한 공격을 막기 위해 `"\x19Ethereum Signed Message:\n" + len(message)`가 해싱 전 메시지 앞 부분에 추가되었다고 앞에서 설명하였다. 즉 기존에 문제가 되던 `eth_sign`은 그대로 유지하고
하지만 이는 Ethereum JSON-RPC API의 `eth_sign`을 보면 혼동될 수 있다. 왜냐하면 이더리움 API에서는 `eth_sign`이 메타마스크의 `personal_sign`과 동일하게 동작하도록 변경되었기 때문이다. 정리하자면, `eth_sign`은 이더리움 공식 API에선 `"\x19Ethereum Signed Message:\n" + len(message)`가 항상 메시지 앞에 붙도록 변경되었고, 메타마스크의 구현에선 옛날에 사용하던 `eth_sign`는 그대로 두고, `"\x19Ethereum Signed Message:\n" + len(message)`를 메시지 앞에 붙이는 `personal_sign` 함수를 별도로 구현하였다. 메타마스크의 `eth_sign`과 Ethereum JSON-RPC API의 `eth_sign`에 대한 혼동이 없길 바란다.    

* 재사용 공격(Replay Attack)

많은 어플리케이션에서 서명은 토큰을 스왑하거나, NFT를 거래하는 등에 사용된다. 서명을 재사용하는 경우에 대한 대비책은 각 어플리케이션이 적절히 테스트하고 방비책을 만들어야한다. 이는 서명 메시지에 nonce 값을 포함하는 등의 방식으로 replay attack을 방지할 수 있다.  

### Reference
1. [Ethereum:EIP712](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-712.md)
2. [MetaMask's Guide for signing the data]
3. [Koh Wei Jie's Blog post for EIP712]
4. [Hackernoon's Blog post for Wallet signature](https://medium.com/hackernoon/writing-for-blockchain-wallet-signature-request-messages-6ede721160d5)

[EIP712]: https://github.com/ethereum/EIPs/blob/master/EIPS/eip-712.md
[MetaMask's Guide for signing the data]: https://docs.metamask.io/guide/signing-data.html#a-brief-history
[hackernoon]:https://medium.com/hackernoon/writing-for-blockchain-wallet-signature-request-messages-6ede721160d5
[Koh Wei Jie's Blog post for EIP712]:https://medium.com/metamask/eip712-is-coming-what-to-expect-and-how-to-use-it-bb92fd1a7a26
[0xProtocol]: https://github.com/0xProject/0x-monorepo/issues/162#issuecomment-328263512
