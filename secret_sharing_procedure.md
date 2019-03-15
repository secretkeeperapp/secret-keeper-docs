### Summary

The purpose of this document is to specify a secret sharing and DMS application with a general set of actions. These actions should allow a testator (i.e. secret creator) to manage secrets in use cases that are more complicated than unlocking a single secret for one or multiple beneficiaries (i.e. secret recipients) after a time delay has passed.

### Use cases


A few motivating examples:

1. I have two family members, and a close family friend. I would like to leave assets to the two family members, but if they are unable to claim them in the case of an emergency, I would like the close family friend to have control over the assets.

2. I’m a board member of an exchange called TridegaBS, and I’m worried about the CTO running off with the only set of private keys and faking his death. I want to split up the keys among the other shareholders, so that a quorum of shareholders can choose to retrieve the keys and have it revealed to all of them at once, if a month has passed since the CTO last checked in.

3. [possibly unrealistic example, also this is just like SafeHaven] I have a child who’s twelve years old and I want him to have my cryptoassets when I pass away. I have a lawyer, but I don’t want to give him the secret to hold. I’d like to give the lawyer an easy way to release access to the secret without holding the secret itself, while preventing my child from accessing the secret until two years after I stop checking in.

4. I have three children, and I want two of the three children to confirm that the secret should be released, before it is released to all three of them at once.

5. I have a recovery account that I know the address of, and I keep the keys to that account etched in steel and buried in a canister under a tree in my backyard. I want to set up a mechanism to give access to the passphrase of my primary account to this recovery account 6 months after I stop checking in from the primary account.

6. The same example as the above, except I want my three friends to confirm that access should be released to the recovery account, in case someone digs up the keys to that account. Of course, these friends wouldn’t have access to the account, or know that what they’re confirming is the release of access to a secret to my recovery account.

7. [possibly too complicated] I want to keep my Trezor passphrase with my friends in case I forget it, and I don’t use a recovery account. I have two friends I trust very well, and three that I’m less close to. I’d like a scheme in which one of the friends from the first group needs to combine a part from two of the friends in the second group, and the secret is released to only the friend in the first group. 

**NOTE**: With all of these examples, the beneficiaries could either have the secret parts themselves to be able to recover the secret without SecretKeeper, or they could only have control over whether or not the secret part was submitted for combination (which doesn’t necessarily need to be executed if Enigma stores the secret directly). There are more complicated schemes here where beneficiaries could choose to combine secret parts with only certain other beneficiaries, and this may not be worth exploring as part of the access control capabilities at this point.

TODO: think about adding incentivization to this as well, this is probably more complicated than it’s worth to do in code.

### Steps

Here are a general set of steps that attempt to capture the use cases above. At a high level, a secret sharing scheme is used to split the secret into parts. 

The parts are assigned to beneficiaries, along with a release condition that needs to be fulfilled for those parts to be claimed. 

Once the release condition is fulfilled, access control policies will be responsible for determining the logic of combining and releasing the secret.

#### 1. Specify beneficiaries

For each beneficiary:

* Input their Ethereum public address

#### 2. Generate secret parts

    * Input the secret

    * Input total number of parts *n*

    * Input threshold *k*

    * Generate parts locally on the client with (*k*,*n*) threshold scheme (Shamir’s secret-sharing)

        1. TODOs:

        2. think about generating parts locally vs. in an Enigma secret contract. There doesn’t seem to be a significant advantage to generating these parts with Enigma, because modifications to *k* or *n* will require all of the parts to be generated again anyway, and access control changes shouldn’t change the underlying secret parts.

        3. Think about whether Enigma should store the actual secret as well (which would be necessary if it had to generate parts in a secret contract)

        4. Think about whether a publicly verifiable secret sharing (PVSS) scheme is necessary to prevent counterfeit parts (if beneficiaries have access to the secret parts themselves). See [here](https://ethresear.ch/t/security-considerations-for-shamirs-secret-sharing/4294).

    * **NOTE**: The default setting is to have *n* equal to the number of beneficiaries, and *k* equal to 1.

#### 3. Assign parts to beneficiaries

    * For each beneficiary, input the secret parts that this beneficiary will have claim to, once the release conditions are met.

        5. As of now, parts can be reused across multiple beneficiaries.

    * For each beneficiary, for the entire set of secret parts, specify whether the beneficiary is able to retrieve the secret parts themselves, or only have control over whether or not the part is submitted for combination. 

    * TODO: Think about if keeping the parts from the beneficiary gives significant advantages for control over the secret.

#### 4. Define release conditions

    * For each beneficiary, define a release condition that must be fulfilled for the secret parts to be viewed (or controlled) by that beneficiary.

    * Example conditions that would be available:

        * Time-lock

            * Fulfilled when a certain amount of time has passed without the testator checking in.

        * Second-level DMS time-lock

            * Fulfilled when a certain amount of time has passed without a specified set of other beneficiaries accessing the secret, and the testator checking in.

            * TODO: what would this look like?

            * If testator has not checked in for X months and secret parts have not been submitted by

                * any beneficiary from this group

                * this group of beneficiaries

                * all other beneficiaries 

            * TODO: not sure if these need to be expanded/ more general.
            
            * TODO: Decide if it makes sense to use secret parts as the fulfillment condition as well (specified secret parts have not been submitted?)

        * **NOTE**: the default is to add a time lock condition of 6 months without a check in (from the same beneficiary), for each beneficiary.

#### 5. [WIP] Define access control rules

Access control rules are specified in the following format:

_Simple:_

> When `[beneficiary group]` submits secret parts for combination and they are successfully combined, then `[beneficiary group]` will have access to retrieve the secret.

_Advanced:_

> When `[set of secret parts]` are submitted for combination and successfully combined, then `[beneficiary group]` will have access to retrieve the secret.

**NOTE**: Each advanced rule must specify only *k* unique secret parts. There can only be one rule with a given beneficiary group or set of secret parts.

Both beneficiaries and secret parts may be labeled with key-value pairs, to make them easier to separate into groups. One (possibly) useful example of a custom label is "num_parts", which would be set to the number of parts a beneficiary controls. TODO: decide whether you want to add this label automatically

Beneficiary groups and secret part sets can be specified explicitly, or through the use of access control helpers.

Access control helpers can be used to select certain predefined groups of beneficiaries, or sets of secret parts.

[WIP] Helpers are implemented with selector functions, which include or exclude beneficiaries or secrets. These functions may possibly be defined by the testator.

TODO: look into selector funcs and logic evaluation for custom helpers. Not really a selector func, more of a set generator. Don’t know if eval’ing code is a bad idea, look further into that. There shouldn’t be malicious attacks if the input is limited to the set of all beneficiary objects or the set of all secret objects, because the testator doesn’t have anything to gain from a bad function.

The input to these functions will be either a list of all `secret_part`s:

```
secret_part {
  map<string,string> labels
}
```

or a list of all `beneficiary` objects:
```
beneficiary {
  bool contributed
  map<string,string> labels
}
```

where the contributed flag is true if and only if a beneficiary has submitted a secret part for combination.

TODO: should this necessarily mean they were the first one to submit a part? Or just that they submitted it? 

Access control helpers for secret parts:

1. Any set of secret parts 
    * `(parts -> return all_size_k_subsets(parts))`

2. Any secret parts with label L 
    * `(parts -> return all_size_k_subsets(parts.filter(|p| p.label == L)))`

3. The set consisting of all secret parts 
    * `(parts -> return parts)`

Access control helpers for beneficiaries:

1. Any group of beneficiaries 
    * `(beneficiaries -> return all_subsets(beneficiaries))`

2. The group consisting of the contributing beneficiaries 
    * `(beneficiaries -> return set(beneficiaries.filter(|b| b.contributed == true)))`

3. Any beneficiaries where label L equals V 
    * `(beneficiaries -> return all_subsets(beneficiaries.filter(|b| b.labels[L] == V)))`

4. The group consisting of all beneficiaries 
    * `(beneficiaries -> return set(beneficiaries))`

    1. **NOTE**: this means that all beneficiaries need to participate for the secret to be released, which doesn’t necessarily imply that each beneficiary holds unique secret parts.

5. Any group of size *k* beneficiaries 
    * `(beneficiaries -> return all_size_k_subsets(beneficiaries)`

6. Any beneficiaries who control *k* parts [this must be implemented manually with custom labels] 
    * `(beneficiaries -> return all_subsets(beneficiaries.filter(|b| b.labels["num_parts"] == “k”)))`

TODO: think about this model and whether it can be simplified. These conditions may reduce to something more basic. Maybe selector functions that are modifiable, and operate on the entire set of secrets or beneficiaries.

TODO: The subset generation has issues, because there are subsets of the beneficiaries that don’t have the parts necessary to retrieve the secret. There could be a smarter way to go about this.

TODO: think about required parts as well as optional ones, and whether that’s worth it.

**NOTE**: The default access control rule is "When [any group of beneficiaries] submits secret parts for combination and they are successfully combined, then [any group of beneficiaries] will have access to retrieve the secret."

#### 6. Specify privacy options

 Most configurations should be private by default:
    
    * Can make the parts private (beneficiaries can only combine, not view parts)
    * Can make last check in time private
    * Can make other beneficiaries private to any beneficiary
    * Can make visibility of which secret part numbers have been submitted private
    * Can make access control rules private

TODO: maybe do this in the appropriate previous steps rather than in a separate one?

#### 7. Review and submit.

TODO: not sure what to put here.

The next step is on the beneficiary side, once the secret parts are unlocked, and when the secrets are combined.

#### 8. [WIP] Secret part combination

beneficiaries may have permissions to retrieve and view secret parts. 

Alternately, they can see whether parts have been posted to unlock the secret, and post parts themselves

if other beneficiaries are private, they can only see the total and which parts have been posted.

should they be able to see parts posted from other beneficiaries?

should they be able to take back a contribution?

should they be able to post all the parts they own for a secret at once? Or be able to submit them one by one?

should there be an incentive system for beneficiaries to submit parts if they don’t get access to the secret? Should this come from the testator, or the beneficiaries that have access?

parts can only be posted after the condition is met.

once enough parts are posted to meet the threshold, the secret is unlocked and the access control policy kicks in.

### [WIP] Examples

Here we’ll take a couple of examples from the "Use cases" section and go through the steps to set up a secret sharing configuration for them.

* I have two family members, and a close family friend. I would like to leave assets to the two family members, but if they are unable to claim them in the case of an emergency, I would like the close family friend to have control over the assets.

split secret into one part. threshold is 1

for family member 1, define time lock condition with single part

for family member 2, define time lock condition with single part

for close family friend, define second-level time lock condition with single part, if no other beneficiaries have submitted parts.

* I’m a board member of an exchange called TridegaBS, and I’m worried about the CTO running off with the only set of private keys and faking his death. I want to split up the keys among the other shareholders, so that a quorum of shareholders can choose to retrieve the keys and have it revealed to all of them at once, if a month has passed since the CTO last checked in.

* TODO: is there a way to do this without having the secret exposed, if the secret was a private key to a newly created account? 

let’s say there are are n shareholders.

split secret into n parts. threshold is (n/2) + 1

for each shareholder, defined time lock condition with single unique part

define access control rule: when [any group of beneficiaries holding 1 part] are combined, give control to [all beneficiaries]

* [possibly unrealistic example, also this is just like SafeHaven] I have a child who’s twelve years old and I want him to have my cryptoassets when I pass away. I have a lawyer, but I don’t want to give him the secret to hold. I’d like to give the lawyer an easy way to release access to the secret without holding the secret itself, while preventing my child from accessing the secret until two years after I stop checking in.

* **NOTE**: In reality, it may be possible to have the lawyer sign a contract stating that they’ll keep the secret part confidential, and release it to your child at some point.

split the secret into two parts

give one to the child, the other to the lawyer

no condition on part given to the lawyer

time lock on the part given to the child for two years

when [lawyer and child] combine secret, then [child] receives access.

* I have three children, and I want two of the three children to confirm that the secret should be released, before it is released to all three of them at once.

* I have a recovery account that I know the address of, and I keep the keys to that account etched in steel and buried in a canister under a tree in my backyard. I want to set up a mechanism to give access to the passphrase of my primary account to this recovery account 6 months after I stop checking in from the primary account. However, I want my three friends to confirm that access should be released to the recovery account, in case someone digs up the keys to that account. Of course, these friends wouldn’t have access to the account, or know that what they’re confirming is the release of access to a secret to my recovery account.

* [possibly too complicated] I want to keep my Trezor passphrase with my friends in case I forget it, and I don’t use a recovery account. I have two friends I trust very well, and three that I’m less close to. I’d like a scheme in which one of the friends from the first group needs to combine a part from two of the friends in the second group, and the secret is released to only the friend in the first group. 

### Scratchpad [TODO: remove]:

four parts, threshold is 3

four beneficiaries

b1 -> parts 1,2,3 (family) (time lock)

b2 -> part 1, 2 (close friend) (longer time lock)

b3 -> part 3 (friend) (no time lock) (incentive?)

b4 -> part 4 (friend) (no time lock)

access control rules

if [b2, b3, b4] then [b2,b3,b4]

if b2 submits parts with either b3 or b4, then b2, b3, b4 get access

i.e. if valid set contains parts from b2, then b2 through b4 get access

if b1 submits parts, only b1 gets access

the purpose of this is to let family have immediate access, but multiple friends can get together to retrieve the secret.

### Open questions:

Should there be a way for beneficiaries to prove to each other that they have the secret parts?