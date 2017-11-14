# Description
The purpose of the contracts is to distribute tokens of the Jincor platform in accordence with the terms delared on the projects' website: https://jincor.com/ <br>
One interesting aspect of the crowdsale is the decision to update the price of the token sold denominated in ether depending on the current ETH/USD rate using an oracle, this brings specific security challenges that require careful consideration.

# Audited contracts

 * [PriceProvider.sol](./contracts/abstract/PriceProvider.sol)
 * [PriceReceiver.sol](./contracts/abstract/PriceReceiver.sol)
 * [BtcPriceProvider.sol](./contracts/BtcPriceProvider.sol)
 * [Burnable.sol](./contracts/Burnable.sol)
 * [EthPriceProvider.sol](./contracts/EthPriceProvider.sol)
 * [Haltable.sol](./contracts/Haltable.sol)
 * [InvestorWhiteList.sol](./contracts/InvestorWhiteList.sol)
 * [JincorToken.sol](./contracts/JincorToken.sol)
 * [JincorTokenICO.sol](./contracts/JincorTokenICO.sol)

# List of issuess

## 1. Referral bonus is counted towards reaching softcap/hardcap even if nobody receives it

In the doPurchase function of JincorTokenICO contrct, value of referral bonus is obtained by calling calculateReferralBonus function and the tokensSold variable is incremented by this bonus, this happens whether someone actually receives the bonus or not. In case nobody is eleigible to receive the bonus, these tokens are effectively removed from the available tokens for sale and remain in the ownership of the crowdsale owner.

###JincorTokenICO.sol / lines 214 - 247
```javascript  
function doPurchase() private icoActive inNormalState {
    require(!crowdsaleFinished);

    uint tokens = msg.value.mul(jcrEthRate);
    uint referralBonus = calculateReferralBonus(tokens);

    tokens = tokens.add(calculateBonus(tokens));

    uint newTokensSold = tokensSold.add(tokens).add(referralBonus);

    require(newTokensSold <= hardCap);

    if (!softCapReached && tokensSold < softCap && newTokensSold >= softCap) {
      softCapReached = true;
      SoftCapReached(softCap);
    }

    token.transfer(msg.sender, tokens);

    if (referralBonus > 0) {
      address referral = investorWhiteList.getReferralOf(msg.sender);
      if (referral != 0x0) {
        token.transfer(referral, referralBonus);
      }
    }

    collected = collected.add(msg.value);

    tokensSold = newTokensSold;

    deposited[msg.sender] = deposited[msg.sender].add(msg.value);

    NewContribution(msg.sender, tokens, msg.value);
  }
```


### Recommendations

Check if buyer has a valid referral address before adding referralBonus to the newTokensSold


## 2. Buyers buying at higher than declared dollar price due to rounding issue

Because solidity rounds down during division, calculating jcrEthRate in this function:

###JincorTokenICO.sol / lines 189 - 192
```javascript
  function receiveEthPrice(uint ethUsdPrice) external onlyEthPriceProvider {
    require(ethUsdPrice > 0);
    jcrEthRate = ethUsdPrice.div(jcrUsdRate);
  }
```
effectively strips down cents from current ETH/USD rate, so 300.99 becomes 300. This means that in the worst case, buyers can lose out on about 0.33% tokens at the current ETH price. If the price of ETH goes down, situation gets worse, at 200 ETH/USD the loss can be 0.5% at 100 ETH/USD, it becomes 1% and so on. Or to put it in other terms, a buyer who invests 1000 ETH can lose out on $1000 worth of tokens due to unfortunate timing. 


### Recommendations

To mitigate this I propose the following fixes. Store ethUsdPrice in cents upon receiving it:

```javascript
  function receiveEthPrice(uint ethUsdPrice) external onlyEthPriceProvider {
    require(ethUsdPrice > 0);
    ethUsdPriceStorage = ethUsdPrice;
  }
```
and move the ETH/USD conversion into the doPurchase function:

```javascript
    uint tokens = msg.value.mul(ethUsdPriceStorage).div(jcrUsdRate);
```
in this way, the maximal loss due to rounding is limited to $1 per buy.

### General note

If we care about precision, any calculation involving division should in general place the division as late as possible.

## 3. Overly specific if statements

In doPurchase function of JincorTokenICO contract, this condition:

###JincorTokenICO.sol / lines 226 - 229
```javascript
    if (!softCapReached && tokensSold < softCap && newTokensSold >= softCap) {
      softCapReached = true;
      SoftCapReached(softCap);
    }
```

could be replaced by:

```javascript
    if (!softCapReached && newTokensSold >= softCap) {
      softCapReached = true;
      SoftCapReached(softCap);
    }
```

Contrary to what's intuitive, making if statements too specific can sometimes make code vulnerable to new attack vectors. If there was any way to increment tokensSold variable without triggering the code above, the original variant would allow malicious actor to make succesfully finalizing the ICO impossible, the second variant while performing the same desired function is more fault tolerant.

## 4. Usage of contract storage in place of Logs

There's multiple instances where contract storage is used for storing values that have seemingly no function beyond providing users information about the contract's state. Examples are: jcrBtcRate or weiRefunded. In general, whenever a value doesn't have a function in the logic of the contract's code, it should be put in the log instead through firing an event, the accessibility of the data from the user's perspective is almost equivalent, especially if there's a custom interface used to track the current state of the contract.

## 5. Concerns around using a centralised oracle for setting ETH/USD exchange rate

Obtaining current ETH/USD exchange rate from API of a centralized exchange like Kraken might be problematic, things to consider

1) bad actor in Kraken organisation can compromise your ICO by sending an incorrect exchange rate
2) flash crash event similar to that which happened on GDAX couple months ago might make your tokens unreasonably expensive for a period of time
3) a bad actor running up the price on kraken in a low volume period might use the opportunity to get the tokens for cheap

### Recommendations
1) set some limits on the variation between successive updates of the price
2) cross-check the price from multiple sources

### Disclaimer

Since this approach to setting token price is innovative and wasn't, at least to my knowledge, used before, it is very hard to ascertain how material these concerns are.


# Summary
Beyond the issues mentioned, the contracts were also checked for overflow/underflow issues, DoS and re-entrancy vulnerabilities. None were discovered.

# Addendum
As of 2017-11-14 (the date of publication of this audit) all above mentioned problems were addressed in a satisfatory manner by the Jincor development team.


