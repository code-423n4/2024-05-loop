# Report


## Gas Optimizations


| |Issue|Instances|
|-|:-|:-:|
| [GAS-1](#GAS-1) | Use assembly to check for `address(0)` | 1 |
| [GAS-2](#GAS-2) | `array[index] += amount` is cheaper than `array[index] = array[index] + amount` (or related variants) | 3 |
| [GAS-3](#GAS-3) | Using bools for storage incurs overhead | 2 |
| [GAS-4](#GAS-4) | State variables should be cached in stack variables rather than re-reading them from storage | 2 |
| [GAS-5](#GAS-5) | For Operations that will not overflow, you could use unchecked | 27 |
| [GAS-6](#GAS-6) | Avoid contract existence checks by using low level calls | 1 |
| [GAS-7](#GAS-7) | State variables only set in the constructor should be declared `immutable` | 2 |
| [GAS-8](#GAS-8) | Functions guaranteed to revert when called by normal users can be marked `payable` | 5 |
| [GAS-9](#GAS-9) | `++i` costs less gas compared to `i++` or `i += 1` (same for `--i` vs `i--` or `i -= 1`) | 1 |
| [GAS-10](#GAS-10) | Using `private` rather than `public` for constants, saves gas | 4 |
| [GAS-11](#GAS-11) | WETH address definition can be use directly | 1 |
### <a name="GAS-1"></a>[GAS-1] Use assembly to check for `address(0)`
*Saves 6 gas per instance*

*Instances (1)*:
```solidity
File: src/PrelaunchPoints.sol

420:         if (recipient != address(this) && recipient != address(0)) {

```
[Link to code](https://github.com/code-423n4/2024-05-loop/blob/main/src/PrelaunchPoints.sol)

### <a name="GAS-2"></a>[GAS-2] `array[index] += amount` is cheaper than `array[index] = array[index] + amount` (or related variants)
When updating a value in an array with arithmetic, using `array[index] += amount` is cheaper than `array[index] = array[index] + amount`.

This is because you avoid an additional `mload` when the array is stored in memory, and an `sload` when the array is stored in storage.

This can be applied for any arithmetic operation including `+=`, `-=`,`/=`,`*=`,`^=`,`&=`, `%=`, `<<=`,`>>=`, and `>>>=`.

This optimization can be particularly significant if the pattern occurs during a loop.

*Saves 28 gas for a storage array, 38 for a memory array*

*Instances (3)*:
```solidity
File: src/PrelaunchPoints.sol

172:             balances[_receiver][ETH] = balances[_receiver][_token] + _amount;

182:                 balances[_receiver][ETH] = balances[_receiver][_token] + _amount;

184:                 balances[_receiver][_token] = balances[_receiver][_token] + _amount;

```
[Link to code](https://github.com/code-423n4/2024-05-loop/blob/main/src/PrelaunchPoints.sol)

### <a name="GAS-3"></a>[GAS-3] Using bools for storage incurs overhead
Use uint256(1) and uint256(2) for true/false to avoid a Gwarmaccess (100 gas), and to avoid Gsset (20000 gas) when changing from ‘false’ to ‘true’, after having been ‘true’ in the past. See [source](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/58f635312aa21f947cae5f8578638a85aa2519f5/contracts/security/ReentrancyGuard.sol#L23-L27).

*Instances (2)*:
```solidity
File: src/PrelaunchPoints.sol

35:     mapping(address => bool) public isTokenAllowed;

48:     bool public emergencyMode;

```
[Link to code](https://github.com/code-423n4/2024-05-loop/blob/main/src/PrelaunchPoints.sol)

### <a name="GAS-4"></a>[GAS-4] State variables should be cached in stack variables rather than re-reading them from storage
The instances below point to the second+ access of a state variable within a function. Caching of a state variable replaces each Gwarmaccess (100 gas) with a much cheaper stack read. Other less obvious fixes/optimizations include having local memory caches of state variable structs, or having local caches of state variable contracts/addresses.

*Saves 100 gas per instance*

*Instances (2)*:
```solidity
File: src/PrelaunchPoints.sol

181:                 totalSupply = totalSupply + _amount;

478:         (bool success,) = payable(exchangeProxy).call{value: 0}(_swapCallData);

```
[Link to code](https://github.com/code-423n4/2024-05-loop/blob/main/src/PrelaunchPoints.sol)

### <a name="GAS-5"></a>[GAS-5] For Operations that will not overflow, you could use unchecked

*Instances (27)*:
```solidity
File: src/PrelaunchPoints.sol

4: import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";

5: import "@openzeppelin/contracts/utils/math/Math.sol";

7: import {ILpETH, IERC20} from "./interfaces/ILpETH.sol";

8: import {ILpETHVault} from "./interfaces/ILpETHVault.sol";

9: import {IWETH} from "./interfaces/IWETH.sol";

50:     mapping(address => mapping(address => uint256)) public balances; // User -> Token -> Balance

99:         loopActivation = uint32(block.timestamp + 120 days);

100:         startClaimDate = 4294967295; // Max uint32 ~ year 2107

107:                 i++;

171:             totalSupply = totalSupply + _amount;

172:             balances[_receiver][ETH] = balances[_receiver][_token] + _amount;

181:                 totalSupply = totalSupply + _amount;

182:                 balances[_receiver][ETH] = balances[_receiver][_token] + _amount;

184:                 balances[_receiver][_token] = balances[_receiver][_token] + _amount;

244:             uint256 userClaim = userStake * _percentage / 100;

246:             balances[msg.sender][_token] = userStake - userClaim;

251:             claimedAmount = address(this).balance - totalETH;

282:             totalSupply = totalSupply - lockedAmount;

304:         if (block.timestamp - loopActivation <= TIMELOCK) {

438:             p := add(p, 36) // Data: selector 4 + lenght data 32

441:             encodedPathLength := calldataload(add(p, 96)) // Get length of encodedPath (obtained through abi.encodePacked)

442:             inputToken := shr(96, calldataload(add(p, 128))) // Shift to the Right with 24 zeroes (12 bytes = 96 bits) to get address

443:             outputToken := shr(96, calldataload(add(p, add(encodedPathLength, 108)))) // Get last address of the hop

459:             inputToken := calldataload(add(p, 4)) // Read slot, selector 4 bytes

460:             outputToken := calldataload(add(p, 36)) // Read slot

461:             inputTokenAmount := calldataload(add(p, 68)) // Read slot

484:         boughtETHAmount = address(this).balance - boughtETHAmount;

```
[Link to code](https://github.com/code-423n4/2024-05-loop/blob/main/src/PrelaunchPoints.sol)

### <a name="GAS-6"></a>[GAS-6] Avoid contract existence checks by using low level calls
Prior to 0.8.10 the compiler inserted extra code, including `EXTCODESIZE` (**100 gas**), to check for contract existence for external function calls. In more recent solidity versions, the compiler will not insert these checks if the external call has a return value. Similar behavior can be achieved in earlier versions by using low-level calls, since low level calls never check for contract existence

*Instances (1)*:
```solidity
File: src/PrelaunchPoints.sol

312:         totalLpETH = lpETH.balanceOf(address(this));

```
[Link to code](https://github.com/code-423n4/2024-05-loop/blob/main/src/PrelaunchPoints.sol)

### <a name="GAS-7"></a>[GAS-7] State variables only set in the constructor should be declared `immutable`
Variables only set in the constructor and never edited afterwards should be marked as immutable, as it would avoid the expensive storage-writing operation in the constructor (around **20 000 gas** per variable) and replace the expensive storage-reading operations (around **2100 gas** per reading) to a less expensive value reading (**3 gas**)

*Instances (2)*:
```solidity
File: src/PrelaunchPoints.sol

96:         exchangeProxy = _exchangeProxy;

97:         WETH = IWETH(_wethAddress);

```
[Link to code](https://github.com/code-423n4/2024-05-loop/blob/main/src/PrelaunchPoints.sol)

### <a name="GAS-8"></a>[GAS-8] Functions guaranteed to revert when called by normal users can be marked `payable`
If a function modifier such as `onlyOwner` is used, the function will revert if a normal user tries to pay the function. Marking the function as `payable` will lower the gas cost for legitimate callers because the compiler will not include checks for whether a payment was provided.

*Instances (5)*:
```solidity
File: src/PrelaunchPoints.sol

303:     function convertAllETH() external onlyAuthorized {

324:     function setOwner(address _owner) external onlyAuthorized {

352:     function allowToken(address _token) external onlyAuthorized {

360:     function setEmergencyMode(bool _mode) external onlyAuthorized {

367:     function recoverERC20(address tokenAddress, uint256 tokenAmount) external onlyAuthorized {

```
[Link to code](https://github.com/code-423n4/2024-05-loop/blob/main/src/PrelaunchPoints.sol)

### <a name="GAS-9"></a>[GAS-9] `++i` costs less gas compared to `i++` or `i += 1` (same for `--i` vs `i--` or `i -= 1`)
Pre-increments and pre-decrements are cheaper.

For a `uint256 i` variable, the following is true with the Optimizer enabled at 10k:

**Increment:**

- `i += 1` is the most expensive form
- `i++` costs 6 gas less than `i += 1`
- `++i` costs 5 gas less than `i++` (11 gas less than `i += 1`)

**Decrement:**

- `i -= 1` is the most expensive form
- `i--` costs 11 gas less than `i -= 1`
- `--i` costs 5 gas less than `i--` (16 gas less than `i -= 1`)

Note that post-increments (or post-decrements) return the old value before incrementing or decrementing, hence the name *post-increment*:

```solidity
uint i = 1;  
uint j = 2;
require(j == i++, "This will be false as i is incremented after the comparison");
```
  
However, pre-increments (or pre-decrements) return the new value:
  
```solidity
uint i = 1;  
uint j = 2;
require(j == ++i, "This will be true as i is incremented before the comparison");
```

In the pre-increment case, the compiler has to create a temporary variable (when used) for returning `1` instead of `2`.

Consider using pre-increments and pre-decrements where they are relevant (meaning: not where post-increments/decrements logic are relevant).

*Saves 5 gas per instance*

*Instances (1)*:
```solidity
File: src/PrelaunchPoints.sol

107:                 i++;

```
[Link to code](https://github.com/code-423n4/2024-05-loop/blob/main/src/PrelaunchPoints.sol)

### <a name="GAS-10"></a>[GAS-10] Using `private` rather than `public` for constants, saves gas
If needed, the values can be read from the verified contract source code, or if there are multiple values there can be a single getter function that [returns a tuple](https://github.com/code-423n4/2022-08-frax/blob/90f55a9ce4e25bceed3a74290b854341d8de6afa/src/contracts/FraxlendPair.sol#L156-L178) of the values of all currently-public constants. Saves **3406-3606 gas** in deployment gas due to the compiler not having to create non-payable getter functions for deployment calldata, not having to store the bytes of the value outside of where it's used, and not adding another entry to the method ID table

*Instances (4)*:
```solidity
File: src/PrelaunchPoints.sol

28:     address public constant ETH = 0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE;

42:     bytes4 public constant UNI_SELECTOR = 0x803ba26d;

43:     bytes4 public constant TRANSFORM_SELECTOR = 0x415565b0;

47:     uint32 public constant TIMELOCK = 7 days;

```
[Link to code](https://github.com/code-423n4/2024-05-loop/blob/main/src/PrelaunchPoints.sol)

### <a name="GAS-11"></a>[GAS-11] WETH address definition can be use directly
WETH is a wrap Ether contract with a specific address in the Ethereum network, giving the option to define it may cause false recognition, it is healthier to define it directly.

    Advantages of defining a specific contract directly:
    
    It saves gas,
    Prevents incorrect argument definition,
    Prevents execution on a different chain and re-signature issues,
    WETH Address : 0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2

*Instances (1)*:
```solidity
File: src/PrelaunchPoints.sol

27:     IWETH public immutable WETH;

```
[Link to code](https://github.com/code-423n4/2024-05-loop/blob/main/src/PrelaunchPoints.sol)


## Non Critical Issues


| |Issue|Instances|
|-|:-|:-:|
| [NC-1](#NC-1) | Missing checks for `address(0)` when assigning values to address state variables | 2 |
| [NC-2](#NC-2) | `constant`s should be defined rather than using magic numbers | 11 |
| [NC-3](#NC-3) | Control structures do not follow the Solidity Style Guide | 2 |
| [NC-4](#NC-4) | Critical Changes Should Use Two-step Procedure | 1 |
| [NC-5](#NC-5) | Event missing indexed field | 5 |
| [NC-6](#NC-6) | Events that mark critical parameter changes should contain both the old and the new value | 2 |
| [NC-7](#NC-7) | Function ordering does not follow the Solidity style guide | 1 |
| [NC-8](#NC-8) | Functions should not be longer than 50 lines | 18 |
| [NC-9](#NC-9) | Lack of checks in setters | 3 |
| [NC-10](#NC-10) | Missing Event for critical parameters change | 1 |
| [NC-11](#NC-11) | Incomplete NatSpec: `@param` is missing on actually documented functions | 2 |
| [NC-12](#NC-12) | Use a `modifier` instead of a `require/if` statement for a special `msg.sender` actor | 1 |
| [NC-13](#NC-13) | Consider using named mappings | 2 |
| [NC-14](#NC-14) | `address`s shouldn't be hard-coded | 1 |
| [NC-15](#NC-15) | `require()` / `revert()` statements should have descriptive reason strings | 1 |
| [NC-16](#NC-16) | Take advantage of Custom Error's return value property | 14 |
| [NC-17](#NC-17) | Contract does not follow the Solidity style guide's suggested layout ordering | 1 |
| [NC-18](#NC-18) | Use Underscores for Number Literals (add an underscore every 3 digits) | 1 |
| [NC-19](#NC-19) | Event is missing `indexed` fields | 9 |
| [NC-20](#NC-20) | Constants should be defined rather than using magic numbers | 2 |
| [NC-21](#NC-21) | Variables need not be initialized to zero | 1 |
### <a name="NC-1"></a>[NC-1] Missing checks for `address(0)` when assigning values to address state variables

*Instances (2)*:
```solidity
File: src/PrelaunchPoints.sol

96:         exchangeProxy = _exchangeProxy;

325:         owner = _owner;

```
[Link to code](https://github.com/code-423n4/2024-05-loop/blob/main/src/PrelaunchPoints.sol)

### <a name="NC-2"></a>[NC-2] `constant`s should be defined rather than using magic numbers
Even [assembly](https://github.com/code-423n4/2022-05-opensea-seaport/blob/9d7ce4d08bf3c3010304a0476a785c70c0e90ae7/contracts/lib/TokenTransferrer.sol#L35-L39) can benefit from using readable constants instead of hex/numeric literals

*Instances (11)*:
```solidity
File: src/PrelaunchPoints.sol

99:         loopActivation = uint32(block.timestamp + 120 days);

100:         startClaimDate = 4294967295; // Max uint32 ~ year 2107

244:             uint256 userClaim = userStake * _percentage / 100;

438:             p := add(p, 36) // Data: selector 4 + lenght data 32

440:             recipient := calldataload(add(p, 64))

441:             encodedPathLength := calldataload(add(p, 96)) // Get length of encodedPath (obtained through abi.encodePacked)

442:             inputToken := shr(96, calldataload(add(p, 128))) // Shift to the Right with 24 zeroes (12 bytes = 96 bits) to get address

443:             outputToken := shr(96, calldataload(add(p, add(encodedPathLength, 108)))) // Get last address of the hop

459:             inputToken := calldataload(add(p, 4)) // Read slot, selector 4 bytes

460:             outputToken := calldataload(add(p, 36)) // Read slot

461:             inputTokenAmount := calldataload(add(p, 68)) // Read slot

```
[Link to code](https://github.com/code-423n4/2024-05-loop/blob/main/src/PrelaunchPoints.sol)

### <a name="NC-3"></a>[NC-3] Control structures do not follow the Solidity Style Guide
See the [control structures](https://docs.soliditylang.org/en/latest/style-guide.html#control-structures) section of the Solidity Style Guide

*Instances (2)*:
```solidity
File: src/PrelaunchPoints.sol

442:             inputToken := shr(96, calldataload(add(p, 128))) // Shift to the Right with 24 zeroes (12 bytes = 96 bits) to get address

489:                                 MODIFIERS

```
[Link to code](https://github.com/code-423n4/2024-05-loop/blob/main/src/PrelaunchPoints.sol)

### <a name="NC-4"></a>[NC-4] Critical Changes Should Use Two-step Procedure
The critical procedures should be two step process.

See similar findings in previous Code4rena contests for reference: <https://code4rena.com/reports/2022-06-illuminate/#2-critical-changes-should-use-two-step-procedure>

**Recommended Mitigation Steps**

Lack of two-step procedure for critical operations leaves them error-prone. Consider adding two step procedure on the critical functions.

*Instances (1)*:
```solidity
File: src/PrelaunchPoints.sol

324:     function setOwner(address _owner) external onlyAuthorized {

```
[Link to code](https://github.com/code-423n4/2024-05-loop/blob/main/src/PrelaunchPoints.sol)

### <a name="NC-5"></a>[NC-5] Event missing indexed field
Index event fields make the field more quickly accessible [to off-chain tools](https://ethereum.stackexchange.com/questions/40396/can-somebody-please-explain-the-concept-of-event-indexing) that parse events. This is especially useful when it comes to filtering based on an address. However, note that each index field costs extra gas during emission, so it's not necessarily best to index the maximum allowed per event (three fields). Where applicable, each `event` should use three `indexed` fields if there are three or more fields, and gas usage is not particularly of concern for the events in question. If there are fewer than three applicable fields, all of the applicable fields should be indexed.

*Instances (5)*:
```solidity
File: src/PrelaunchPoints.sol

58:     event Converted(uint256 amountETH, uint256 amountlpETH);

61:     event Recovered(address token, uint256 amount);

62:     event OwnerUpdated(address newOwner);

63:     event LoopAddressesUpdated(address loopAddress, address vaultAddress);

64:     event SwappedTokens(address sellToken, uint256 buyETHAmount);

```
[Link to code](https://github.com/code-423n4/2024-05-loop/blob/main/src/PrelaunchPoints.sol)

### <a name="NC-6"></a>[NC-6] Events that mark critical parameter changes should contain both the old and the new value
This should especially be done if the new value is not required to be different from the old value

*Instances (2)*:
```solidity
File: src/PrelaunchPoints.sol

324:     function setOwner(address _owner) external onlyAuthorized {
             owner = _owner;
     
             emit OwnerUpdated(_owner);

336:     function setLoopAddresses(address _loopAddress, address _vaultAddress)
             external
             onlyAuthorized
             onlyBeforeDate(loopActivation)
         {
             lpETH = ILpETH(_loopAddress);
             lpETHVault = ILpETHVault(_vaultAddress);
             loopActivation = uint32(block.timestamp);
     
             emit LoopAddressesUpdated(_loopAddress, _vaultAddress);

```
[Link to code](https://github.com/code-423n4/2024-05-loop/blob/main/src/PrelaunchPoints.sol)

### <a name="NC-7"></a>[NC-7] Function ordering does not follow the Solidity style guide
According to the [Solidity style guide](https://docs.soliditylang.org/en/v0.8.17/style-guide.html#order-of-functions), functions should be laid out in the following order :`constructor()`, `receive()`, `fallback()`, `external`, `public`, `internal`, `private`, but the cases below do not follow this pattern

*Instances (1)*:
```solidity
File: src/PrelaunchPoints.sol

1: 
   Current order:
   external lockETH
   external lockETHFor
   external lock
   external lockFor
   internal _processLock
   external claim
   external claimAndStake
   internal _claim
   external withdraw
   external convertAllETH
   external setOwner
   external setLoopAddresses
   external allowToken
   external setEmergencyMode
   external recoverERC20
   internal _validateData
   internal _decodeUniswapV3Data
   internal _decodeTransformERC20Data
   internal _fillQuote
   
   Suggested order:
   external lockETH
   external lockETHFor
   external lock
   external lockFor
   external claim
   external claimAndStake
   external withdraw
   external convertAllETH
   external setOwner
   external setLoopAddresses
   external allowToken
   external setEmergencyMode
   external recoverERC20
   internal _processLock
   internal _claim
   internal _validateData
   internal _decodeUniswapV3Data
   internal _decodeTransformERC20Data
   internal _fillQuote

```
[Link to code](https://github.com/code-423n4/2024-05-loop/blob/main/src/PrelaunchPoints.sol)

### <a name="NC-8"></a>[NC-8] Functions should not be longer than 50 lines
Overly complex code can make understanding functionality more difficult, try to further modularize your code to ensure readability 

*Instances (18)*:
```solidity
File: src/PrelaunchPoints.sol

121:     function lockETH(bytes32 _referral) external payable {

130:     function lockETHFor(address _for, bytes32 _referral) external payable {

140:     function lock(address _token, uint256 _amount, bytes32 _referral) external {

151:     function lockFor(address _token, uint256 _amount, address _for, bytes32 _referral) external {

163:     function _processLock(address _token, uint256 _amount, address _receiver, bytes32 _referral)

202:     function claim(address _token, uint8 _percentage, Exchange _exchange, bytes calldata _data)

217:     function claimAndStake(address _token, uint8 _percentage, Exchange _exchange, bytes calldata _data)

231:     function _claim(address _token, address _receiver, uint8 _percentage, Exchange _exchange, bytes calldata _data)

303:     function convertAllETH() external onlyAuthorized {

324:     function setOwner(address _owner) external onlyAuthorized {

336:     function setLoopAddresses(address _loopAddress, address _vaultAddress)

352:     function allowToken(address _token) external onlyAuthorized {

360:     function setEmergencyMode(bool _mode) external onlyAuthorized {

367:     function recoverERC20(address tokenAddress, uint256 tokenAmount) external onlyAuthorized {

393:     function _validateData(address _token, uint256 _amount, Exchange _exchange, bytes calldata _data) internal view {

429:     function _decodeUniswapV3Data(bytes calldata _data)

451:     function _decodeTransformERC20Data(bytes calldata _data)

472:     function _fillQuote(IERC20 _sellToken, uint256 _amount, bytes calldata _swapCallData) internal {

```
[Link to code](https://github.com/code-423n4/2024-05-loop/blob/main/src/PrelaunchPoints.sol)

### <a name="NC-9"></a>[NC-9] Lack of checks in setters
Be it sanity checks (like checks against `0`-values) or initial setting checks: it's best for Setter functions to have them

*Instances (3)*:
```solidity
File: src/PrelaunchPoints.sol

324:     function setOwner(address _owner) external onlyAuthorized {
             owner = _owner;
     
             emit OwnerUpdated(_owner);

336:     function setLoopAddresses(address _loopAddress, address _vaultAddress)
             external
             onlyAuthorized
             onlyBeforeDate(loopActivation)
         {
             lpETH = ILpETH(_loopAddress);
             lpETHVault = ILpETHVault(_vaultAddress);
             loopActivation = uint32(block.timestamp);
     
             emit LoopAddressesUpdated(_loopAddress, _vaultAddress);

360:     function setEmergencyMode(bool _mode) external onlyAuthorized {
             emergencyMode = _mode;

```
[Link to code](https://github.com/code-423n4/2024-05-loop/blob/main/src/PrelaunchPoints.sol)

### <a name="NC-10"></a>[NC-10] Missing Event for critical parameters change
Events help non-contract tools to track changes, and events prevent users from being surprised by changes.

*Instances (1)*:
```solidity
File: src/PrelaunchPoints.sol

360:     function setEmergencyMode(bool _mode) external onlyAuthorized {
             emergencyMode = _mode;

```
[Link to code](https://github.com/code-423n4/2024-05-loop/blob/main/src/PrelaunchPoints.sol)

### <a name="NC-11"></a>[NC-11] Incomplete NatSpec: `@param` is missing on actually documented functions
The following functions are missing `@param` NatSpec comments.

*Instances (2)*:
```solidity
File: src/PrelaunchPoints.sol

330:     /**
          * @notice Sets the lpETH contract address
          * @param _loopAddress address of the lpETH contract
          * @dev Can only be set once before 120 days have passed from deployment.
          *      After that users can only withdraw ETH.
          */
         function setLoopAddresses(address _loopAddress, address _vaultAddress)

364:     /**
          * @dev Allows the owner to recover other ERC20s mistakingly sent to this contract
          */
         function recoverERC20(address tokenAddress, uint256 tokenAmount) external onlyAuthorized {

```
[Link to code](https://github.com/code-423n4/2024-05-loop/blob/main/src/PrelaunchPoints.sol)

### <a name="NC-12"></a>[NC-12] Use a `modifier` instead of a `require/if` statement for a special `msg.sender` actor
If a function is supposed to be access-controlled, a `modifier` should be used instead of a `require/if` statement for more readability.

*Instances (1)*:
```solidity
File: src/PrelaunchPoints.sol

493:         if (msg.sender != owner) {

```
[Link to code](https://github.com/code-423n4/2024-05-loop/blob/main/src/PrelaunchPoints.sol)

### <a name="NC-13"></a>[NC-13] Consider using named mappings
Consider moving to solidity version 0.8.18 or later, and using [named mappings](https://ethereum.stackexchange.com/questions/51629/how-to-name-the-arguments-in-mapping/145555#145555) to make it easier to understand the purpose of each mapping

*Instances (2)*:
```solidity
File: src/PrelaunchPoints.sol

35:     mapping(address => bool) public isTokenAllowed;

50:     mapping(address => mapping(address => uint256)) public balances; // User -> Token -> Balance

```
[Link to code](https://github.com/code-423n4/2024-05-loop/blob/main/src/PrelaunchPoints.sol)

### <a name="NC-14"></a>[NC-14] `address`s shouldn't be hard-coded
It is often better to declare `address`es as `immutable`, and assign them via constructor arguments. This allows the code to remain the same across deployments on different networks, and avoids recompilation when addresses need to change.

*Instances (1)*:
```solidity
File: src/PrelaunchPoints.sol

28:     address public constant ETH = 0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE;

```
[Link to code](https://github.com/code-423n4/2024-05-loop/blob/main/src/PrelaunchPoints.sol)

### <a name="NC-15"></a>[NC-15] `require()` / `revert()` statements should have descriptive reason strings

*Instances (1)*:
```solidity
File: src/PrelaunchPoints.sol

476:         require(_sellToken.approve(exchangeProxy, _amount));

```
[Link to code](https://github.com/code-423n4/2024-05-loop/blob/main/src/PrelaunchPoints.sol)

### <a name="NC-16"></a>[NC-16] Take advantage of Custom Error's return value property
An important feature of Custom Error is that values such as address, tokenID, msg.value can be written inside the () sign, this kind of approach provides a serious advantage in debugging and examining the revert details of dapps such as tenderly.

*Instances (14)*:
```solidity
File: src/PrelaunchPoints.sol

168:             revert CannotLockZero();

175:                 revert TokenNotAllowed();

237:             revert NothingToClaim();

268:                 revert CurrentlyNotPossible();

271:                 revert NoLongerPossible();

279:             revert CannotWithdrawZero();

287:                 revert FailedToSendEther();

305:             revert LoopNotActivated();

369:             revert NotValidToken();

411:             revert WrongExchange();

480:             revert SwapCallFailed();

494:             revert NotAuthorized();

501:             revert CurrentlyNotPossible();

508:             revert NoLongerPossible();

```
[Link to code](https://github.com/code-423n4/2024-05-loop/blob/main/src/PrelaunchPoints.sol)

### <a name="NC-17"></a>[NC-17] Contract does not follow the Solidity style guide's suggested layout ordering
The [style guide](https://docs.soliditylang.org/en/v0.8.16/style-guide.html#order-of-layout) says that, within a contract, the ordering should be:

1) Type declarations
2) State variables
3) Events
4) Modifiers
5) Functions

However, the contract(s) below do not follow this ordering

*Instances (1)*:
```solidity
File: src/PrelaunchPoints.sol

1: 
   Current order:
   UsingForDirective.Math
   UsingForDirective.IERC20
   UsingForDirective.ILpETH
   VariableDeclaration.lpETH
   VariableDeclaration.lpETHVault
   VariableDeclaration.WETH
   VariableDeclaration.ETH
   VariableDeclaration.exchangeProxy
   VariableDeclaration.owner
   VariableDeclaration.totalSupply
   VariableDeclaration.totalLpETH
   VariableDeclaration.isTokenAllowed
   EnumDefinition.Exchange
   VariableDeclaration.UNI_SELECTOR
   VariableDeclaration.TRANSFORM_SELECTOR
   VariableDeclaration.loopActivation
   VariableDeclaration.startClaimDate
   VariableDeclaration.TIMELOCK
   VariableDeclaration.emergencyMode
   VariableDeclaration.balances
   EventDefinition.Locked
   EventDefinition.StakedVault
   EventDefinition.Converted
   EventDefinition.Withdrawn
   EventDefinition.Claimed
   EventDefinition.Recovered
   EventDefinition.OwnerUpdated
   EventDefinition.LoopAddressesUpdated
   EventDefinition.SwappedTokens
   ErrorDefinition.NothingToClaim
   ErrorDefinition.TokenNotAllowed
   ErrorDefinition.CannotLockZero
   ErrorDefinition.CannotWithdrawZero
   ErrorDefinition.FailedToSendEther
   ErrorDefinition.SwapCallFailed
   ErrorDefinition.WrongSelector
   ErrorDefinition.WrongDataTokens
   ErrorDefinition.WrongDataAmount
   ErrorDefinition.WrongRecipient
   ErrorDefinition.WrongExchange
   ErrorDefinition.LoopNotActivated
   ErrorDefinition.NotValidToken
   ErrorDefinition.NotAuthorized
   ErrorDefinition.CurrentlyNotPossible
   ErrorDefinition.NoLongerPossible
   FunctionDefinition.constructor
   FunctionDefinition.lockETH
   FunctionDefinition.lockETHFor
   FunctionDefinition.lock
   FunctionDefinition.lockFor
   FunctionDefinition._processLock
   FunctionDefinition.claim
   FunctionDefinition.claimAndStake
   FunctionDefinition._claim
   FunctionDefinition.withdraw
   FunctionDefinition.convertAllETH
   FunctionDefinition.setOwner
   FunctionDefinition.setLoopAddresses
   FunctionDefinition.allowToken
   FunctionDefinition.setEmergencyMode
   FunctionDefinition.recoverERC20
   FunctionDefinition.receive
   FunctionDefinition._validateData
   FunctionDefinition._decodeUniswapV3Data
   FunctionDefinition._decodeTransformERC20Data
   FunctionDefinition._fillQuote
   ModifierDefinition.onlyAuthorized
   ModifierDefinition.onlyAfterDate
   ModifierDefinition.onlyBeforeDate
   
   Suggested order:
   UsingForDirective.Math
   UsingForDirective.IERC20
   UsingForDirective.ILpETH
   VariableDeclaration.lpETH
   VariableDeclaration.lpETHVault
   VariableDeclaration.WETH
   VariableDeclaration.ETH
   VariableDeclaration.exchangeProxy
   VariableDeclaration.owner
   VariableDeclaration.totalSupply
   VariableDeclaration.totalLpETH
   VariableDeclaration.isTokenAllowed
   VariableDeclaration.UNI_SELECTOR
   VariableDeclaration.TRANSFORM_SELECTOR
   VariableDeclaration.loopActivation
   VariableDeclaration.startClaimDate
   VariableDeclaration.TIMELOCK
   VariableDeclaration.emergencyMode
   VariableDeclaration.balances
   EnumDefinition.Exchange
   ErrorDefinition.NothingToClaim
   ErrorDefinition.TokenNotAllowed
   ErrorDefinition.CannotLockZero
   ErrorDefinition.CannotWithdrawZero
   ErrorDefinition.FailedToSendEther
   ErrorDefinition.SwapCallFailed
   ErrorDefinition.WrongSelector
   ErrorDefinition.WrongDataTokens
   ErrorDefinition.WrongDataAmount
   ErrorDefinition.WrongRecipient
   ErrorDefinition.WrongExchange
   ErrorDefinition.LoopNotActivated
   ErrorDefinition.NotValidToken
   ErrorDefinition.NotAuthorized
   ErrorDefinition.CurrentlyNotPossible
   ErrorDefinition.NoLongerPossible
   EventDefinition.Locked
   EventDefinition.StakedVault
   EventDefinition.Converted
   EventDefinition.Withdrawn
   EventDefinition.Claimed
   EventDefinition.Recovered
   EventDefinition.OwnerUpdated
   EventDefinition.LoopAddressesUpdated
   EventDefinition.SwappedTokens
   ModifierDefinition.onlyAuthorized
   ModifierDefinition.onlyAfterDate
   ModifierDefinition.onlyBeforeDate
   FunctionDefinition.constructor
   FunctionDefinition.lockETH
   FunctionDefinition.lockETHFor
   FunctionDefinition.lock
   FunctionDefinition.lockFor
   FunctionDefinition._processLock
   FunctionDefinition.claim
   FunctionDefinition.claimAndStake
   FunctionDefinition._claim
   FunctionDefinition.withdraw
   FunctionDefinition.convertAllETH
   FunctionDefinition.setOwner
   FunctionDefinition.setLoopAddresses
   FunctionDefinition.allowToken
   FunctionDefinition.setEmergencyMode
   FunctionDefinition.recoverERC20
   FunctionDefinition.receive
   FunctionDefinition._validateData
   FunctionDefinition._decodeUniswapV3Data
   FunctionDefinition._decodeTransformERC20Data
   FunctionDefinition._fillQuote

```
[Link to code](https://github.com/code-423n4/2024-05-loop/blob/main/src/PrelaunchPoints.sol)

### <a name="NC-18"></a>[NC-18] Use Underscores for Number Literals (add an underscore every 3 digits)

*Instances (1)*:
```solidity
File: src/PrelaunchPoints.sol

100:         startClaimDate = 4294967295; // Max uint32 ~ year 2107

```
[Link to code](https://github.com/code-423n4/2024-05-loop/blob/main/src/PrelaunchPoints.sol)

### <a name="NC-19"></a>[NC-19] Event is missing `indexed` fields
Index event fields make the field more quickly accessible to off-chain tools that parse events. However, note that each index field costs extra gas during emission, so it's not necessarily best to index the maximum allowed per event (three fields). Each event should use three indexed fields if there are three or more fields, and gas usage is not particularly of concern for the events in question. If there are fewer than three fields, all of the fields should be indexed.

*Instances (9)*:
```solidity
File: src/PrelaunchPoints.sol

56:     event Locked(address indexed user, uint256 amount, address token, bytes32 indexed referral);

57:     event StakedVault(address indexed user, uint256 amount);

58:     event Converted(uint256 amountETH, uint256 amountlpETH);

59:     event Withdrawn(address indexed user, address token, uint256 amount);

60:     event Claimed(address indexed user, address token, uint256 reward);

61:     event Recovered(address token, uint256 amount);

62:     event OwnerUpdated(address newOwner);

63:     event LoopAddressesUpdated(address loopAddress, address vaultAddress);

64:     event SwappedTokens(address sellToken, uint256 buyETHAmount);

```
[Link to code](https://github.com/code-423n4/2024-05-loop/blob/main/src/PrelaunchPoints.sol)

### <a name="NC-20"></a>[NC-20] Constants should be defined rather than using magic numbers

*Instances (2)*:
```solidity
File: src/PrelaunchPoints.sol

442:             inputToken := shr(96, calldataload(add(p, 128))) // Shift to the Right with 24 zeroes (12 bytes = 96 bits) to get address

443:             outputToken := shr(96, calldataload(add(p, add(encodedPathLength, 108)))) // Get last address of the hop

```
[Link to code](https://github.com/code-423n4/2024-05-loop/blob/main/src/PrelaunchPoints.sol)

### <a name="NC-21"></a>[NC-21] Variables need not be initialized to zero
The default value for variables is zero, so initializing them to zero is superfluous.

*Instances (1)*:
```solidity
File: src/PrelaunchPoints.sol

104:         for (uint256 i = 0; i < length;) {

```
[Link to code](https://github.com/code-423n4/2024-05-loop/blob/main/src/PrelaunchPoints.sol)


## Low Issues


| |Issue|Instances|
|-|:-|:-:|
| [L-1](#L-1) | `approve()`/`safeApprove()` may revert if the current approval is not zero | 2 |
| [L-2](#L-2) | Some tokens may revert when zero value transfers are made | 3 |
| [L-3](#L-3) | Missing checks for `address(0)` when assigning values to address state variables | 2 |
| [L-4](#L-4) | Deprecated approve() function | 2 |
| [L-5](#L-5) | Empty `receive()/payable fallback()` function does not authenticate requests | 1 |
| [L-6](#L-6) | External call recipient may consume all transaction gas | 2 |
| [L-7](#L-7) | Solidity version 0.8.20+ may not work on other chains due to `PUSH0` | 1 |
| [L-8](#L-8) | Sweeping may break accounting if tokens with multiple addresses are used | 1 |
| [L-9](#L-9) | Unsafe ERC20 operation(s) | 2 |
### <a name="L-1"></a>[L-1] `approve()`/`safeApprove()` may revert if the current approval is not zero
- Some tokens (like the *very popular* USDT) do not work when changing the allowance from an existing non-zero allowance value (it will revert if the current approval is not zero to protect against front-running changes of approvals). These tokens must first be approved for zero and then the actual allowance can be approved.
- Furthermore, OZ's implementation of safeApprove would throw an error if an approve is attempted from a non-zero value (`"SafeERC20: approve from non-zero to non-zero allowance"`)

Set the allowance to zero immediately before each of the existing allowance calls

*Instances (2)*:
```solidity
File: src/PrelaunchPoints.sol

222:         lpETH.approve(address(lpETHVault), claimedAmount);

476:         require(_sellToken.approve(exchangeProxy, _amount));

```
[Link to code](https://github.com/code-423n4/2024-05-loop/blob/main/src/PrelaunchPoints.sol)

### <a name="L-2"></a>[L-2] Some tokens may revert when zero value transfers are made
Example: https://github.com/d-xo/weird-erc20#revert-on-zero-value-transfers.

In spite of the fact that EIP-20 [states](https://github.com/ethereum/EIPs/blob/46b9b698815abbfa628cd1097311deee77dd45c5/EIPS/eip-20.md?plain=1#L116) that zero-valued transfers must be accepted, some tokens, such as LEND will revert if this is attempted, which may cause transactions that involve other tokens (such as batch operations) to fully revert. Consider skipping the transfer if the amount is zero, which will also save gas.

*Instances (3)*:
```solidity
File: src/PrelaunchPoints.sol

177:             IERC20(_token).safeTransferFrom(msg.sender, address(this), _amount);

290:             IERC20(_token).safeTransfer(msg.sender, lockedAmount);

371:         IERC20(tokenAddress).safeTransfer(owner, tokenAmount);

```
[Link to code](https://github.com/code-423n4/2024-05-loop/blob/main/src/PrelaunchPoints.sol)

### <a name="L-3"></a>[L-3] Missing checks for `address(0)` when assigning values to address state variables

*Instances (2)*:
```solidity
File: src/PrelaunchPoints.sol

96:         exchangeProxy = _exchangeProxy;

325:         owner = _owner;

```
[Link to code](https://github.com/code-423n4/2024-05-loop/blob/main/src/PrelaunchPoints.sol)

### <a name="L-4"></a>[L-4] Deprecated approve() function
Due to the inheritance of ERC20's approve function, there's a vulnerability to the ERC20 approve and double spend front running attack. Briefly, an authorized spender could spend both allowances by front running an allowance-changing transaction. Consider implementing OpenZeppelin's `.safeApprove()` function to help mitigate this.

*Instances (2)*:
```solidity
File: src/PrelaunchPoints.sol

222:         lpETH.approve(address(lpETHVault), claimedAmount);

476:         require(_sellToken.approve(exchangeProxy, _amount));

```
[Link to code](https://github.com/code-423n4/2024-05-loop/blob/main/src/PrelaunchPoints.sol)

### <a name="L-5"></a>[L-5] Empty `receive()/payable fallback()` function does not authenticate requests
If the intention is for the Ether to be used, the function should call another function, otherwise it should revert (e.g. require(msg.sender == address(weth))). Having no access control on the function means that someone may send Ether to the contract, and have no way to get anything back out, which is a loss of funds. If the concern is having to spend a small amount of gas to check the sender against an immutable address, the code should at least have a function to rescue unused Ether.

*Instances (1)*:
```solidity
File: src/PrelaunchPoints.sol

380:     receive() external payable {}

```
[Link to code](https://github.com/code-423n4/2024-05-loop/blob/main/src/PrelaunchPoints.sol)

### <a name="L-6"></a>[L-6] External call recipient may consume all transaction gas
There is no limit specified on the amount of gas used, so the recipient can use up all of the transaction's gas, causing it to revert. Use `addr.call{gas: <amount>}("")` or [this](https://github.com/nomad-xyz/ExcessivelySafeCall) library instead.

*Instances (2)*:
```solidity
File: src/PrelaunchPoints.sol

284:             (bool sent,) = msg.sender.call{value: lockedAmount}("");

478:         (bool success,) = payable(exchangeProxy).call{value: 0}(_swapCallData);

```
[Link to code](https://github.com/code-423n4/2024-05-loop/blob/main/src/PrelaunchPoints.sol)

### <a name="L-7"></a>[L-7] Solidity version 0.8.20+ may not work on other chains due to `PUSH0`
The compiler for Solidity 0.8.20 switches the default target EVM version to [Shanghai](https://blog.soliditylang.org/2023/05/10/solidity-0.8.20-release-announcement/#important-note), which includes the new `PUSH0` op code. This op code may not yet be implemented on all L2s, so deployment on these chains will fail. To work around this issue, use an earlier [EVM](https://docs.soliditylang.org/en/v0.8.20/using-the-compiler.html?ref=zaryabs.com#setting-the-evm-version-to-target) [version](https://book.getfoundry.sh/reference/config/solidity-compiler#evm_version). While the project itself may or may not compile with 0.8.20, other projects with which it integrates, or which extend this project may, and those projects will have problems deploying these contracts/libraries.

*Instances (1)*:
```solidity
File: src/PrelaunchPoints.sol

2: pragma solidity 0.8.20;

```
[Link to code](https://github.com/code-423n4/2024-05-loop/blob/main/src/PrelaunchPoints.sol)

### <a name="L-8"></a>[L-8] Sweeping may break accounting if tokens with multiple addresses are used
There have been [cases](https://blog.openzeppelin.com/compound-tusd-integration-issue-retrospective/) in the past where a token mistakenly had two addresses that could control its balance, and transfers using one address impacted the balance of the other. To protect against this potential scenario, sweep functions should ensure that the balance of the non-sweepable token does not change after the transfer of the swept tokens.

*Instances (1)*:
```solidity
File: src/PrelaunchPoints.sol

367:     function recoverERC20(address tokenAddress, uint256 tokenAmount) external onlyAuthorized {

```
[Link to code](https://github.com/code-423n4/2024-05-loop/blob/main/src/PrelaunchPoints.sol)

### <a name="L-9"></a>[L-9] Unsafe ERC20 operation(s)

*Instances (2)*:
```solidity
File: src/PrelaunchPoints.sol

222:         lpETH.approve(address(lpETHVault), claimedAmount);

476:         require(_sellToken.approve(exchangeProxy, _amount));

```
[Link to code](https://github.com/code-423n4/2024-05-loop/blob/main/src/PrelaunchPoints.sol)


## Medium Issues


| |Issue|Instances|
|-|:-|:-:|
| [M-1](#M-1) | Contracts are vulnerable to fee-on-transfer accounting-related issues | 1 |
### <a name="M-1"></a>[M-1] Contracts are vulnerable to fee-on-transfer accounting-related issues
Consistently check account balance before and after transfers for Fee-On-Transfer discrepancies. As arbitrary ERC20 tokens can be used, the amount here should be calculated every time to take into consideration a possible fee-on-transfer or deflation.
Also, it's a good practice for the future of the solution.

Use the balance before and after the transfer to calculate the received amount instead of assuming that it would be equal to the amount passed as a parameter. Or explicitly document that such tokens shouldn't be used and won't be supported

*Instances (1)*:
```solidity
File: src/PrelaunchPoints.sol

177:             IERC20(_token).safeTransferFrom(msg.sender, address(this), _amount);

```
[Link to code](https://github.com/code-423n4/2024-05-loop/blob/main/src/PrelaunchPoints.sol)
