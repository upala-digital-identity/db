# DB

Stores individual scores and proofs.
Pool managers publish with their private key. Anyone can read. 
Every network has it's own instance of db (e.g. https://rinkeby.db.upala.io).

## Writing data (POST)

#### Request 

Example message to DB (proof field differs for different pool types):

    {
        poolAddress: "0x11111ed78501edb696adca9e41e78d8256b6",
        poolManagerAddress: "0x33321ed78501edb696adca9e41e78d8256b6",
        scoreBundleId: '0x11111e501...fa0434d7cf87d92345',
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

        singature: "0x3asdgd78501edb696adca9e41e78dhdr5", 
    }

#### Workflow 

- Checks if the sender is an acive pool manager (Graph request "Get pool manager for the pool address").
Error: Signer is not an active pool manager. 
- Checks if the root is active onchain (Graph request "Get bundleId timestamp by bundleId"). Write bundleId to db. 
Error: No such bundleId on-chain
- Checks if [signature] is correct. The whole message must be signed by poolManagerAddress.
Error: Wrong signature.
- Writes data (only increases scores)
Warning: "You tried to decsrease scores. There's no sense in doing it this way. You'll have to publish new score bundle and delete this one (please see the docs)."
- Save the whole transaction into log. 
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
        userID: [bytes20]   // user Upala ID and list of delegates
        pools: [bytes20]    // array of pool addresses trusted by the DApp
                            // todo does multipasport send all? possible pools?
    }

#### Workflow 
- Do house keeping (see below) if time since last one is above 5? minutes 
- Return data

#### Response
Return user scores in **all** trusted pools (one best for each pool). 

E.g.:

    {
        scores:
        [
            {
                userID: "0x23265231ed78501edb696adca9e41e78d8256b6",
                poolAddress: "0x11111ed78501edb696adca9e41e78d8256b6",
                timestamp: '0xa35d', // todo wtf??
                transactionId: '0x42..abcd'   // db transaction id
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
- No scores found for the user

## Housekeeping (GET)
Run housekeeping through an authorized GET request.

- Checks the graph protocol for deleted bundleIds (Graph request "Get array of deleted bundleIds since [latestDeletedBundleTimestamp]")
- Archives stale bundleIds
- Update latestDeletedBundleTimestamp


# DB schema:

    Pools {
        poolAddress: bytes20,
        ScoreBundles: {
            id: bytes32,  // unique bundle id. tree root if Mekrle.
            isDeleted: bool,
            timestamp: unit256,
            bundleSpecificBaseScore: unit256,  //future, overrides baseScore
            Claims: [
                {
                    userUpalaId: bytes20,
                    score: uint8,
                    proof: {  //can be saved as text. Pool specific. 
                        index: uint256,  // for Merkle pool
                        merkleProof: [bytes32],  // for Merkle pool
                        signature: bytes  // for SignedScoresPool
                    }
                }
            ]
        }

    Transactions {
        id: some hash
        data: string  // whole transaction json 
        signature: 
    }

    Housekeeping {
        lastHouseKeepingTimestamp: unit256,
        latestDeletedBundleTimestamp: unit256,
    }


# Admin dashboard
Inspect records, modify for tests purposes. Look for ready-made solutions. 

# Group manager dashboard
Web UI to view scores. Search, filter functionality. 


# Under deprication

## Housekeeping (GET)
Run housekeeping through an authorized GET request.

- Check for updated baseScores (Graph request "Get array of base score updates sinse "latestBaseScoreUpdateTimestamp]")
- Update baseScores 
- Update latestBaseScoreUpdateTimestamp

Schema:

    Housekeeping {
        latestBaseScoreUpdateTimestamp: uint256,
    }