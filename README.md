# DB

Stores individual scores and proofs.
Pool managers publish with their private key. Anyone can read.

## Writing data (POST)

#### Request

Example message to DB (proof field differs for different pool types):

    {
        chainID: 100,
        poolAddress: "0x11111ed78501edb696adca9e41e78d8256b6",
        poolManagerAddress: "0x33321ed78501edb696adca9e41e78d8256b6",
        bundleId: '0x13411e501...fa0434d7cf87d92345',
        subBundleID: '0x284a1e501...fa0434d7cf87d91111', // is hash of claims
        claims: {
            [wallet0.address]: {
              score: 67,
              proof: {  // for SignedScoresPool:
                        signature: '0x2a411...'
                    }
            },
            [wallet1.address]: {
              score: 87,
              proof: {  // for Merkle pool:
                        index: 3,  
                        merkleProof: ['0x2a411...','0x2acf16c6...'] 
                    }
            },
        },
        timestamp: 1654201893,
        previousSignature: "0x4bsdgd78501edb696adca9e41e78dadf3",
        singature: "0x3asdgd78501edb696adca9e41e78dhdr5", 
    }

#### Workflow

- Checks if the sender is an acive pool manager (Graph request "Get pool manager for the pool address").
Error: Signer is not an active pool manager.
- Checks if the root is active onchain (Graph request "Get bundleId timestamp by bundleId"). Write bundleId to db. 
Error: No such bundleId on-chain
- Checks if [signature] is correct. The whole message must be signed by poolManagerAddress.
Error: Wrong signature.
- Checks previous signature
Error: Cannot find previous signed db transaction
- Checks timestamp.
Error: Wrong timestamp
- Checks scores (must only increase)
Error: "You tried to decsrease scores. There's no sense in doing it this way. You'll have to publish new score bundle and delete this one (please see the docs)."
- Calculates total scores for each user in the bundle  // todo or better do it on app side? 
- Writes data
- Save the whole transaction into log // todo deprecate if we store all subBundles with timestamps?
- Return result


#### Response

Example return:

    {
        updatesCount: number of updated scores;
        failedToDecreaseCount: number of scores not updated due
        warning: "You tried to decsrease scores..."
        error: ""
    }


## Reading data (GET)

Anyone can read data through a GET request. 

#### Request

    {
        chainID: [uint256]  // chain id if dapp (and array if multipassport)
        userID: [bytes20]   // user Upala ID and/or list of delegates
        pools: [bytes20]    // array of pool addresses trusted by the DApp
                            // todo does multipasport send all? possible pools?
        signature: [bytes20]// multipassport can querry multiple IDs 
                            // on multipple chains
    }

#### Workflow

- check request validity
- check signature if present
- Return data (calculates)

#### Response

Return user scores in **all** trusted pools (one best for each pool).  **TODO** Or better return totalScores?

E.g.:

    {
        scores:
        [
            {
                chainID: 100,
                userID: "0x23265231ed78501edb696adca9e41e78d8256b6",
                poolAddress: "0x11111ed78501edb696adca9e41e78d8256b6",
                timestamp: '0xa35d', // todo wtf??
                transactionId: '0x42..abcd'   // db transaction id // todo wtf?? // shoud be subBundle ID?
                claim: 
                {
                    score: '2', // uint8
                    proof: { signature: "0x447f8...fa0434d7cf87d92345" }
                }
            },
        ...
        ]
    }


#### Errors:

- No such user
- No scores found for the user on the specified chain

## Housekeeping

Run housekeeping through an authorized GET request (or every n minutes?).

- Checks the graph protocol for deleted bundleIds (Graph request "Get array of deleted bundleIds since [lastHouseKeepingBlock]").
- Checks the graph protocol for pool ownership transfers (Graph request "Get array of new poolManagerAddresses since [lastHouseKeepingBlock]")
- Deactivate(delete) stale bundleIds (deleted and with new owners)
- Check for updated baseScores (Graph request "Get array of base score updates sinse "latestBaseScoreUpdateTimestamp]")
- Recalculate baseScores? // todo
- Update [lastHouseKeepingBlock]


## DB schema:

    Pools {
        chainID: uint256,
        poolAddress: bytes20,
        baseScore: uint256,
        scoreBundles: {
            id: bytes32,  // unique bundle id. tree root if Mekrle.
            isDeleted: bool,
            timestamp: unit256,  // is assigned by smart contract
            bundleSpecificBaseScore: unit256,  //future, overrides baseScore
            subBundles: {  // todo should includ here?
                id: id: bytes32,
                claims: [
                    {
                        userUpalaId: bytes20,
                        score: uint8,   // the score used in proof
                        proof: {        //can be saved as text. Pool specific. 
                            index: uint256,             // for Merkle pool
                            merkleProof: [bytes32],     // for Merkle pool
                            signature: bytes            // for SignedScoresPool
                        },
                        totalScore: uint256 // [Total] = [baseScore] * [score] 
                                            // calculated when POSTing, housekeeping
                    }
                ]
            }
        }

    Transactions {  // deprecate? use only subBundles?
        id: some hash
        data: string  // whole transaction json 
        signature: 
    }

    Housekeeping {
        chainID: {
            lastHouseKeepingBlock: unit256,
        }
        
    }


#### Admin dashboard

Inspect records, modify for tests purposes. Look for ready-made solutions. 

#### Group manager dashboard

Web UI to view scores. Search, filter functionality. 

#### Questions

- increasing-decreasing scores. Just return error if manager tries to decrease?
- where total scores better be calculated? In the DB or Dapp side?
- Include subBundles into schema? Deprecate transaction tracking? subBundles are transactions!