---

title: My Cut Audit Report
author: QiteBlock
date: September 6, 2024

# My Cut Audit Report

Prepared by: QiteBlock
Lead Auditors:

- [QiteBlock]

Assisting Auditors:

- None

# Table of contents

<details>

<summary>See table</summary>

- [My Cut Audit Report](#my-cut-audit-report)
- [Table of contents](#table-of-contents)
- [About QiteBlock](#about-qiteblock)
- [Disclaimer](#disclaimer)
- [Risk Classification](#risk-classification)
- [Audit Details](#audit-details)
  - [Scope](#scope)
- [Protocol Summary](#protocol-summary)
  - [Roles](#roles)
- [Executive Summary](#executive-summary)
  - [Issues found](#issues-found)
- [Findings](#findings)
  - [High](#high)
    - [\[H-1\] Usage of participants length instead of claimant length to calculate claimantCut in `closePot` function](#h-1-usage-of-participants-length-instead-of-claimant-length)
    </details>
    </br>

# About QiteBlock

I'm a blockchain developer with 5 years of experience, specializing in building secure and scalable blockchain solutions for large companies. My work emphasizes rigorous security practices, including smart contract audits, cryptographic integrity, and ensuring compliance with industry standards. I have a deep understanding of decentralized technologies and consistently focus on mitigating risks, safeguarding digital assets, and delivering solutions that stand up to intense security scrutiny.

# Disclaimer

QiteBlock makes all effort to find as many vulnerabilities in the code in the given time period, but holds no responsibilities for the findings provided in this document. A security audit by the team is not an endorsement of the underlying business or product. The audit was time-boxed and the review of the code was solely on the security aspects of the Solidity implementation of the contracts.

# Risk Classification

|            |        | Impact |        |     |
| ---------- | ------ | ------ | ------ | --- |
|            |        | High   | Medium | Low |
|            | High   | H      | H/M    | M   |
| Likelihood | Medium | H/M    | M      | M/L |
|            | Low    | M      | M/L    | L   |

# Audit Details

**The findings described in this document correspond the following commit hash:**

```
946231db0fe717039429a11706717be568d03b54
```

## Scope

```
#-- interfaces
|   #-- IFlashLoanReceiver.sol
|   #-- IPoolFactory.sol
|   #-- ITSwapPool.sol
|   #-- IThunderLoan.sol
#-- protocol
|   #-- AssetToken.sol
|   #-- OracleUpgradeable.sol
|   #-- ThunderLoan.sol
#-- upgradedProtocol
    #-- ThunderLoanUpgraded.sol
```

# Protocol Summary

MyCut is a contest rewards distribution protocol which allows the set up and management of multiple rewards distributions, allowing authorized claimants 90 days to claim before the manager takes a cut of the remaining pool and the remainder is distributed equally to those who claimed in time!

## Roles

- Owner/Admin (Trusted) - Is able to create new Pots, close old Pots when the claim period has elapsed and fund Pots
- User/Player - Can claim their cut of a Pot

# Executive Summary

## Issues found

| Severity | Number of issues found |
| -------- | ---------------------- |
| High     | 1                      |
| Medium   | 0                      |
| Low      | 0                      |
| Info     | 0                      |
| Gas      | 0                      |
| Total    | 0                      |

# Findings

## High

### [H-1] `s_previousVoteEndTimeStamp` is not initiated

**Description:** `s_previousVoteEndTimeStamp` is not initiated :

In the constructor we don't initialized `s_previousVoteEndTimeStamp`. So `s_previousVoteEndTimeStamp` is 0. The following check in the function `selectPresident()` will always be true.

```solidity
    if (
        block.timestamp - s_previousVoteEndTimeStamp <=
        i_presidentalDuration
    ) {
        revert RankedChoice__NotTimeToVote();
    }
```

**Impact:** We can break the first election of president with that. We can `rankCandidate()` and `selectPresident()` directly after to make my voted candidate president directly.

**Proof of Code:**

<details>
<summary>Code</summary>
Add the following code to the `RankedChoiceTest.t.sol` file.

```javascript
    function testFirstElectionDoNotNeedToWaitMinimumDuration() public {
        assert(rankedChoice.getCurrentPresident() != candidates[0]);

        orderedCandidates = [candidates[0], candidates[1], candidates[2]];
        uint256 startingIndex = 0;
        uint256 endingIndex = 24;
        for (uint256 i = startingIndex; i < endingIndex; i++) {
            vm.prank(voters[i]);
            rankedChoice.rankCandidates(orderedCandidates);
        }

        vm.warp(block.timestamp + rankedChoice.getDuration());

        rankedChoice.selectPresident();
        assertEq(rankedChoice.getCurrentPresident(), candidates[0]);
    }
```

</details>

**Recommended Mitigation:**

Please initialize `s_previousVoteEndTimeStamp` to block.timestamp in the constructor.

## Medium

### [M-1] Potential Replay attacks on `rankCandidatesBySig`

**Description:** Potential Replay attacks on `rankCandidatesBySig` :

In the function `rankCandidatesBySig` we don't include any nonce and chainId to prevent the transaction to be re-executed.

If this contract were deployed on multiple blockchains, someone could replay a signature from one chain on another. Since signatures don’t contain any chain-specific information, this would allow the same vote to be counted multiple times across chains. This could be problematic in multi-chain systems.

If a voter sign a message and someone else sent the transaction for him. Then even he change himself the vote by calling `rankCandidate()` then a malicious person can change that by calling again `rankCandidatesBySig` and the change will rollback. So the voter can't change his vote.

**Impact:** This issue make the voter not able to change his vote once it's voted by calling `rankCandidatesBySig`. Futhermore if the smart contract is running on multiple blockchain. Then someone can vote for him in another blockchain.

**Recommended Mitigation:**

Please add chainId and a nonce in the signature to prevent that.

## Low

### [L-1] No Tie Handling

**Description:** No Tie Handling :

The current handling of ties may not be the most fair or transparent. The first candidate in the list with the fewest votes gets eliminated without considering the others that also tied.

**Impact:** The place of the candidate in the beginning list become very important when tie happens. I think it's not a wanted behaviour when tie happens.

**Proof of Code:**duras!5-edissa89-objurgarat?-adicis9!

<details>
<summary>Code</summary>
Add the following code to the `RankedChoiceTest.t.sol` file.

```javascript
    function testSelectPresidentTie() public {
        assert(rankedChoice.getCurrentPresident() != candidates[0]);

        orderedCandidates = [candidates[0], candidates[1], candidates[2]];
        uint256 startingIndex = 0;
        uint256 endingIndex = 10;
        for (uint256 i = startingIndex; i < endingIndex; i++) {
            vm.prank(voters[i]);
            rankedChoice.rankCandidates(orderedCandidates);
        }

        startingIndex = 11;
        endingIndex = 21;
        orderedCandidates = [candidates[3], candidates[1], candidates[4]];
        for (uint256 i = startingIndex; i < endingIndex; i++) {
            vm.prank(voters[i]);
            rankedChoice.rankCandidates(orderedCandidates);
        }

        vm.warp(block.timestamp + rankedChoice.getDuration());

        rankedChoice.selectPresident();
        assertEq(rankedChoice.getCurrentPresident(), candidates[3]);
    }
```

</details>

**Recommended Mitigation:**

Instead of always eliminating the first candidate, you could implement a more explicit and fair tie-breaking rule (such as random selection, or checking performance in earlier rounds).
