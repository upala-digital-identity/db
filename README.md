# DB

Stores individual scores and proofs.
Group managers can publish with their private key. Anyone can read. 
Every network has it's own instance of db (e.g. https://rinkeby.db.upala.io).

## Writing data (POST)

- Checks if the sender is an acive group manager (Graph request "Get group manager for the group address").
Error: Signer is not an active group manager. 
- Checks if the root is active onchain (Graph request "Get root timestamp by roothash"). Write root to db. 
Error: No such root on-chain
- Checks if [signature] is correct. The whole message must be signed by groupManagerAddress.
Error: Wrong signature.
- Converts scores to decimal? (unit256)??? todo
- Writes data (only increases scores)
Warning: "You tried to decsrease scores use --force-decrease to update those"
- Save transaction the whole transaction into log. 
- Return result

Example message to DB (proof field differs for different pool types):

    {
        groupAddress: "0x11111ed78501edb696adca9e41e78d8256b6",
        groupManagerAddress: "0x33321ed78501edb696adca9e41e78d8256b6",
        singature: "0x3asdgd78501edb696adca9e41e78dhdr5", 
        scoreBundleId: '0x11111e501...fa0434d7cf87d92345',
        claims: {
            [wallet0.address]: {
              score: 67,
              proof: {  
                        signature: '0x2a411...'  // if SignedScoresPool
                    }
            },
            [wallet1.address]: {
              score: 87,
              proof: {
                        index: 3,  // if Merkle pool
                        merkleProof: ['0x2a411...','0x2acf16c6...'] 
                    }
            },
        },
    }

Example return:

    {
        updatesCount: number of updates scores;
        failedToDecrease: number of scores not updated due
        warning: "You tried to decsrease scores use --force-decrease to update those"
        error: ""
    }


## Reading data (GET)
Anyone can read data through a GET request.
Request params:
userID - Upala user ID
groups - array of groups addresses trusted by the DApp that sends the request
For multipassport:
number of best scores - how many best scores should be retrieved
start from - multipasport may request scores from posittion 10 to 20 when user presses "more" button.

Returns:
Returns all individual scores.

#### Workflow:

- Request db and return user scores in trusted groups (one best for each group)

#### Response:

    {
        scores:
        [
            {
                groupAddress: "0x11111ed78501edb696adca9e41e78d8256b6",
                pool: '0x11111e501...fa0434d7cf87d92345',
                timestamp: '0xa35d', // todo wtf??
                transactionId: '0x42..abcd'   // db transaction id
                claim: 
                {
                    score: '2', // decimal??
                    proof: { signature: "0x447f8...fa0434d7cf87d92345" }
                }
            },
        ...
        ]
    }


#### Errors:
- No such user
- No scores found for the user

# DB schema:

    Groups {
        groupAddress: bytes20,
        baseScore: uint256,
        ScoreBundles: {
            hash: bytes32,  // unique bundle id. tree root if Mekrle.
            isDeleted: bool,
            timestamp: unit256,
            bundleSpecificBaseScore: unit256,  //future, overrides baseScore
            Claims: [
                {
                    userUpalaId: bytes20,
                    score: uint256, // or decimal??
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




# Admin dashboard
Inspect records, modify for tests purposes. Look for ready-made solutions. 

# Group managers dashboard
Web UI to view scores. Search, filter functionality. 


# Under deprication

## Housekeeping (GET)
**UPDT. Nothing is needed in this section anymore**
Run housekeeping through an authorized GET request.

- Checks the graph protocol for deleted roots (Graph request "Get array of deleted roots since [latestDeletedRootTimestamp]")
- Archives stale roots
- Update latestDeletedRootTimestamp
- Check for updated baseScores (Graph request "Get array of base score updates sinse "latestBaseScoreUpdateTimestamp]")
- Update baseScores 
- Update latestBaseScoreUpdateTimestamp

Schema:

    Housekeeping {
        lastHouseKeepingTimestamp: unit256,
        latestDeletedRootTimestamp: unit256,
        latestBaseScoreUpdateTimestamp: uint256,
    }