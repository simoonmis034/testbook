# Commitment Status



Commitment Status

The commitment metric gives clients a standard measure of the network confirmation for the block. Clients can then use this information to derive their own measures of commitment.There are three specific commitment statuses:

* Processed
* Confirme
* Finalized

​

| Property                              | Processed | Confirmed | Finalized |
| ------------------------------------- | --------- | --------- | --------- |
| Received block                        | X         | X         | X         |
| Block on majority fork                | X         | X         | X         |
| Block contains target tx              | X         | X         | X         |
| 66%+ stake voted on block             | -         | X         | X         |
| 31+ confirmed blocks built atop block | -         | -         | X         |
