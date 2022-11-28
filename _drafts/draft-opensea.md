---
layout: post
title:  "Opensea: Wyvern 프로토콜과 Seaport 코드 분석"
categories: Smart Contract
---
<!-- jekyll serve --drafts -->

### Introduction

test
### Wyvern v2.2 취약점

2022년 1분기에, 오픈씨가 사용하던 Wyvern v2.2 프로토콜 컨트랙트에서 크리티컬한 취약점이 발견되었다. 해당 취약점을 이용하면, 공격자는 offer를 수행한 사용자들에게서 ETH를 탈취할 수 있었다. 다행인 점은 문제가 발견되고 그동안의 로그를 분석해봤을 때 해당 취약점을 이용한 공격은 없었다고 한다. 오픈씨는 off-chain 서명을 통해 거래를 처리해왔는데, 해당 서명에서 사용하던 order 구조체에서 취약점이 발견되었다.  

[CTO of Opensa: A critical vulnerability in Wyvern Protocol]에서 이 취약점에 대해 자세히 설명하고있다. 해당 취약점은 Horton 원칙을 위반하면서 발생하였다. Hortion 원칙은 암호학 시스템의 디자인 원칙으로서, "말하는 바가 아니라 의미하는 바를 인증해라 (Authenticate what is being meant, not what is being said)"라는 표현이다. 이러한 원칙은 메시지 인증 코드(Message Authentication Codes, MAC)를 사용할 때 매우 중요한데, 앨리스가 밥에게 메시지 `m := a | b | c`을 MAC과 함께 보내 메시지를 인증하더라도, 밥이 해당 메시지를 컴포넌트 a, b, c로 정확히 분할하는 방법을 모른다면 잘못된 의미로 해석될 수 있다. 따라서 Hortion 원칙은 메시지 그 자체가 아닌 의미하는 바(meaning)를 인증하도록 한다. 예를 들어, MAC은 메시지 m의 바이트 스트링 자체만을 인증하므로, 메시지 m이 구성되는 방식 또한 인증하도록 하여야한다. 이는 [EIP712]처럼 메시지를 구조화, 형식화하고 메시지의 포맷을 서명에 포함하도록 함으로써 문제를 해결할 수 있다. 이제 Horton 원칙을 위반해서 발생했던 Wyvern v2.2 취약점에 대해 자세히 알아보자.

오픈씨에서는 off-chain 서명을 통해 판매와 구매를 수행한다. Wyvern v2.2에서는 [EIP712]가 적용되기 전으로 아래 코드에 나와있는 [order 구조체](https://github.com/ProjectWyvern/wyvern-ethereum/blob/master/contracts/exchange/ExchangeCore.sol#L92)에 서명을 수행한다.

```solidity
/* An order on the exchange. */
struct Order {
    /* Exchange address, intended as a versioning mechanism. */
    address exchange;
    /* Order maker address. */
    address maker;
    /* Order taker address, if specified. */
    address taker;
    /* Maker relayer fee of the order, unused for taker order. */
    uint makerRelayerFee;
    /* Taker relayer fee of the order, or maximum taker fee for a taker order. */
    uint takerRelayerFee;
    /* Maker protocol fee of the order, unused for taker order. */
    uint makerProtocolFee;
    /* Taker protocol fee of the order, or maximum taker fee for a taker order. */
    uint takerProtocolFee;
    /* Order fee recipient or zero address for taker order. */
    address feeRecipient;
    /* Fee method (protocol token or split fee). */
    FeeMethod feeMethod;
    /* Side (buy/sell). */
    SaleKindInterface.Side side;
    /* Kind of sale. */
    SaleKindInterface.SaleKind saleKind;
    /* Target. */
    address target;
    /* HowToCall. */
    AuthenticatedProxy.HowToCall howToCall;
    /* Calldata. */
    bytes calldata;
    /* Calldata replacement pattern, or an empty byte array for no replacement. */
    bytes replacementPattern;
    /* Static call target, zero-address for no static call. */
    address staticTarget;
    /* Static call extra data. */
    bytes staticExtradata;
    /* Token used to pay for the order, or the zero-address as a sentinel value for Ether. */
    address paymentToken;
    /* Base price of the order (in paymentTokens). */
    uint basePrice;
    /* Auction extra parameter - minimum bid increment for English auctions, starting/ending price difference. */
    uint extra;
    /* Listing timestamp. */
    uint listingTime;
    /* Expiration timestamp - 0 for no expiry. */
    uint expirationTime;
    /* Order salt, used to prevent duplicate hashes. */
    uint salt;
}

/**
 * @dev Hash an order, returning the canonical order hash, without the message prefix
 * @param order Order to hash
 * @return Hash of order
 */
function hashOrder(Order memory order)
    internal
    pure
    returns (bytes32 hash)
{
    /* Unfortunately abi.encodePacked doesn't work here, stack size constraints. */
    uint size = sizeOf(order);
    bytes memory array = new bytes(size);
    uint index;
    assembly {
        index := add(array, 0x20)
    }
    index = ArrayUtils.unsafeWriteAddress(index, order.exchange);
    index = ArrayUtils.unsafeWriteAddress(index, order.maker);
    index = ArrayUtils.unsafeWriteAddress(index, order.taker);
    index = ArrayUtils.unsafeWriteUint(index, order.makerRelayerFee);
    index = ArrayUtils.unsafeWriteUint(index, order.takerRelayerFee);
    index = ArrayUtils.unsafeWriteUint(index, order.makerProtocolFee);
    index = ArrayUtils.unsafeWriteUint(index, order.takerProtocolFee);
    index = ArrayUtils.unsafeWriteAddress(index, order.feeRecipient);
    index = ArrayUtils.unsafeWriteUint8(index, uint8(order.feeMethod));
    index = ArrayUtils.unsafeWriteUint8(index, uint8(order.side));
    index = ArrayUtils.unsafeWriteUint8(index, uint8(order.saleKind));
    index = ArrayUtils.unsafeWriteAddress(index, order.target);
    index = ArrayUtils.unsafeWriteUint8(index, uint8(order.howToCall));
    index = ArrayUtils.unsafeWriteBytes(index, order.calldata);
    index = ArrayUtils.unsafeWriteBytes(index, order.replacementPattern);
    index = ArrayUtils.unsafeWriteAddress(index, order.staticTarget);
    index = ArrayUtils.unsafeWriteBytes(index, order.staticExtradata);
    index = ArrayUtils.unsafeWriteAddress(index, order.paymentToken);
    index = ArrayUtils.unsafeWriteUint(index, order.basePrice);
    index = ArrayUtils.unsafeWriteUint(index, order.extra);
    index = ArrayUtils.unsafeWriteUint(index, order.listingTime);
    index = ArrayUtils.unsafeWriteUint(index, order.expirationTime);
    index = ArrayUtils.unsafeWriteUint(index, order.salt);
    assembly {
        hash := keccak256(add(array, 0x20), size)
    }
    return hash;
}
```

위 코드에서 order는 거래에 사용되는 매우 많은 데이터들을 담고 있는데, 이 order는 구매자 혹은 판매자에 의해 해시함수를 거쳐 서명된다. 문제가 되는 부분은 `calldata`, `replacementPattern`, `staticExtradata`와 같은 Bytes 데이터들이다. 위 코드에서 order 구조체를 해싱하는 `hashOrder` 함수를 살펴보자. 아래 함수는 order 구조체에 대해 `keccak256(abi.encodePacked(hashOrder))`를 수행한 것과 동일하다. [abi.encodePacked()](https://docs.soliditylang.org/en/latest/abi-spec.html#non-standard-packed-mode)는 Non-standard Packed Mode로서 Figure 1 처럼, string과 bytes와 같은 동적 타입들이 그대로 다른 값의 뒤에 이어진다. 문제는 order는 Bytes 동적 타입 데이터를 여러개 갖고 있기 때문에, 여기서 공격 취약점이 발생한다.

<p style="text-align: center;">
	<img src="{{ site.url }}/assets/images/seaport/encodepacked.png" alt="Drawing" style="max-width: 80%; height: auto;"/>
	<figcaption align = "center"><b>Figure 1. abi.encodePacked() 예시</b></figcaption>
</p>

우리는 order 구조체에서 `calldata`, `replacementPattern`, `staticExtradata` 3개의 Bytes 데이터가 존재한다는 것을 보았다. 어떻게 이를 통해 공격이 수행될 수 있을지를 보자. 예시로 변수 string a = "test", string b = "collision"이라고 해보자. `keccak256(abi.encodePacked(a, b)) = keccak256(abi.encodePacked("test", "collision")) = keccak256(abi.encodePacked("te", "stcollision"))`와 동일하다. 즉, 동적 타입 a와 b 값이 달라지더라도 인코딩된 값이 동일하기 때문에 같은 해시 값이 나오게 된다.

>이는 `abi.encodePacked`을 서명, 인증 및 데이터 무결성 체크에 확인할 때 주의해야하는 이유이다.
`keccak256(abi.encodePacked(a, b))`를 계산할 때 a와 b가 string, bytes와 같은 동적 타입이라면 a와 b의 값 일부를 서로 간에 이동시켜도 동일한 결과 값을 얻을 수 있다. 따라서 특별한 이유가 없다면 abi.encode 사용을 권고한다. 이러한 경고는 [Solidity Docs](https://docs.soliditylang.org/en/latest/abi-spec.html#non-standard-packed-mode)에 non-standard mode의 주의사항으로 나와있으며 [SWC-133](https://swcregistry.io/docs/SWC-133)에도 이와 같은 해시 충돌에 대한 주의사항이 나와있다.   

이를 악용하면 `calldata`, `replacementPattern`, `staticExtradata`의 바이트 값들을 서로 간에 이동 (shifted)시키더라도 같은 해시 값을 얻게 된다. 그럼 공격자는 이 3개의 변수를 조작함으로써 최종적으로 어떤 공격을 수행할 수 있을지를 더 살펴보자. 우선 이 데이터들이 어떻게 활용되는지를 살펴봐야 한다.     

#### Calldata와 Replacement 패턴


해당 이슈는 Wyvern v2.3에서 [EIP712]를 사용함으로써 해결되었다.

### Yul을 통한 Seaport의 가스 최적화

### Reference
1. [CTO of Opensa: A critical vulnerability in Wyvern Protocol]
2. [Haechi Audit: Wyvern v2.2 1-day Vulnerabilities]
3. [EIP712]

[Haechi Audit: Wyvern v2.2 1-day Vulnerabilities]: https://blog.audit.haechi.io/wyvern_v2_2_1_1day_vulnerabilities
[EIP712]:https://github.com/ethereum/EIPs/blob/master/EIPS/eip-712.md
[CTO of Opensa: A critical vulnerability in Wyvern Protocol]: https://nft.mirror.xyz/VdF3BYwuzXgLrJglw5xF6CHcQfAVbqeJVtueCr4BUzs
