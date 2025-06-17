# Hawk High - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Incorrect Teacher Payment Calculation Breaks Invariant](#H-01)
- ## Medium Risk Findings
    - ### [M-01.  Storage Collision in LevelTwo.sol Due to Mismatched Storage Layout](#M-01)
- ## Low Risk Findings
    - ### [L-01. Review Count Not Incremented, Allowing Unlimited Student Reviews](#L-01)
    - ### [L-02. Graduated event not emitted during major contract upgrade](#L-02)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #39

### Dates: May 1st, 2025 - May 8th, 2025

[See more contest details here](https://codehawks.cyfrin.io/c/2025-05-hawk-high)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 1
- Medium: 1
- Low: 2


# High Risk Findings

## <a id='H-01'></a>H-01. Incorrect Teacher Payment Calculation Breaks Invariant            



## Summary

The `LevelOne::graduateAndUpgrade()` function miscalculates teacher compensation by allocating **35% of the bursary to each teacher**, rather than **splitting 35% among all teachers**. This violates a critical protocol invariant and results in excessive fund distribution, potentially depleting the bursary.

## Vulnerability Details

The intent of the protocol is to allocate:

* 35% of the total bursary to **all teachers combined**
* 5% to the principal

However, the current implementation distributes 35% to **each teacher**:

```Solidity
uint256 payPerTeacher = (bursary * TEACHER_WAGE) / PRECISION;
```

This means:

* With 2 teachers: they receive 35% + 35% = 70% of bursary
* With 3 teachers: 105% — exceeding available funds
* And so on.

This breaks the invariant and drains the bursary.

## Impact

Can lead to:

* Bursary depletion
* Principal underpayment
* Protocol imbalance
* Causes overpayment to each teachers

## Proof of Concept (PoC Test)

```solidity
function test_wrongPayPerTeacher() public {
        uint256 bursary = 1000; // Mocked bursary amount
        uint256 totalTeachers = 2;

        // Simulate current (broken) logic
        uint256 payPerTeacher = (bursary * levelOneProxy.TEACHER_WAGE()) / levelOneProxy.PRECISION();

        for (uint256 n = 0; n < totalTeachers; n++) {
            bursary = bursary - payPerTeacher;
        }
        console2.log("Bursary: ",bursary); // should be 650 but it's 300 due to wrong payPerTeacher calculation.

        // bursary should be 650 as only 35% of bursary is paid to teachers
        assertLe(bursary, 650); // Breaking of invariant - teachers share of 35% of bursary

    }
```

## Tools Used

* Manual Code Review
* Custom test written in Foundry

## Recommendations

Update the teacher payment calculation to divide the allocated 35% evenly among all teachers:

```diff
 function graduateAndUpgrade(address _levelTwo, bytes memory) public onlyPrincipal {
        if (_levelTwo == address(0)) {
            revert HH__ZeroAddress();
        }

        uint256 totalTeachers = listOfTeachers.length;

-       uint256 payPerTeacher = (bursary * TEACHER_WAGE) / PRECISION;

+       uint256 totalTeacherShare = (bursary * TEACHER_WAGE) / PRECISION;
+       uint256 payPerTeacher = totalTeacherShare / totalTeachers;

        uint256 principalPay = (bursary * PRINCIPAL_WAGE) / PRECISION;

        _authorizeUpgrade(_levelTwo);

        for (uint256 n = 0; n < totalTeachers; n++) {
            usdc.safeTransfer(listOfTeachers[n], payPerTeacher);
        }

        usdc.safeTransfer(principal, principalPay);
    }

   
```

    
# Medium Risk Findings

## <a id='M-01'></a>M-01.  Storage Collision in LevelTwo.sol Due to Mismatched Storage Layout            



## Summary

Upgradeable contracts using proxy patterns (like UUPS) rely on a strict invariant: the storage layout must remain compatible across implementations. Any deviation or reordering can cause storage slot collisions, where a new variable overwrites an existing variable at the same storage slot.

## Vulnerability Details

Upgradeable contracts using proxy patterns (like UUPS) rely on a strict invariant: the storage layout must remain compatible across implementations. Any deviation or reordering can cause storage slot collisions, where a new variable overwrites an existing variable at the same storage slot.

The problem begins at slot 2, where schoolFees (from LevelOne) gets overwritten by sessionEnd (from LevelTwo). This continues down the layout, breaking multiple assumptions and corrupting the state.

This silent corruption is not reversible, and because Solidity does not enforce storage layout compatibility across upgrades, it will not raise any compile-time or deploy-time warnings.

## Impact

* State Corruption: Legitimate state variables from LevelOne are silently overwritten.
* If bursary or payout logic uses corrupted variables, funds could be misrouted or drained.
* Once upgraded, storage is permanently altered. There is no rollback without redeployment.

## Tools Used

* Manual code review
* Storage Layout Inspection

## Recommendations

* Ensure exact matching storage layout between LevelOne and LevelTwo.
* Declare \_gap like this to leave room for future variables:

```solidity
uint256[50] private __gap;
```


# Low Risk Findings

## <a id='L-01'></a>L-01. Review Count Not Incremented, Allowing Unlimited Student Reviews            



## Summary

reviewCount mapping is used in an access control check but never updated, allowing infinite reviews per student.

## Vulnerability Details

The `LevelOne.sol` contract has the following check in the `giveReview()` function:

```Solidity
require(reviewCount[_student] < 5, "Student review count exceeded!!!");
```

However, reviewCount\[\_student] is never incremented anywhere in the codebase. This means that for any \_student, reviewCount\[\_student] will always remain 0, effectively bypassing the review limit check. That is reviewCount\[\_student] will always be less than 5.

As a result:

* Teachers can call giveReview() unlimited times.
* They can arbitrarily reduce a student’s studentScore, since bad reviews deduct 10 points.
* The invariant “students must only be reviewed once per week, up to 4 times per session” is violated.

## Impact

* **Score manipulation:** A malicious or buggy teacher can give multiple bad reviews, tanking a student’s score below the cutOffScore, preventing them from graduating.
* **Bypass core logic:** The system assumes a fixed number of reviews per student per session. This bug breaks that invariant.
* **Potential griefing attack:** Repeated bad reviews could be used to block a student from progressing indefinitely.

<br />

## Tools Used

* Manual code review
* forge test suite

## POC

Here is the foundry test:

```solidity
function test_reviewCountFlaw() public {
         // Step 1: Dan enrolls
        vm.startPrank(dan);
        usdc.approve(address(levelOneProxy), schoolFees);
        levelOneProxy.enroll();
        vm.stopPrank();

        assertEq(levelOneProxy.isStudent(dan), true);
        assertEq(levelOneProxy.studentScore(dan), 100);

        vm.prank(principal);
        levelOneProxy.addTeacher(alice);

        vm.warp(block.timestamp + 1 weeks);
        // Step 2: Alice gives Dan a bad review every week, more than 4 times
        for(uint i = 0; i<10; i++) {
            vm.warp(block.timestamp + 1 weeks);
            vm.prank(alice);
            levelOneProxy.giveReview(dan, false);
        }

        // Step 3: Final assertion - studentScore drops below graduation threshold
        assertEq(levelOneProxy.studentScore(dan), 0);

    }
```

* This test demonstrates a critical flaw: since reviewCount is never incremented,
* teachers can repeatedly call giveReview every week, indefinitely penalizing the student.

## Recommendations

Increment reviewCount\[\_student] inside the giveReview() function:

```solidity
reviewCount[_student]++;
```

<br />

<br />

## <a id='L-02'></a>L-02. Graduated event not emitted during major contract upgrade            



## Summary

The Graduated event is defined but never emitted in the contract. In particular, the graduateAndUpgrade() function, which performs a significant contract upgrade and distributes funds, fails to emit this event, reducing transparency and observability for off-chain systems.

## Vulnerability Details

The contract defines an event:

```solidity
event Graduated(address indexed levelTwo);
```

However, this event is never emitted, not even in the critical `LevelOne::graduateAndUpgrade()` function, which performs a major state transition and contract upgrade.

This is a critical operation that changes the logic contract via UUPS, but it does not emit any event to signal this state transition to external observers.

## Impact

* No functional vulnerability or on-chain risk.
* Reduces transparency of major system transitions (like upgrades).
* Breaks best practices and limits integration with off-chain tools (e.g. The Graph, indexers, analytics dashboards, or governance tracking systems).
* Makes it harder for DApps or users to react to upgrades, increasing operational risk.

## Tools Used

* Manual code review

## Recommendations

Emit the Graduated event after a successful upgrade and before the function completes:

```Solidity
function graduateAndUpgrade(address _levelTwo, bytes memory) public onlyPrincipal {
        if (_levelTwo == address(0)) {
            revert HH__ZeroAddress();
        }
        uint256 totalTeachers = listOfTeachers.length;
        uint256 payPerTeacher = (bursary * TEACHER_WAGE) / PRECISION;
        uint256 principalPay = (bursary * PRINCIPAL_WAGE) / PRECISION;

        _authorizeUpgrade(_levelTwo);

        for (uint256 n = 0; n < totalTeachers; n++) {
            usdc.safeTransfer(listOfTeachers[n], payPerTeacher);
        }

        usdc.safeTransfer(principal, principalPay);
   ->   emit Graduated(_levelTwo);

    }
```



