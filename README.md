# DB. Express-front

Stores merkle trees. No permissions needed. Anyone can publish according to schema. Anyone can read.

## Writing data (POST)
Anyone can write data through a POST request. Checking the tree and the root on-chain provides sufficient secutity (except ddos attack). 

Example message to DB:

    {
        groupAddress: "0x11111ed78501edb696adca9e41e78d8256b6",
        groupManagerAddress: "0x33321ed78501edb696adca9e41e78d8256b6",
        singature: "0x3asdgd78501edb696adca9e41e78dhdr5", 
        merkleRoot: '0x11111e501...fa0434d7cf87d92345',
        claims: {
            [wallet0.address]: {
              index: 0, // unit256?? todo
              score: '0xc8',
              proof: ['0x2a411ed78501edb....fa0434d7cf87d916c6'],
            },
            [wallet1.address]: {
              index: 1, // unit256?? todo
              score: '0x012c',
              proof: [
                '0xbfeb956a3b70505...55c0a5fcab57124cb36f7b',
                '0xd31de46890d4a77...73ec69b51efe4c9a5a72fa',
              ],
            },
        },
    }


#### Workflow:
- Checks if signature is correct. "Welcome to the Resistance!" must be signed by groupManagerAddress.
Error: Wrong signature.

- Checks if the sender is an acive group manager (Graph request "Get group manager for the group address").
Error: Signer is not an active group manager. 

- Checks if the root is active onchain (Graph request "Get root timestamp by roothash")
Error: No such root on-chain

- Checks if schema is correct
Error: Wrong data format.

- Converts scores to decimal? (unit256)??? todo

- Writes data

- Run housekeeping (will populate baseScore and check if root was deleted)

- Return rootTimeStamp

## Housekeeping (GET)
Run housekeeping through an authorized GET request.

- Checks the graph protocol for deleted roots (Graph request "Get array of deleted roots since [latestDeletedRootTimestamp]")
- Archives stale roots
- Update latestDeletedRootTimestamp
- Check for updated baseScores (Graph request "Get array of base score updates sinse "latestBaseScoreUpdateTimestamp]")
- Update baseScores 
- Update latestBaseScoreUpdateTimestamp


## Reading data (GET)
Anyone can read data through a GET request.
Request params:
userID - Upala user ID
groups - array of groups addresses trusted by the DApp that sends the request

#### Workflow:

- Run housekeeping if more than 1 minutes passed since the last one
- Request db and return user scores in trusted groups (one best for each group)

#### Response:

    {
        scores:
        [
            {
                groupAddress: "0x11111ed78501edb696adca9e41e78d8256b6",
                baseScore: '20'  // decimal??
                merkleRoot: '0x11111e501...fa0434d7cf87d92345',
                timestamp: '0xa35d',
                claim: 
                {
                    index: 1,
                    score: '2', // decimal??
                    proof: [
                    '0xbfeb956a3b70505...55c0a5fcab57124cb36f7b',
                    '0xd31de46890d4a77...73ec69b51efe4c9a5a72fa',
                    ],
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
        Roots: {
            hash: bytes32,
            deleted: bool,
            timestamp: unit256,
            rootSpecificBaseScore: unit256,
            claims: [
                {
                    index: uint256,
                    userUpalaId: bytes20,
                    score: uint256, // decimal??
                    proof: [bytes32]
                },
            ]
        }

    Housekeeping {
        lastHouseKeepingTimestamp: unit256,
        latestDeletedRootTimestamp: unit256,
        latestBaseScoreUpdateTimestamp: uint256,
    }


# Admin dashboard
Inspect records, modify for tests purposes. Look for ready-made solutions. 