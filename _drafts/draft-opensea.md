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

이를 악용하면 `calldata`, `replacementPattern`, `staticExtradata`의 바이트 값들을 서로 간에 이동 (shifted)시키더라도 같은 해시 값을 얻게 된다. 그럼 공격자는 이 3개의 변수를 조작함으로써 최종적으로 어떤 공격을 수행할 수 있을지를 더 살펴보자. 우선 이 데이터들이 어떻게 활용되는지를 살펴봐야 한다. 아래는 Wyvern 컨트렉트에서 실제 NFT와 토큰의 거래가 매칭되는 [ExchangeCore.sol](https://github.com/ProjectWyvern/wyvern-ethereum/blob/master/contracts/exchange/ExchangeCore.sol#L665) 컨트렉트의 `atomicMatch` 함수이다.      

```solidity
/**
 * @dev Atomically match two orders, ensuring validity of the match, and execute all associated state transitions. Protected against reentrancy by a contract-global lock.
 * @param buy Buy-side order
 * @param buySig Buy-side order signature
 * @param sell Sell-side order
 * @param sellSig Sell-side order signature
 */
function atomicMatch(Order memory buy, Sig memory buySig, Order memory sell, Sig memory sellSig, bytes32 metadata)
    internal
    reentrancyGuard
{
    /* CHECKS */

    /* Ensure buy order validity and calculate hash if necessary. */
    bytes32 buyHash;
    if (buy.maker == msg.sender) {
        require(validateOrderParameters(buy));
    } else {
        buyHash = requireValidOrder(buy, buySig);
    }

    /* Ensure sell order validity and calculate hash if necessary. */
    bytes32 sellHash;
    if (sell.maker == msg.sender) {
        require(validateOrderParameters(sell));
    } else {
        sellHash = requireValidOrder(sell, sellSig);
    }

    /* Must be matchable. */
    require(ordersCanMatch(buy, sell));

    /* Target must exist (prevent malicious selfdestructs just prior to order settlement). */
    uint size;
    address target = sell.target;
    assembly {
        size := extcodesize(target)
    }
    require(size > 0);

    /* Must match calldata after replacement, if specified. */
    if (buy.replacementPattern.length > 0) {
      ArrayUtils.guardedArrayReplace(buy.calldata, sell.calldata, buy.replacementPattern);
    }
    if (sell.replacementPattern.length > 0) {
      ArrayUtils.guardedArrayReplace(sell.calldata, buy.calldata, sell.replacementPattern);
    }
    require(ArrayUtils.arrayEq(buy.calldata, sell.calldata));

    /* Retrieve delegateProxy contract. */
    OwnableDelegateProxy delegateProxy = registry.proxies(sell.maker);

    /* Proxy must exist. */
    require(delegateProxy != address(0));

    /* Assert implementation. */
    require(delegateProxy.implementation() == registry.delegateProxyImplementation());

    /* Access the passthrough AuthenticatedProxy. */
    AuthenticatedProxy proxy = AuthenticatedProxy(delegateProxy);

    /* EFFECTS */

    /* Mark previously signed or approved orders as finalized. */
    if (msg.sender != buy.maker) {
        cancelledOrFinalized[buyHash] = true;
    }
    if (msg.sender != sell.maker) {
        cancelledOrFinalized[sellHash] = true;
    }

    /* INTERACTIONS */

    /* Execute funds transfer and pay fees. */
    uint price = executeFundsTransfer(buy, sell);

    /* Execute specified call through proxy. */
    require(proxy.proxy(sell.target, sell.howToCall, sell.calldata));

    /* Static calls are intentionally done after the effectful call so they can check resulting state. */

    /* Handle buy-side static call if specified. */
    if (buy.staticTarget != address(0)) {
        require(staticCall(buy.staticTarget, sell.calldata, buy.staticExtradata));
    }

    /* Handle sell-side static call if specified. */
    if (sell.staticTarget != address(0)) {
        require(staticCall(sell.staticTarget, sell.calldata, sell.staticExtradata));
    }

    /* Log match event. */
    emit OrdersMatched(buyHash, sellHash, sell.feeRecipient != address(0) ? sell.maker : buy.maker, sell.feeRecipient != address(0) ? buy.maker : sell.maker, price, metadata);
}
```


<p style="text-align: center;">
	<img src="{{ site.url }}/assets/images/seaport/guardedArrayReplace.png" alt="Drawing" style="max-width: 80%; height: auto;"/>
	<figcaption align = "center"><b>Figure 2. guardedArrayReplace 함수 동작 예시</b></figcaption>
</p>

즉 위의 내용을 정리하자면, Wyvern v2.2의 서명 관련 취약점은 아래와 같다.

1. EIP-712이 나오기 전이기 때문에 서명에 자체적인 Order 구조체를 사용. Order 구조체는 동적 타입인 Bytes 변수인 Calldata, replacementPattern, staticExtradata를 갖고 있음
2. 여러 동적 타입 변수들을 갖고있음에도 불구하고, abi.encodePacked와 유사한 방식으로 데이터를 연속적으로 묶어 처리했기 때문에 해시 충돌이 발생할 수 있음. 공격자는 calldata, replacementPattern, staticExtradata 간에 비트 이동을 수행하더라도 같은 서명 값이 나오도록 조작할 수 있음.  
3. atomicMatch 함수에서는 buyer/seller의 calldata와 replacementPattern을 통해 새로운 바이트 배열(calldata)을 생성. 해당 바이트 배열의 첫번째 4 바이트에는 transferFrom(address,address,uint256)에 해당하는 함수 selector가 포함되어 있으며 (`0x23b872dd`), nft 판매자의 주소와 구매자의 주소 그리고 tokenId 값이 포함되어 있음
4. 원래는 calldata를 인자로 delegratecall을 호출함으로써 nft를 구매자에게 전송하는 `transferfrom` 함수가 수행되어야하지만, 공격자는 calldata, replacementPattern, staticExtradata 값을 조작함으로써 transferfrom 함수 대신에 `getApproved(uint)`가 호출되도록 calldta에 포함된 함수 selector를 변경할 수 있음. 이러한 조작은 replacementPattern과 guardedArrayReplace 함수를 이용하여 가능함.
5. 따라서 이러한 공격을 통해 nft 전송은 수행하지 않고, 구매자 혹은 offer를 수행한 사람들의 돈만 가져가는 것이 가능함. transferfrom 대신에 호출되는 `getApproved(uint256 tokenId)` 함수는 특정 토큰 id에 승인된 주소를 반환할 뿐인 함수이기 때문에 어떠한 상태 변화도 없이 정상적으로 처리된 것으로 간주됨.

#### guardedArrayReplace 메모리 overwrite 취약점

Wyvern 프로토콜에서 `atomicMatch` 함수는 replacementPattern을 이용하여 calldata를 변경한다. 이는 판매자가 nft를 판매하려고 리스팅하는 시점에서는 구매자가 정해져있지 않기 때문에, `transferFrom` 함수를 통해 nft를 수신할 주소를 확정하지 못하기 때문이다. 추후 transferfrom을 실행하려면 calldata에 수취인(구매자)의 주소가 포함되어야하고, 이를 위해 구매자가 replacementPattern을 이용해 `구매자 주소` 부분만 변경된 calldata를 생성한다. 예시 그림은 아래와 같으며 여기서는 판매자가 먼저 nft를 리스팅하는 시나리오를 기반으로 설명한다.

<그림3>

판매자가 리스팅을 수행하는 경우, order 구조체의 calldata에 첫 4 바이트엔 transferfrom(address,address,uint256)에 해당하는 함수 선택자 (function selector)인 `0x23b872dd`가 들어가며, 그 다음으로는 판매자의 address, 구매자의 address (이 시점엔 구매자를 모르므로 address(0)), 토큰 id 값이 순서대로 포함된다. 판매자의 order 구조체의 replacementPattern 바이트 변수는 calldata와 동일한 사이즈를 가진 bit mask이다. 해당 bit mask는 추후 교체될 구매자의 주소 자리에는 1 값을 갖고, 나머지 자리에는 0 값을 갖는다 (bytes로 표현될 때에는 0x00000000...FFFF...FFFF...0000의 형태이다).

판매자가 리스팅 후 구매자가 구매하려할 때에는 판매자의 calldata에 자신의 구매자 주소를 추가한다. replacementPattern 역시 판매자와 동일하게 구매자의 주소 부분만 1 값을 갖는 배열이다. atomicMatch 함수에서는 `require(ordersCanMatch(buy, sell));`를 통해 이를 확인한다.  

판매자가 nft를 먼저 리스팅하는것과는 반대로 구매자가 먼저 nft에 offer를 수행하는 경우를 가정해보자. 이 시나리오에서는 구매자의 calldata에서 판매자의 address 부분이 0으로 채워진다. offer를 제안하는 시점에 판매자의 주소는 알 수 있는데 왜 0으로 채우냐는 의문을 가질 수 있지만, 해당 nft가 다른 주인에게 이전되도 offer가 유효하게 유지되어야하므로 우리는 판매자의 주소를 거래 시점에 채우도록 한다. 여기서는 replacementPattern이 판매자의 주소 비트 부분이 1 값으로 채워지고, 추후 atomicMatch 함수에서 calldata의 판매자의 주소 부분이 교체된다.

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
