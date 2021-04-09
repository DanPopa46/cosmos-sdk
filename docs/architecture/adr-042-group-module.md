# ADR 042: Group Module

## Changelog

- 2020/04/09: Initial Draft

## Status

Draft

## Abstract

This ADR defines the `x/group` module which allows the creation and management of on-chain multi-signature accounts and enables voting for message execution based on configurable decision policies.

## Context

The current multi-signature mechanism of the Cosmos SDK has certain limitations:
- Key rotation is not possible, although this can be solved with [account rekeying](adr-034-account-rekeying.md).
- Thresholds can't be changed.
- UX is not straightforward for non-technical users ([#5661](https://github.com/cosmos/cosmos-sdk/issues/5661)).
- It requires `legacy_amino` sign mode ([#8141](https://github.com/cosmos/cosmos-sdk/issues/8141)).

While the group module is not meant to be a total replacement for the current multi-signature accounts, it provides a solution to the limitations described above, with a more flexible key management system where keys can be added, updated or removed, as well as configurable thresholds.
It's meant to be used with other key management modules such as [`x/feegrant`](./adr-029-fee-grant-module.md) ans [`x/authz`](adr-030-authz-module.md) to simplify key management for individuals and organizations.

The current implementation of the group module can be found in https://github.com/regen-network/regen-ledger/tree/master/proto/regen/group/v1alpha1 and https://github.com/regen-network/regen-ledger/tree/master/x/group

## Decision

We propose merging the `x/group` module with its supporting [ORM/Table Store package](https://github.com/regen-network/regen-ledger/tree/master/orm) ([#7098](https://github.com/cosmos/cosmos-sdk/issues/7098)) into the Cosmos SDK and continuing development here. There will be a dedicated ADR for the ORM package.

### Group

A group is an aggregation of accounts with associated weights. It is not
an account and doesn't have a balance. It doesn't in and of itself have any
sort of voting or decision weight. It has an `admin` account which can manage members in the group.

Groups are stored in state as part of an ORM-based `groupTable` using the `GroupInfo` type. The `group_id` is an auto-increment integer.

```proto
message GroupInfo {

    // group_id is the unique ID of the group.
    uint64 group_id = 1;

    // admin is the account address of the group's admin.
    string admin = 2;
    
    // metadata is any arbitrary metadata to attached to the group.
    bytes metadata = 3;

    // version is used to track changes to a group's membership structure that
    // would break existing proposals. Whenever any members weight is changed,
    // or any member is added or removed this version is incremented and will
    // cause proposals based on older versions of this group to fail
    uint64 version = 4;

    // total_weight is the sum of the group members' weights.
    string total_weight = 5;
}
```

Group members are stored in a `groupMemberTable` using the `GroupMember` type:

```proto
message GroupMember {

    // group_id is the unique ID of the group.
    uint64 group_id = 1;

    // member is the member data.
    Member member = 2;
}

// Member represents a group member with an account address,
// non-zero weight and metadata.
message Member {

    // address is the member's account address.
    string address = 1;
    
    // weight is the member's voting weight that should be greater than 0.
    string weight = 2;
    
    // metadata is any arbitrary metadata to attached to the member.
    bytes metadata = 3;
}
```

### Group Account

A group account is an account associated with a group and a decision policy.
A group account does have a balance.

Group accounts are abstracted from groups because a single group may have
multiple decision policies for different types of actions. Managing group
membership separately from decision policies results in the least overhead
and keeps membership consistent across different policies. The pattern that
is recommended is to have a single master group account for a given group,
and then to create separate group accounts with different decision policies
and delegate the desired permissions from the master account to
those "sub-accounts" using the [`x/authz` module](adr-030-authz-module.md).

Group accounts are stored as part of the `groupAccountTable` and modeled by `GroupAccountInfo`.

```proto
message GroupAccountInfo {

    // address is the group account address.
    string address = 1;
    
    // group_id is the unique ID of the group.
    uint64 group_id = 2;

    // admin is the account address of the group admin.
    string admin = 3;
    
    // metadata is any arbitrary metadata to attached to the group account.
    bytes metadata = 4;

    // version is used to track changes to a group's GroupAccountInfo structure that
    // would create a different result on a running proposal.
    uint64 version = 5;

    // decision_policy specifies the group account's decision policy.
    google.protobuf.Any decision_policy = 6 [(cosmos_proto.accepts_interface) = "DecisionPolicy"];
}
```

The group account address is generated based on an auto-increment integer which is used to derive the group `RootModuleKey` into a `DerivedModuleKey`, as stated in [ADR-033](adr-033-protobuf-inter-module-comm.md#modulekeys-and-moduleids). The group account is added as a new `ModuleAccount` through `x/auth`.

### Decision Policy

A decision policy is the mechanism by which members of a group can vote on 
proposals.

All decision policies generally would have a minimum and maximum voting window.
The minimum voting window is the minimum amount of time that must pass in order
for a proposal to potentially pass, and it may be set to 0. The maximum voting
window is the maximum time that a proposal may be voted on before it is closed.
Both of these values must be less than a chain-wide max voting window parameter.

We define the `DecisionPolicy` interface that all decision policies must implement:

```go
type DecisionPolicy interface {
	codec.ProtoMarshaler

	ValidateBasic() error
	GetTimeout() types.Duration
	Allow(tally Tally, totalPower string, votingDuration time.Duration) (DecisionPolicyResult, error)
	Validate(g GroupInfo) error
}

type DecisionPolicyResult struct {
	Allow bool
	Final bool
}
```

#### Threshold decision policy

A threshold decision policy defines a threshold of yes votes (based on a tally
of voter weights) that must be achieved in order for a proposal to pass. For
this decision policy, abstain and veto are simply treated as no's.

```proto
message ThresholdDecisionPolicy {

    // threshold is the minimum weighted sum of yes votes that must be met or exceeded for a proposal to succeed.
    string threshold = 1;
    
    // timeout is the duration from submission of a proposal to the end of voting period
    // Within this times votes and exec messages can be submitted.
    google.protobuf.Duration timeout = 2 [(gogoproto.nullable) = false];
}
```

### Proposal

Any member of a group can submit a proposal for a group account to decide upon.
A proposal consists of a set of `sdk.Msg`s that will be executed if the proposal
passes as well as any metadata associated with the proposal.

Proposals are stored as part of the `proposalTable` using the `Proposal` type. The `proposal_id` is an auto-increment integer.

```proto
message Proposal {

    // proposal_id is the unique id of the proposal.
    uint64 proposal_id = 1;

    // address is the group account address.
    string address = 2;
    
    // metadata is any arbitrary metadata to attached to the proposal.
    bytes metadata = 3;

    // proposers are the account addresses of the proposers.
    repeated string proposers = 4;
    
    // submitted_at is a timestamp specifying when a proposal was submitted.
    google.protobuf.Timestamp submitted_at = 5 [(gogoproto.nullable) = false];
    
    // group_version tracks the version of the group that this proposal corresponds to.
    // When group membership is changed, existing proposals from previous group versions will become invalid.
    uint64 group_version = 6;

    // group_account_version tracks the version of the group account that this proposal corresponds to.
    // When a decision policy is changed, existing proposals from previous policy versions will become invalid.
    uint64 group_account_version = 7;

    // Status defines proposal statuses.
    enum Status {
        option (gogoproto.goproto_enum_prefix) = false;
        
        // An empty value is invalid and not allowed.
        STATUS_UNSPECIFIED = 0 [(gogoproto.enumvalue_customname) = "ProposalStatusInvalid"];
        
        // Initial status of a proposal when persisted.
        STATUS_SUBMITTED = 1 [(gogoproto.enumvalue_customname) = "ProposalStatusSubmitted"];
        
        // Final status of a proposal when the final tally was executed.
        STATUS_CLOSED = 2 [(gogoproto.enumvalue_customname) = "ProposalStatusClosed"];
        
        // Final status of a proposal when the group was modified before the final tally.
        STATUS_ABORTED = 3 [(gogoproto.enumvalue_customname) = "ProposalStatusAborted"];
    }

    // Status represents the high level position in the life cycle of the proposal. Initial value is Submitted.
    Status status = 8;

    // Result defines types of proposal results.
    enum Result {
        option (gogoproto.goproto_enum_prefix) = false;

        // An empty value is invalid and not allowed
        RESULT_UNSPECIFIED = 0 [(gogoproto.enumvalue_customname) = "ProposalResultInvalid"];
        
        // Until a final tally has happened the status is unfinalized
        RESULT_UNFINALIZED = 1 [(gogoproto.enumvalue_customname) = "ProposalResultUnfinalized"];
       
        // Final result of the tally
        RESULT_ACCEPTED = 2 [(gogoproto.enumvalue_customname) = "ProposalResultAccepted"];
        
        // Final result of the tally
        RESULT_REJECTED = 3 [(gogoproto.enumvalue_customname) = "ProposalResultRejected"];
    }

    // result is the final result based on the votes and election rule. Initial value is unfinalized.
    // The result is persisted so that clients can always rely on this state and not have to replicate the logic.
    Result result = 9;

    // vote_state contains the sums of all weighted votes for this proposal.
    Tally vote_state = 10 [(gogoproto.nullable) = false];

    // timeout is the timestamp of the block where the proposal execution times out. Header times of the votes and execution messages
    // must be before this end time to be included in the election. After the timeout timestamp the proposal can not be
    // executed anymore and should be considered pending delete.
    google.protobuf.Timestamp timeout = 11 [(gogoproto.nullable) = false];

    // ExecutorResult defines types of proposal executor results.
    enum ExecutorResult {
        option (gogoproto.goproto_enum_prefix) = false;
        
        // An empty value is not allowed.
        EXECUTOR_RESULT_UNSPECIFIED = 0  [(gogoproto.enumvalue_customname) = "ProposalExecutorResultInvalid"];
        
        // We have not yet run the executor.
        EXECUTOR_RESULT_NOT_RUN = 1 [(gogoproto.enumvalue_customname) = "ProposalExecutorResultNotRun"];
        
        // The executor was successful and proposed action updated state.
        EXECUTOR_RESULT_SUCCESS = 2 [(gogoproto.enumvalue_customname) = "ProposalExecutorResultSuccess"];
        
        // The executor returned an error and proposed action didn't update state.
        EXECUTOR_RESULT_FAILURE = 3 [(gogoproto.enumvalue_customname) = "ProposalExecutorResultFailure"];
    }

    // executor_result is the final result based on the votes and election rule. Initial value is NotRun.
    ExecutorResult executor_result = 12;

    // msgs is a list of Msgs that will be executed if the proposal passes.
    repeated google.protobuf.Any msgs = 13;
}

// Tally represents the sum of weighted votes.
message Tally {
    option (gogoproto.goproto_getters) = false;

    // yes_count is the weighted sum of yes votes.
    string yes_count = 1;
    
    // no_count is the weighted sum of no votes.
    string no_count = 2;
    
    // abstain_count is the weighted sum of abstainers.
    string abstain_count = 3;
    
    // veto_count is the weighted sum of vetoes.
    string veto_count = 4;
}
```

### Voting

There are four choices to choose while voting - yes, no, abstain and veto. Not
all decision policies will support them. Votes can contain some optional metadata.
During the voting window, accounts that have already voted may change their vote.
In the current implementation, the voting window begins as soon as a proposal
is submitted.

Votes are stored in the `voteTable`.

```proto
message Vote {

    // proposal is the unique ID of the proposal.
    uint64 proposal_id = 1;
    
    // voter is the account address of the voter.
    string voter = 2;
    
    // choice is the voter's choice on the proposal.
    Choice choice = 3;

    // metadata is any arbitrary metadata to attached to the vote.
    bytes metadata = 4;

    // submitted_at is the timestamp when the vote was submitted.
    google.protobuf.Timestamp submitted_at = 5 [(gogoproto.nullable) = false];
}
```

Voting internally updates the proposal `Status` and `Result`.

### Executing Proposals

Proposals will not be automatically executed by the chain in this current design,
but rather a user must submit a `Msg/Exec` transaction to attempt to execute the
proposal based on the current votes and decision policy. A future upgrade could
automate this and have the group account (or a fee granter) pay.

Inter-module communication introduced by [ADR-033](adr-033-protobuf-inter-module-comm.md) will be used to route a proposal's messages using the `DerivedModuleKey` corresponding to the proposal's group account. It can also temporarily support routing of non `ServiceMsg`s through the `sdk.Router` (see [#8864](https://github.com/cosmos/cosmos-sdk/issues/8864)).
For these messages to execute successfully, their signer should be set as the group account.

#### Changing Group Membership

In the current implementation, updating a group or a group account after submitting a proposal will make it invalid. It will simply fail if someone calls `Msg/Exec` and will eventually be garbage collected.

## Consequences

### Positive

- Improved UX for multi-signature accounts allowing key rotation and custom decision policies.

### Negative

### Neutral

- It uses ADR 033 so it will need to be implemented within the Cosmos SDK, but this doesn't imply necessarily any large refactoring of existing Cosmos SDK modules.
- It requires the ORM package.

## Further Discussions

- Convergence of `/group` and `x/gov` as both support proposals and voting: https://github.com/cosmos/cosmos-sdk/discussions/9066
- `x/group` possible future improvements:
  - Execute proposals on submission (https://github.com/regen-network/regen-ledger/issues/288)
  - Withdraw a proposal (https://github.com/regen-network/cosmos-modules/issues/41) 

## References

- Initial specification:
  - https://gist.github.com/aaronc/b60628017352df5983791cad30babe56#group-module
  - #5236
- Proposal to add `x/group` into the SDK: #7633
