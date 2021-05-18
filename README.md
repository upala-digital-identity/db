Stores merkle trees. No permissions needed. Anyone can publish according to schema. Anyone can read. 

# DB
Records use same schema as in group-tools (todo link)
+ timestamp


# Express-front

## Writing data
Anyone can write data through a POST request. Checking the tree and the root on-chain provides sufficient secutity (except spam attack). 

Writing workflow:
- Checks if the root is active onchain (all roots are ready-stored by workers or requested from graph node)
- Checks if schema is correct
- Randomly checks one merkle proof (Checks if the provided root can be obtained)
- Converts scores to decimal
- Writes data

## Reading data
Anyone can read data through a GET request.
Request params:
userID - Upala user ID
groups - array of groups trusted by DApp that sends the request

Returns user scores in trusted groups (one best for each group):
{
    scores:
    [
        {
            groupID: "0x11111ed78501edb696adca9e41e78d8256b6",
            baseScore: '20'  // decimal
            merkleRoot: '0x11111e501...fa0434d7cf87d92345',
            timestamp: '0xa35d',
            claim: 
            {
                index: 1,
                score: '2', // decimal
                proof: [
                '0xbfeb956a3b70505...55c0a5fcab57124cb36f7b',
                '0xd31de46890d4a77...73ec69b51efe4c9a5a72fa',
                ],
            }
        },
    ...
    ]
}

# Worker
Keeps track of active roots and base scores (probably reads from graph node)

# Admin dashboard
Inspect records, modify for tests purposes. Look for ready-made solutions. 