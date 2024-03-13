"Green Rabbit".

### [H-1] NO proper logic for handling funds. Funds are lost in contract

**Description:** The contract currently has no proper implementation and logic for handling users funds. This results in funds getting lost and users are able to mint new nfts for free everytime they call the mint function @ `ethereal.sol::mint`, which calls two other internal functions `ethereal.sol::_redeemEth` and `ethereal.sol::_redeemWstEth` . Funds are not deposited into the contract's address or any other EAO accounts, instead they are only being stored as abstract values in `metadata[_tokenId].balance`

**Impact:** Lose of funds and break in protocol logic.

**Proof Of Concept:** (Proof of concept)

```javascript
    function mint(
        uint256 _id,
        address _recipient
    ) external payable nonReentrant whenNotPaused returns (uint256 tokenId_) {
        require(msg.value == gems[_id].denomination, "Wrong ether amount");

        if (collections[gems[_id].collection].validator) {
            require(
                collections[gems[_id].collection].validatorAddress ==
                    msg.sender,
                "Not Validator"
            );
        }
        require(gems[_id].active, "No longer mintable");

        if (collections[gems[_id].collection].ethereum) {
            return _mintEth(_id, gems[_id].collection, _recipient);
        } else {
            return _mintWstEth(_id, gems[_id].collection, _recipient);
        }

    }
```

which holds the following internal functions

```javascript
     function _mintEth(
        uint256 _id,
        uint256 _collection,
        address _recipient
    ) internal returns (uint256 tokenId_) {

        tokenId_ = ++_tokenCounter;
        _safeMint(_recipient, tokenId_);
        circulatingGems++;

        metadata[tokenId_] = Metadata(msg.value, _collection, _id);

        emit GemMinted(tokenId_, _recipient, msg.value);
    }

    function _mintWstEth(
        uint256 _id,
        uint256 _collection,
        address _recipient
    ) internal returns (uint256 tokenId_) {
        tokenId_ = ++_tokenCounter;
        _safeMint(_recipient, tokenId_);
        circulatingGems++;

        uint256 preBalance = IwstETH(wstETH).balanceOf(address(this));

        (bool success, ) = wstETH.call{value: msg.value}("");
        require(success, "Failed to deposit Ether");

        metadata[tokenId_] = Metadata(
            IwstETH(wstETH).balanceOf(address(this)) - preBalance,
            _collection,
            _id
        );

        emit GemMinted(tokenId_, _recipient, msg.value);
    }

    The below test show that IwstETH account and the smart contract account are not receiving any funds from the minting processes.

    To run it -
    1. forge install openzepellin library
    2. run forge test

    function test_MintForUser() public {
        ethereal = new Ethereal();

        _createCollection("1st Collection", false, address(0), false, BASE_URL);
        _createGem(0, 100 * 1e18, 10, true);

        vm.deal(user1, 100 * 1e18);

        vm.prank(user1);
        console.log("before minting", ethereal.balanceOf(address(user1)));
        uint256 tokenId = ethereal.mint{value: 100 * 1e18}(0, user1);

        console.log("after minting", ethereal.balanceOf(user1));

        (uint256 balance, uint256 collection, uint256 gem) = ethereal.metadata(
            tokenId
        );

        uint256 contractBalance = ethereal.balanceOf(address(this));
        console.log("contract Balance", contractBalance);

        address payout = ethereal.payout();
        uint256 payoutAddress = ethereal.balanceOf(payout);
        console.log("Payout Balance", payoutAddress);
    }
```

**Recommended Mitigation:**

Once again, the code structure should be revamped with the right logical implementations

````


### [H-2] Contract owner unable to withdraw fees, this is a break of structure and leads to loss of fees

**Description:** The withdrawal function in the `ethereal.sol::withdrawfees` does not function as intended. This is as a result of lack of proper funds storage logic implementation in the code. The contract owner is unable to withdraw the fees which is a percentage amount of a minted asset value Inwhich was never even available.

**Impact:** Loss of funds and break in protocol logic implementation

**Proof Of Concept:** (Proof of concept)

```javascript
     function withdrawFees() external onlyOwner {
        (bool success, ) = wstETH.call{value: fees}("");
        require(success, "Transfer failed");
        fees = 0;
    }
````

    The below test show that contract owner is unable to withdraw the fees.

    To run it -
    1. forge install openzepellin library
    2. run forge test

```javascript
    function test_WithdrawFees() external {
        ethereal = new Ethereal();
        user1 = makeAddr("user1");
        user2 = makeAddr("user2");

        _createCollection("1st Collection", false, address(0), true, BASE_URL);
        _createGem(0, 100 * 1e18, 10, true);

        vm.deal(user1, 100 * 1e18); // Ensuring user1 has enough ETH
        vm.prank(user1);
        mintedTokenId = ethereal.mint{value: 100 * 1e18}(0, user1); // Mint a token and store its ID

        address wstETH = ethereal.wstETH();
        uint256 bal = ethereal.balanceOf(address(wstETH));
        console.log("bal", bal);

        // assertEq(ethereal.fees(), 0);
        console.log("fees before redeem", ethereal.fees());

        vm.prank(user1);
        ethereal.redeem(mintedTokenId);

        vm.prank(ethereal.payout());
        ethereal.withdrawFees();

        address payout = ethereal.payout();
        uint256 contractBalance = ethereal.balanceOf(payout);
        console.log("payout Balance", contractBalance);

    }
```

**Recommended Mitigation:**
TokenId should be placed right under the safeMint so that it does'nt increment in a situation where the minting fails.

```diff
    function _mintEth(
        uint256 _id,
        uint256 _collection,
        address _recipient
    ) internal returns (uint256 tokenId_) {

        _safeMint(_recipient, tokenId_);
+       tokenId_ = ++_tokenCounter;
        circulatingGems++;

        metadata[tokenId_] = Metadata(msg.value, _collection, _id);

        emit GemMinted(tokenId_, _recipient, msg.value);
    }

    function _mintWstEth(
        uint256 _id,
        uint256 _collection,
        address _recipient
    ) internal returns (uint256 tokenId_) {

        _safeMint(_recipient, tokenId_);
+       tokenId_ = ++_tokenCounter;
        circulatingGems++;

        uint256 preBalance = IwstETH(wstETH).balanceOf(address(this));

        (bool success, ) = wstETH.call{value: msg.value}("");
        require(success, "Failed to deposit Ether");

        metadata[tokenId_] = Metadata(
            IwstETH(wstETH).balanceOf(address(this)) - preBalance,
            _collection,
            _id
        );

        emit GemMinted(tokenId_, _recipient, msg.value);
    }

```

### [M-1] Increamenting Id when transaction fails could leaad to misinformation

**Description:** The `ethereal.sol::mint` Logs the `totalPrinted` even when the `_safeMint` fails due to an invalid tokenId.

**Impact:** This Can lead to misinformation and descripancies in ascertaining the correcte number of minted NFTs.

**Proof Of Concept:** (Proof of concept)

```javascript
     function _mintEth(
        uint256 _id,
        uint256 _collection,
        address _recipient
    ) internal returns (uint256 tokenId_) {

        tokenId_ = ++_tokenCounter;
        _safeMint(_recipient, tokenId_);
        circulatingGems++;

        metadata[tokenId_] = Metadata(msg.value, _collection, _id);

        emit GemMinted(tokenId_, _recipient, msg.value);
    }

    function _mintWstEth(
        uint256 _id,
        uint256 _collection,
        address _recipient
    ) internal returns (uint256 tokenId_) {
        tokenId_ = ++_tokenCounter;
        _safeMint(_recipient, tokenId_);
        circulatingGems++;

        uint256 preBalance = IwstETH(wstETH).balanceOf(address(this));

        (bool success, ) = wstETH.call{value: msg.value}("");
        require(success, "Failed to deposit Ether");

        metadata[tokenId_] = Metadata(
            IwstETH(wstETH).balanceOf(address(this)) - preBalance,
            _collection,
            _id
        );

        emit GemMinted(tokenId_, _recipient, msg.value);
    }
```

**Recommended Mitigation:**
TokenId should be placed right under the safeMint so that it does'nt increment in a situation where the minting fails.

```diff
    function _mintEth(
        uint256 _id,
        uint256 _collection,
        address _recipient
    ) internal returns (uint256 tokenId_) {

        _safeMint(_recipient, tokenId_);
+       tokenId_ = ++_tokenCounter;
        circulatingGems++;

        metadata[tokenId_] = Metadata(msg.value, _collection, _id);

        emit GemMinted(tokenId_, _recipient, msg.value);
    }

    function _mintWstEth(
        uint256 _id,
        uint256 _collection,
        address _recipient
    ) internal returns (uint256 tokenId_) {

        _safeMint(_recipient, tokenId_);
+       tokenId_ = ++_tokenCounter;
        circulatingGems++;

        uint256 preBalance = IwstETH(wstETH).balanceOf(address(this));

        (bool success, ) = wstETH.call{value: msg.value}("");
        require(success, "Failed to deposit Ether");

        metadata[tokenId_] = Metadata(
            IwstETH(wstETH).balanceOf(address(this)) - preBalance,
            _collection,
            _id
        );

        emit GemMinted(tokenId_, _recipient, msg.value);
    }

```

### [I-1] Best Practices and tips

**Description:**

- You should rename your contracts name to start with capital letter.
  `Ethereal.sol` instead of `ethereal.sol`.
- You should arrange your functions in an orderly manner, starting with public functions, external, internal, private.
