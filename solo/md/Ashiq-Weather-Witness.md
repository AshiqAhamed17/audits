# Weather Witness - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Locked Ether Due to Missing Withdrawal Functionality](#H-01)
    - ### [H-02. No validation of ownership when fulfilling mint requests causing NFTs being able to be stolen by anyone](#H-02)





# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #40

### Dates: May 15th, 2025 - May 22nd, 2025

[See more contest details here](https://codehawks.cyfrin.io/c/2025-05-weather-witness)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 2
- Medium: 0
- Low: 0


# High Risk Findings

## <a id='H-01'></a>H-01. Locked Ether Due to Missing Withdrawal Functionality            



# Root + Impact
The contract collects ETH through the requestMintWeatherNFT payable function, but it does not implement any withdrawal mechanism to retrieve this ETH. As a result, the ETH becomes permanently locked in the contract’s balance.

	•	Funds paid by users during minting are permanently locked and cannot be retrieved by the contract owner or protocol treasury.
	•	This undermines the financial utility of the protocol — if ETH was intended as payment or revenue, it becomes irretrievably stuck, resulting in lost funds.
	•	In a live deployment, this could lead to real monetary losses, protocol mismanagement, or a negative trust perception from users and contributors.

## Description

* The contract allows users to mint NFTs by sending ETH through the `WeatherNft::requestMintWeatherNFT` function, which is marked payable.
* However, the contract does not implement any mechanism for withdrawing the accumulated ETH, effectively locking the funds in the contract permanently.

<br />

```javascript
function requestMintWeatherNFT(
    string memory _pincode,
    string memory _isoCode,
    bool _registerKeeper,
    uint256 _heartbeat,
    uint256 _initLinkDeposit
) external payable returns (bytes32 _reqId) {
    require(
        msg.value == s_currentMintPrice,
        WeatherNft__InvalidAmountSent()
    );
    s_currentMintPrice += s_stepIncreasePerMint;
    ...
}
```

## Risk

**Likelihood**:

* Users regularly interact with the requestMintWeatherNFT function during normal minting operations.
* Each call to this function transfers ETH to the contract, but with no retrieval mechanism, this ETH will accumulate and be permanently inaccessible.

  <br />

**Impact**:

* ETH sent to the contract is locked and cannot be withdrawn or reused, resulting in a permanent loss of funds.
* If ETH was intended to be collected for protocol operations (e.g., treasury, revenue), this breaks financial functionality.

<br />

## Proof of Concept

```javascript
// Call mint function with correct ETH
weatherNft.requestMintWeatherNFT{value: weatherNft.s_currentMintPrice()}(
    "560001",   // pincode
    "IN",       // ISO country code
    false,      // no keeper
    0,          // heartbeat
    0           // no LINK
);

// Check that the contract received ETH
assert(address(weatherNft).balance > 0);

// But contract has no withdraw() function to access this balance
// Ether is permanently locked
```

## Recommended Mitigation

```diff
// Add a withdraw function to withdraw the funds form the contract
+ function withdraw() external onlyOwner {
+     uint256 balance = address(this).balance;
+     (bool success, ) = payable(msg.sender).call{value: balance}("");
+     require(success, "Withdraw failed");
+ }
```

    

# 

## <a id='H-02'></a>H-02. No validation of ownership when fulfilling mint requests causing NFTs being able to be stolen by anyone

## Description

* A user calls the `WeatherNft.sol::requestMintWeatherNFT` function to start the minting process. At this time the user also pays the mint price. The weather data is requested from an oracle and once this is received, the user calls the `WeatherNft.sol::fulfillMintRequest` function to mint the NFT.

* The `WeatherNft.sol::fulfillMintRequest` function does not check if the caller is the owner of the request. This means that anyone can call this function using the publicly readable request id and mint an NFT paid by some other user, effectively stealing their NFT.

```javascript
function fulfillMintRequest(bytes32 requestId) external {
    bytes memory response = s_funcReqIdToMintFunctionReqResponse[requestId].response;
    bytes memory err = s_funcReqIdToMintFunctionReqResponse[requestId].err;

    require(response.length > 0 || err.length > 0, WeatherNft__Unauthorized());

    if (response.length == 0 || err.length > 0) {
        return;
    }

    ...

@>    _mint(msg.sender, tokenId);

}
```

The only check that is done is whether there is a response for the weather data request.
When the nft is minted, the `msg.sender` is used as the owner of the NFT and not the original user who called the `requestMintWeatherNFT` function.

## Risk

**Likelihood**: High

* It is very easy to listen to the events emitted by the `requestMintWeatherNFT` function and get the requestId to use later to steal the NFT.

**Impact**: High

* Anyone can mint an NFT paid by someone else. This can lead to a loss of funds for the original user and a loss of trust in the system.

## Proof of Concept

This is a simple test where a user requests and pays for the minting of an NFT. The attacker listens to the events emitted by the `requestMintWeatherNFT` function and gets the requestId. The attacker then calls the `fulfillMintRequest` function with the stolen requestId and mints the NFT for themselves.

```javascript
function test_steal_weatherNFT() public {
    string memory pincode = "125001";
    string memory isoCode = "IN";
    bool registerKeeper = false;
    uint256 heartbeat = 12 hours;
    uint256 initLinkDeposit = 5e18;
    uint256 tokenId = weatherNft.s_tokenCounter();

    // User request and pays for the mint
    vm.startPrank(user);
    linkToken.approve(address(weatherNft), initLinkDeposit);
    vm.recordLogs();
    weatherNft.requestMintWeatherNFT{value: weatherNft.s_currentMintPrice()}(
        pincode,
        isoCode,
        registerKeeper,
        heartbeat,
        initLinkDeposit
    );
    vm.stopPrank();

    // Attacker steals the request id from the chain
    Vm.Log[] memory logs = vm.getRecordedLogs();
    bytes32 reqId;
    for (uint256 i; i < logs.length; i++) {
        if (logs[i].topics[0] == keccak256("WeatherNFTMintRequestSent(address,string,string,bytes32)")) {
            (, , , reqId) = abi.decode(logs[i].data, (address, string, string, bytes32));
            break;
        }
    }
    assert(reqId != bytes32(0));

    // Resolve the weather request
    (
        uint256 reqHeartbeat,
        address reqUser,
        bool reqRegisterKeeper,
        uint256 reqInitLinkDeposit,
        string memory reqPincode,
        string memory reqIsoCode
    ) = weatherNft.s_funcReqIdToUserMintReq(reqId);

    vm.prank(functionsRouter);
    bytes memory weatherResponse = abi.encode(WeatherNftStore.Weather.RAINY);
    weatherNft.handleOracleFulfillment(reqId, weatherResponse, "");

    // Attacker calls the fulfillMintRequest function with the stolen request id
    vm.prank(attacker);
    weatherNft.fulfillMintRequest(reqId);
    // The attacker now owns the NFT
    assertEq(weatherNft.ownerOf(tokenId), attacker);

    // attacker did not pay for the mint
    assertEq(linkToken.balanceOf(attacker), 1000e18);
}
```

## Recommended Mitigation

Two different solutions can be used to fix this issue:

1. **Check the owner of the requestId**: The `fulfillMintRequest` function should check if the caller is the owner of the requestId before minting the NFT. This can be done by adding a check like this:

```diff
+ require(msg.sender == s_funcReqIdToUserMintReq[requestId].user, WeatherNft__Unauthorized());
```

1. **Use the original requestId**: The `fulfillMintRequest` function should use the original requestId to mint the NFT.

```diff
- _mint(msg.sender, tokenId);
+ _mint(s_funcReqIdToUserMintReq[requestId].user, tokenId);
```




